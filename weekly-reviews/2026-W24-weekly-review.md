# 每日推理一题周复盘（2026-06-08 ~ 2026-06-14）

---

## 1. 本周学习轨迹

### 06/08（周一）— Prefix Cache 与 Preemption 的博弈（P800 版）

> **严谨性说明**：原题中的 8×P800 32GB 场景数值存在不自洽（显存压力不充分、preemption 触发条件不明确），经用户指出后以 admission control 框架重新修正了题目逻辑。

**📋 题干**

**硬件**：8×昆仑芯 P800 (32GB)，vLLM，Llama3-70B (INT4)，TP=8，max_num_seqs=96，block_size=16

**KV参数**：每卡每token 40 KiB → 每block 640 KiB → 共 36,500 blocks/卡

**流量**：
| 类型 | 占比 | 每请求KV blocks | 备注 |
|------|------|----------------|------|
| A（RAG问答） | ~80% | 150 blocks（共享prefix 128 + 新增 25） | 共享2K RAG前缀 |
| B（长文档分析） | ~20% | ~2,019 blocks/请求 | 32K长文档，几乎无共享 |

**Q1：显存预算验证** — (a) 96请求满running时是否超出36,500？(b) prefix cache完全失效时？(c) B类单独能否放下？

**Q2：Preemption + Prefix Cache连锁反应** — (a) preemption触发机制及释放效率分析；(b) A请求prefix命中率为什么从85%降到30%；(c) 0.35% blocks被evict为何导致全局性能崩溃？

**Q3：方案设计** — (1) B请求准入控制；(2) 缓存管理与eviction策略；(3) 业务层/调度层优化

**核心考点**：Prefix Cache 开启后共享 prefix blocks 对 preemption 行为的影响；admission control（准入控制）作为 preemption 的第一道防线；B 请求 burst 导致 cache thrashing 的连锁反应。

**标准答案**：

**Q1：显存预算验证**

- (a) 正常运行时（prefix 命中）：
  - B 占用 blocks：19 × 2,019 = 38,361
  - A 新增 blocks：77 × 25 = 1,925
  - A 共享 prefix：128 blocks
  - 总占用：38,361 + 1,925 + 128 = **40,414 blocks**
  - 是否超过 36,500？**是，超出 3,914 blocks**

- (b) Prefix cache 完全失效：
  - A 占用 blocks：77 × 150 = 11,550
  - B 占用 blocks：19 × 2,019 = 38,361
  - 总占用：**49,911 blocks**
  - 相比 (a) 多占 49,911 - 40,414 = **9,497 blocks**——这些 block 从 free pool 中拿，进一步加剧显存压力

- (c) 全部 B 请求（最坏情况）：
  - 96 × 2,019 = **193,824 blocks**
  - 总容量 36,500，**显然放不下**
  - 这说明：**B 请求的长上下文（32K）是系统的压力主源**

**Q2：Preemption + Prefix Cache 连锁反应**

- (a) 所有 A 被 preempt 释放 blocks：77 × 25 = **1,925 blocks**
  - 缺口 3,914 - 1,925 = **1,989 blocks 仍然不够**
  - Scheduler 下一步：必须继续 preempt B 请求（每个释放 ~2,019 blocks，足够弥补缺口）
  - B 被 preempt 后恢复代价：32K tokens 需要重新 **prefill 32K tokens ≈ 213ms**

- (b) 正反馈循环链条：
  ```
  B 请求 admit → (1) free pool 耗尽 → (2) trigger preemption → (3) prefix evicted
    → (4) A 请求 cache miss → (5) 需要重新 prefill 2K tokens
    → (6) prefill 占用 GPU 时间 → (7) decode 变慢 → (8) 队列积压
  ```
  正反馈条件：**evict 频率 > prefix 恢复速度**，即 B 持续到达反复触发 preemption

- (c) 命中率 85%→30% 的原因：
  A 的 prefix 只需要 128 blocks（占 36,500 的 0.35%），比例极小。但 preemption 发生时 scheduler 不关心"这是 cache block"，只关心"哪个 block 可以释放"。128 blocks 虽然占比小，但它们是**所有 A 请求的共享依赖**——evict 了这 128 blocks，所有 A 请求全部 miss。

**Q3：方案设计**

- **方案 1 - B 请求准入控制**：限制 B 请求的 max_concurrent，或对 B 单独设更小的 max_num_seqs，或 B 不做 prefix cache insert（避免 cache pollution）。
  - 权衡：B 的 TTFT 可能上升（需要排队），但 A 的 TPOT 稳定性大幅改善

- **方案 2 - 缓存管理优化**：对 prefix cache 做 soft reservation（预留 ~5% blocks ≈ 1,800 blocks），或设 sticky flag 降低 eviction 优先级。
  - 权衡：预留的资源在空闲时可被 B 借用，但要确保 B 归还

- **方案 3 - 业务层调度**：A/B 分池处理、超长 B 做 chunked prefill 降低瞬时 block 压力、RAG prefix 做固化

---

### 06/09（周二）— 数值验证：Preemption 根因定位

**📋 题干**

基于 06/08 同一组硬件参数（8×P800 32GB, Llama3-70B INT4, TP=8），深入验证显存预算：

- **重新验算 KV budget**：P800 32GB × gpu_memory_util=0.9 = 28.8 GiB/卡。扣除 weights（~5 GiB/卡）和 non-KV（~1.5 GiB）= **22.3 GiB/卡**留给 KV Cache → 36,500 blocks/卡
- **流量分析**：96请求（~77 A + ~19 B）在prefix命中时需要 ~40,414 blocks，**超出** 36,500 → **scheduler根本不会允许全部admit**
- **根因定位**：preemption 不是因为 prefix cache 占太多，而是 `max_num_seqs=96` 设置过高，导致 admit 的请求超过 KV pool 承载能力

**核心考点**：通过精确的显存预算估算发现 preemption 的根因是 optimistic over-admission 而非 prefix cache。

**标准答案**：

- P800 32GB，gpu_memory_util=0.9 → 28.8 GiB/卡可用
- 扣除 weights（Llama3-70B INT4 ≈ 40 GB total, TP=8 → 5 GB/卡）和 non-KV（~1.5 GB）= **22.3 GB/卡留给 KV Cache**
- 每 block = 640 KiB → **36,500 blocks/卡**
- A 请求（2K prefix + ~200 input + ~200 output）= 150 blocks/请求（其中 128 共享 prefix）
- B 请求（32K 文档 + ~100 指令 + ~200 output）= ~2,019 blocks/请求
- **结论**：preemption 不是因为 prefix cache 占太多，而是 max_num_seqs=96 设置过高，导致 admit 的请求超过 KV pool 承载能力。Preemption 是"超额准入的兜底机制"。

---

### 06/10（周三）— Chunked Prefill 的成本与收益权衡

**📋 题干（完整原文）**

**背景**：7B FP16 模型部署在单张 A100 80GB 上。流量包含两类请求：短请求 S（512 tokens input + 128 output）和长请求 L（4K tokens input + 128 output），各占 50%。

| 参数 | 值 | 说明 |
|------|------|------|
| 模型 | 7B FP16 | 单张 A100 80GB |
| Prefill 吞吐 | ~6,000 tok/s | compute-bound |
| Decode TPOT（batch=16） | ~30ms | memory-bound |
| 流量 | 50% 短请求（512+128）+ 50% 长请求（4K+128） | — |
| 并发 | 假设 16 请求全满 | — |

**Q1：长请求 prefill 期间发生了什么？**

(a) 4K 长请求的 prefill 需要多少连续 GPU 时间？
(b) 在此期间同 batch 中 decode 的短请求 TPOT 会变成多少？
(c) 如果有 8 个短请求在 decode，这批请求的 P99 TPOT 会受多大影响？

**Q2：Chunked Prefill 的平滑效果**

开启 chunked prefill（chunk=256 tokens），假设 chunked prefill throughput 是全量 prefill 的 60%。

(a) 每个 chunk 的 prefill 时间？
(b) decode 请求的 TPOT 变成多少？
(c) 长请求的总 prefill 时间变成多少？
(d) 用一句话总结 chunked prefill 的收益和代价。

**Q3：最优 Chunk Size**

若 chunk size 可选 128/256/512/1024，对应 throughput 分别为 40%/60%/75%/90% 的全量效率。

(a) 在短请求 TPOT 优先的场景下，应选哪个 chunk size？
(b) 在长请求 TTFT 优先的场景下呢？
(c) GQA（8 KV heads vs 32 MHA heads）对 chunk size 选择范围有什么影响？

**核心考点**：Chunked Prefill 的 TTFT↔TPOT 权衡；blocking 机制的本质变化；chunk size 光谱选择；GQA 对 KV Cache 的影响。

**标准答案**：

- **无 Chunked Prefill**：L 的 TTFT=667ms（prefill 4K tokens），S 的 TPOT=667+25=**692ms**（被 L 的 prefill 阻塞，等 L prefill 完才能 decode）
- **有 Chunked Prefill（chunk=256）**：L 的 TTFT=4,096/256 × 78ms = 1,248ms（16 个 chunk），S 的 TPOT=**78ms**（每 step 都做 decode，不再被长 prefill 阻塞）
- **核心 trade-off**：TTFT ↑（1,248 vs 667ms），TPOT ↓（78 vs 692ms）
- **Chunk size 选择**：大 chunk → TTFT 好、TPOT 差；小 chunk → TTFT 差、TPOT 好。不存在全域最优值
- **GQA 的影响**：8 KV heads（vs MHA 32 heads）→ KV Cache 减 4× → 更多 KV block → 更大的 chunk size 选择空间

---

### 06/11（周四）— vLLM Preemption 的成本与调度策略

> **严谨性说明**：原题设定被 preempt 请求占用 64 blocks 与 4K prompt 物理不自洽（64×16=1,024 < 4,096），已修正为 288 blocks（256 for prompt + 32 for decode）。

**📋 题干（完整原文）**

> **难度**：⭐⭐⭐⭐ | **标签**：#vLLM #Scheduler #Preemption #KVCache #Serving

**背景**：你负责维护一个线上推理服务，配置如下：
- 模型：Qwen2-72B (FP16, GQA, n_kv_heads=8, d_head=128, layers=80)
- GPU：2×A100 80GB（TP=2，NVLink 互联）
- vLLM 配置：max_num_seqs=128，max_model_len=16384，gpu_memory_utilization=0.9
- 峰值并发：~110 个请求同时在线

最近用户投诉高峰时段体验大幅劣化：
- TTFT 从平均 1.2s → 8~15s
- P99 TPOT 从 45ms → 120ms
- 服务日志高频出现 `Sequence group preemption detected`，伴随大量 `Swapping blocks to CPU`

**问题 1：Preemption 成本估算（计算分析）**

vLLM 有两种 preemption 模式：swap（将 KV Cache 搬到 CPU）和 recompute（释放 block，下次重新 prefill）。假设 block_size=16，一个被 preempt 的请求目前已占用 64 个 block（prompt 已全部 prefill 完成，处于 decode 阶段）。

(a) 这个请求的 KV Cache 在 GPU 上占多少 MiB？（公式：2 × layers × n_kv_heads × d_head × block_size × num_blocks × dtype_bytes）

(b) 如果采用 swap preemption，通过 PCIe 4.0 x16 将这 64 个 block 全部 swap 到 CPU，理论传输时间是多少？（假设 PCIe 4.0 x16 有效单向带宽 ~20 GB/s）

(c) 如果采用 recompute preemption，假设原 prompt 长度为 4K tokens，A100 单卡的 prefill throughput 约为 150K tokens/s（FP16，TP=2），recompute 需要多长时间？

(d) 综合判断：给定上述计算，swap 和 recompute 分别在什么条件下更优？如果 block 数翻倍（128 blocks），结论会变吗？

**问题 2：调度逻辑代码 Review（代码审查）**

下面是一段简化版的 vLLM Scheduler preemption 逻辑，请找出至少 3 处工程问题：

```python
class Scheduler:
    def __init__(self, block_manager, swap_out_threshold=64):
        self.block_manager = block_manager
        self.running: list[SeqGroup] = []
        self.waiting: list[SeqGroup] = []
        self.swap_out_threshold = swap_out_threshold

    def preempt(self, victim: SeqGroup, blocks_needed: int):
        """从 victim 抢占 blocks_needed 个 block 给新请求"""
        if victim.num_blocks < self.swap_out_threshold:
            # === 方案 A: recompute ===
            victim.state = "recompute"
            self.block_manager.free(victim.seq_ids)
            self.waiting.append(victim)  # 放回队尾
        else:
            # === 方案 B: swap ===
            blocks_to_swap = victim.block_ids[-blocks_needed:]  # [1]
            self.block_manager.swap_out(blocks_to_swap, "cpu")
            victim.block_ids = victim.block_ids[:-blocks_needed]
            if len(victim.block_ids) > 0:
                self.running.append(victim)  # [2]
            else:
                victim.state = "swapped"
                self.running.append(victim)  # [3]
```

逐行审查标注了 [1]、[2]、[3] 的位置以及更大的策略问题。你觉得哪里有问题？

**问题 3：方案设计**

(a) 日志显示被 preempt 的请求几乎全是长 prompt 的新请求（输入 8K~16K tokens），短对话很少被 preempt。这合理吗？为什么？vLLM 的调度策略是否应该优先保护短对话？

(b) 你认为 max_num_seqs=128 设置合理吗？如果 gpu_memory_utilization=0.9，请你粗略估算这个配置下每请求平均可用多少 KV Cache block，是否足够支持 max_model_len=16384？

(c) 如果启用了 Prefix Cache，多个请求共享同一 prefix 的 block。preempt 其中一个请求时，这些共享 block 应该怎么处理？释放还是不释放？

**应用场景**：Preemption 是 vLLM/SGLang/TGI 等 serving 框架应对显存超载的"安全阀"。频繁 preemption 会直接击穿 TTFT 和 TPOT 的 SLA。理解 preemption 的成本构成、选择合适的策略、优化调度优先级，是线上推理服务调优绕不开的核心课题。

**优化方向**
| 层级 | 优化思路 |
|------|----------|
| 调度策略 | 优先级队列（长 prompt 降权）、max_num_seqs 动态调整、preemption 保护短请求 |
| 显存管理 | Prefix Cache 减少重复 KV、更好的 block 分配算法、block_size 调优 |
| 通信优化 | GPU→CPU swap 的 PCIe 瓶颈 → SSD/NVMe offload？→ 压缩传输？ |
| 策略组合 | 混合 preemption：小 prompt 用 recompute（快），长 prompt 用 swap（省） |

**诚实边界**
- GPU↔CPU 的 PCIe 带宽强烈依赖拓扑（是否经过 P2P switch、是否同一 NUMA node、是否被其他 GPU 共享带宽）。20 GB/s 是一个乐观估算，生产环境实测可能只有 10~14 GB/s。建议用 cudaMemcpy + nvbandwidth 在你自己的集群上跑一下。
- 不同模型架构（Dense vs MoE）的 preemption 行为会有差异，尤其 MoE 模型每层 expert 参数量大，recompute 成本会更高，swap 的相对优势更明显。

**核心考点**：KV Cache 计算、swap/recompute 成本对比、victim 选择、preemption 代码实现。

**标准答案**：

**Q1：成本估算**

- Q1-1 KV Cache：每 token = 2×80×8×128×2 = 320 KiB。每 block = 320×16 = 5 MiB。288 blocks = **1,440 MiB ≈ 1.41 GiB total**，per GPU = **720 MiB**
- Q1-2 Swap 时间：1,440 MiB / 20 GB/s = **72ms**（[简化] 总 KV / 单 PCIe 链路）
- Q1-3 Recompute 时间：(4,096+512) / 150K = **30.7ms**（vLLM 的 recompute 会把已知 output 拼到 prompt 后一次性 re-prefill）
- Q1-4 对比：两者都随总 token 数线性增长，斜率比恒定 ~2.46×。Recompute 时间更短，但 swap 不争抢 GPU compute 资源。高峰期 prefill capacity 打满时 swap 可能更优

**Q2：代码审查**

| 标记 | 问题 | 正确做法 |
|------|------|----------|
| [1] | `[-blocks_needed:]` 只 swap 末尾 N block，不解决根本问题 | 应释放 victim 的全部或大部分 unique blocks |
| [2] | Swap 后放回 running，KV 不完整会崩溃 | 应移入 swapped 队列 |
| [3] | `state="swapped"` 但仍在 running，状态与队列矛盾 | swapped 状态的请求不应在 running 中 |
| 策略 | 固定阈值 64 blocks | 动态比较 swap_cost vs recompute_cost |
| 缺漏 | 未处理 prefix cache 共享 blocks | 只释放 unique blocks，保留 refcount>1 的共享 prefix |
| 缺漏 | Recompute 放队尾 | 放队首防 starvation |
| 缺漏 | 无 victim 选择逻辑 | 优先选恢复成本低（总 token 数少）的请求 |

**Q3：方案设计**

- 长 prompt 容易被 preempt 合理但策略不合理——应优先 preempt 恢复成本低的请求
- max_num_seqs=128 严重过高（FP16 72B 在 2×A100 上 KV budget 极低）
- Prefix Cache 共享 blocks 在 preemption 时应分层处理：refcount>1 保留，refcount=0 可 evict

---

### 06/12（周五）— Chunked Prefill 的调度博弈

**📋 题干（完整原文）**

> **难度**：⭐⭐⭐ | **标签**：#ChunkedPrefill #Scheduler #vLLM #SGLang #TTFT #TPOT

**应用场景**：你的团队用 vLLM 部署了 Llama3-70B 推理服务。线上流量包含大量长文档分析请求（输入 > 16K tokens）和实时对话请求（短输入、低延迟要求）。近期运维反馈：长请求一多，所有用户的打字机输出都变得一顿一顿的，TPOT 飙到 200ms+，但直接关掉 chunked prefill 又导致 TTFT 爆炸。

**背景假设**

| 参数 | 值 | 说明 |
|------|-----|------|
| 模型 | Llama3-70B, FP16, GQA (n_kv_heads=8, d_head=128), 80 layers | — |
| 硬件 | 8×A100 80GB, TP=4 | per-replica 4 GPU |
| 显存 | gpu_memory_utilization=0.9 | weights ≈ 35 GB/GPU, KV Cache budget ≈ 37 GB/GPU |
| Prefill 吞吐 | ~4,000 tok/s/GPU (FP16, TP=4, 连续 batching 下的均值) | **估算值** |
| Decode 单步延迟 | ~40ms (TP=4, batch=64) | **估算值** |
| 流量画像 | 30% 长文档 (32K input, 500 output) + 70% 短请求 (512 input, 256 output) | 峰值 ~32 并发，其中同时约 8-10 个长请求 |

**问题 1：TTFT 计算分析**

(a) 不开启 chunked prefill 时，一个 32K token 请求的 prefill 耗时多长？这期间同 batch 的其他 decode 请求的 TPOT 会变成多少？

(b) 开启 chunk_size=4096 后，这个 32K 请求被拆成几个 chunk？每个 chunk 处理期间，同 batch 的 decode 请求大约能跑几步？

(c) 对比 chunk_size=512 vs chunk_size=4096 对长请求 TTFT 和短请求 P99 TPOT 的各自影响。用数字说话。

**问题 2：调度策略 Review**

你同事写了下面这段简化调度逻辑：

```python
def schedule(ready_queue, running_seqs, max_num_seqs):
    prefill = [r for r in ready_queue if r.is_prefill]
    decode = [r for r in ready_queue if not r.is_prefill]
    batch = []

    # 先把所有 prefill 都塞进 batch
    while prefill and len(batch) < max_num_seqs:
        batch.append(prefill.pop(0))

    # 剩余容量给 decode
    while decode and len(batch) < max_num_seqs:
        batch.append(decode.pop(0))

    return batch
```

(a) 这段调度逻辑在 chunked prefill 开启时会有什么问题？试想 8 个长请求同时到达的场景。

(b) 如果 batch 被 8 个 prefill chunk 占满，decode 请求完全进不来——这种饥饿状态会持续多久？如何从调度策略层面避免？

(c) vLLM 和 SGLang 在 prefill/decode 混合调度上的思路有什么本质区别？

**问题 3：Benchmark 设计**

Leader 让你给出"当前流量下最优的 chunk_size"。

(a) 设计 benchmark，需要覆盖哪些 workload 维度？

(b) 除了 TTFT / TPOT / OTPS，还需要采集哪些指标才能完整评估 chunked prefill 的效果？

(c) 实测发现：chunk_size=512 相比 4096，长请求 P99 TTFT 从 12s 升到 18s，但短请求 P99 TPOT 从 120ms 降到 55ms。你会建议用哪个值？理由是什么？如果 SLA 是 "TTFT < 15s 且 TPOT < 80ms"，答案会变吗？

**诚实边界**

- Prefill 吞吐 4000 tokens/s 是 **估算**，实际受 batch 组成、FlashAttention 对 O(n²) 的缓解程度、chunk_size、GPU 降频等影响，波动范围 2000-6000 tokens/s
- Decode 步延迟 40ms 是 **估算**，随 batch_size 增大线性增加。上述计算假设 batch 基本稳定
- 调度器伪代码做了大量 **教学简化**，vLLM v0.6.x 和 SGLang v0.3.x 的调度器实现差异较大
- 最优 chunk_size 必须通过实际 A/B 测试验证，本题目的是建立分析框架

**优化方向**

| 层级 | 优化思路 |
|------|----------|
| 框架/调度层 | chunk_size 自适应调节、request-aware 调度（长/短请求分离调度）、优先级队列 |
| 系统层 | PD 分离——从根本上解耦 prefill 和 decode，让两种请求各跑各的 GPU |

**核心考点**：Chunked Prefill 的 TTFT↔TPOT 光谱权衡；调度器中 prefill/decode slot 分配的优先级反转问题；单一 chunk size 在双 SLA 约束下的局限性；SGLang Stream 模型 vs vLLM SeqGroup 模型的调度差异。

**标准答案**：

**Q1：TTFT 计算分析**

- Q1(a) 无 chunked prefill：
  - 32K prefill = 32,000 / 4,000 = **8 秒**
  - 这 8 秒内 GPU 全程 prefill，decode 一步不能跑 → 同 batch decode TPOT = **8,000 ms**
  - 生产灾难性——打一个字等 8 秒

- Q1(b) Chunked prefill, chunk=4096：
  - 拆成 32,000 / 4,096 ≈ **8 个 chunk**
  - 每 chunk 时间：4,096 / (4,000×90%) ≈ **1.14s**
  - 每 step prefill chunk + decode 混合 → 8 步内 decode 推进 **8 个 token**

- Q1(c) 对比：

| Chunk size | 有效 throughput | 长请求 TTFT | 短请求 TPOT |
|-----------|----------------|------------|------------|
| 无 chunked | 4,000 tok/s | **8.0s** | **8,000ms** |
| 4096 | ~3,600 tok/s (90%) | **9.1s** (+14%) | **1,140ms** (28×正常) |
| 512 | ~2,400 tok/s (60%) | **13.4s** (+67%) | **210ms** (5×正常) |

**Q2：调度策略 Review**

- Q2(a) 问题：**优先级反转**——代码先塞满 prefill chunk 再给 decode，8 个长请求（64 个 chunk）可以连续多轮霸占全部 slot，decode **完全进不来**
  - Starvation 持续：8 轮 × 1.14s ≈ **9 秒**

- Q2(b) 正确做法——**先预留 decode slot**：
  ```python
  min_decode_slots = 4  # 保证每轮 decode 不被饿死
  batch += decode[:min_decode_slots]
  batch += prefill[:max_num_seqs - len(batch)]
  ```
  更精确：用 token budget（`max_num_batched_tokens`）控制每轮 prefill token 总量

- Q2(c) vLLM vs SGLang 本质区别：

| 维度 | vLLM | SGLang |
|------|------|--------|
| 调度单位 | **SeqGroup**（请求级） | **Stream**（流式执行上下文） |
| Prefix 感知 | 调度层和 cache 层解耦 | **深度耦合**——RadixCache 直接影响调度决策 |
| 核心机制 | Token budget 分配 | Stream 编排，可在 RadixCache 节点 fork/join |
| 多请求共享 | BlockManager 处理，调度器无感知 | 调度器**显式感知**共享 prefix → 自动聚簇 |
| Preemption 粒度 | 请求级 | 更细（stream fork 可独立管理） |

**Q3：Benchmark 设计**

- Q3(a) Workload 维度：请求长度分布（纯短/混合/纯长）、并发度（低/中/高）、到达模式（稳态/burst）、chunk size（无/512/1024/2048/4096）、prefix 共享度（0%/50%/80%）

- Q3(b) 额外指标：Prefill 有效 throughput、GPU 利用率（compute vs memory）、decode starvation 时长、prefill chunk 排队深度、batch 组成（prefill/decode tokens 占比）、P99/P99.9 尾部延迟、Jitter（TPOT 标准差）

- Q3(c) 双 SLA 下（TTFT<15s 且 TPOT<80ms）：
  - Chunk=512：TTFT=18s > 15s ❌
  - Chunk=4096：TPOT=120ms > 80ms ❌
  - **两者都不满足** → 需要 PD 分离、请求感知调度（长请求大 chunk、短请求小 chunk）、或自适应 chunk size

**用户讨论亮点**：
- 用户准确指出"chunked prefill 在分离式场景没有收益"，并追问 PD 分离场景开启 chunked prefill 的目的——dodo 解释在 PD 分离下 chunked prefill 的角色从"保护 decode TPOT"变为"细粒度 prefill 任务调度 + 减少 pipeline bubble + 平滑 KV 传输"
- 用户追问 SGLang Stream 模型，dodo 用 RadixCache fork/join 类比解释了 vLLM SeqGroup vs SGLang Stream 的本质差异

---

### 06/13~06/14（周六~周日）

> **诚实说明**：用户尚未查看这几天的题目。每日推理一题定时任务已按计划生成题目，但用户在本次周复盘期间尚未回顾 06/13 和 06/14 的内容。本周完整学习内容覆盖 06/08~06/12 的问答记录，其中 06/12 为本周重点讨论。

---

## 2. 本周核心 Takeaways 汇总

### 2.1 Prefix Cache + Preemption + 准入控制

1. **Preemption 是乐观超额准入的兜底机制，不是 prefix cache 的错**——当 max_num_seqs 设置过高，scheduler 承认了超过 KV pool 承载能力的并发，preemption 必然高频触发。根本解法是 admission control，而不是优化 eviction 策略。

2. **Prefix cache 在显存受限场景下是"受害者"而非"肇事者"**——A 请求的共用 prefix 只需要 128 blocks（占 36,500 总 blocks 的 0.35%）。被 evict 不是因为它们占太多显存，而是因为 B 请求 burst 时 preemption 连带释放了 cache 中的 blocks。

3. **单次 prefix eviction 的影响是一次性的**——一个 A 请求 miss → re-prefill 2K tokens → prefix 挂回 RadixTree → 后续 A 请求全部 cache hit。持续劣化需要 evict 频率 > prefix 恢复速度。

4. **B 请求（32K tokens）是系统的压力主源**——每个 B 请求占 ~2,019 blocks，是 A 请求（~150 blocks）的 13 倍。对 B 做准入控制比优化 A 的 eviction 优先级更有效。

5. **Preemption 的连锁反应**：B burst → free pool 耗尽 → preempt B → B 释放 blocks → 新 B 又进来 → 再 preempt。B 在 preempt/recompute 循环中反复横跳，TPOT 从 45ms 暴涨到 210ms。

### 2.2 Chunked Prefill

1. **Chunked Prefill 不是消除 prefill 对 decode 的影响，而是用 TTFT 换 TPOT 稳定性**——长请求的首 token 延迟变长了，但同 batch 其他请求的 decode 不再被长时间阻塞。

2. **Chunk size 的选择是一个连续光谱，不存在全域最优值**——大 chunk 利于 TTFT 但 TPOT 差，小 chunk 反之。最优值取决于 SLA 优先级和 GQA 的 KV head 压缩比。

3. **Chunked Prefill 的 blocking 机制在有无 chunk 时本质不同**——无 chunk 时是 scheduling 串行（prefill 和 decode 调度阶段互斥），有 chunk 时是 compute 资源共享（prefill 计算占主导但 decode 可搭车）。

4. **FLOPs 估算公式 2×N×T 的适用边界**——当 T 较小时（如 256），attention FLOPs 仅占 ~0.03%；当 T 很大时（如 128K），attention FLOPs 占 80%+。

### 2.3 vLLM Preemption（Swap vs Recompute）

1. **Swap 和 Recompute 代价都随总 token 数线性增长**，斜率比由硬件参数决定，比值恒定（~2.46），不随 decode 推进变化。**"recompute 代价几乎不变"是错误的理解。**

2. **Swap 的核心价值不在"快"，而在"不争抢"**——recompute 消耗 GPU compute 和 prefill capacity（高峰期是稀缺资源）；swap 走 PCIe DMA 用的是闲置资源。高峰期 prefill capacity 打满时，recompute 有效 throughput 下降，swap 可能反超。

3. **Victim 选择比怎么 preempt 更重要**——所有请求的 blocks/ms 恢复效率相同（= prefill_throughput / block_size，硬件常数）。选 victim 应看绝对恢复时间、SLA 优先级、prefix 共享程度。

4. **正确的 preemption 实现要点**：动态 cost 比较、独立 swapped/waiting 队列、共享 blocks 分层处理、recompute 放队首防 starvation。

5. **数值自洽是出题底线**——64 blocks × 16 tokens/block = 1,024 < 4K prompt，物理不自洽导致整道题的分析失去根基。

### 2.4 Chunked Prefill 的调度博弈

1. **Chunked Prefill 的本质是 TTFT↔TPOT 光谱权衡，不是开关**——chunk size 是一个连续光谱，大 chunk 保 TTFT 但 TPOT 差，小 chunk 反之。不存在全局最优值，最优值由 SLA 优先级和 workload 组成决定。

2. **调度器优先级反转是 chunked prefill 场景下 decode starvation 的关键根因**——"先塞 prefill 再给 decode"的策略导致 8 个长请求可以连续 9 秒让 decode 进不来。正确做法是**先预留 decode slot**（min_decode_slots），剩余 budget 再分配 prefill。

3. **vLLM 和 SGLang 的调度哲学差异**——vLLM 以 SeqGroup 为调度单位 + token budget 分配机制（调度层和 cache 层解耦）；SGLang 以 Stream 为调度单位 + RadixCache 深度耦合（调度器显式感知 prefix 共享，可自动聚簇调度）。

4. **单一 chunk size 无法同时满足 TTFT 和 TPOT 双 SLA**——当 SLA 为 TTFT<15s 且 TPOT<80ms 时，chunk=512 和 chunk=4096 均不满足。这指向更高阶方案：PD 分离（从根本上解耦）、请求感知调度（长/短请求分不同 chunk size）、或自适应 chunk size。

5. **SGLang Stream 模型的核心优势**——stream 是轻量执行上下文，可在 RadixCache 节点上 fork/join，支持细粒度调度、prefix 感知聚簇、beam search 分支独立管理。相比 vLLM 的请求级调度粒度更细。

---

## 3. 本周掌握得不错的点

1. **Chunked Prefill 的核心 trade-off 理解到位**——用户能准确指出无 Chunked Prefill 时 S 的 TPOT 应为 667+25=692ms，并在 06/12 题目中进一步指出 chunk size 光谱的本质是 TTFT↔TPOT 的连续权衡。

2. **对 PD 分离场景下 Chunked Prefill 的独特视角**——用户准确指出"chunked prefill 在分离式场景没有收益"，并追问 PD 分离场景开启 chunked prefill 的目的，说明能区分集中式 vs 分离式部署下同一技术的不同作用机制。

3. **KV Cache 计算熟练**——用户能独立完成 2×layers×n_kv_heads×d_head×dtype_bytes 的计算，并正确考虑 TP=2 的切分。

3. **Swap vs Recompute 的 crossover 分析到位**——用户理解 crossover 在高带宽访存快于计算时 swap 优、算力强计算快于访存时 recompute 优。同时指出 `blocks/ms 应该是常数`，说明对线性关系的本质有准确直觉。

4. **精准识别 preemption 根因**——在 P800 题目分析中，用户直接指出"preemption 触发是乐观超额准入的兜底，不是 prefix cache 导致的"，准确抓住根本原因。

5. **能主动识别题目中的不自洽**——用户先后发现 36,500 blocks vs 96 请求的数值问题、64 blocks × 16 = 1,024 < 4K prompt 的物理矛盾。

6. **善于追问和纠正**——用户能精确指出回答中的逻辑漏洞（如"recompute 代价并非几乎不变"、"swap 的优势在资源不在时间"），推动了对 preemption 策略更深入的理解。

---

## 4. 本周薄弱点与误区

1. **"Preemption 是 prefix cache 导致的"因果混淆**
   - **误区**：看到 prefix cache hit rate 下降且日志有 prefix eviction，认为 prefix cache 是 preemption 的根源。
   - **正确理解**：Preemption 的根本原因是 admission 策略（max_num_seqs 过高），prefix cache 是受害者。
   - **工程后果**：方向性错误——去优化 eviction 策略而不限制准入，无法解决根本问题。

2. **"Recompute 代价几乎不变"的错误认知**
   - **误区**：认为 recompute 代价和 decode 产出无关，只和 prompt 长度有关。
   - **正确理解**：Recompute 需要 re-prefill 全部 token（prompt + output），output 越多序列越长，代价也线性增长。只是系数比 swap 小，而不是"不变"。
   - **工程后果**：在 long output 场景下高估 recompute 优势。

3. **"Swap 更快"的片面认知**
   - **误区**：从纯时间公式推出 swap 比 recompute 慢，就认为 swap 没有价值。
   - **正确理解**：Swap 不争抢 GPU compute 资源。高峰期 prefill capacity 打满时，swap 的确定性延迟反而更有优势。
   - **工程后果**：在高峰期只根据空闲时的基准测试选择 recompute，加剧拥堵。

4. **Victim 选择的释放效率误区**
   - **误区**：长 prompt 请求 blocks 多，释放效率高，是更好的 victim。
   - **正确理解**：所有请求的 blocks/ms 恢复效率相同（= prefill_throughput / block_size），是硬件常数。
   - **工程后果**：优先 preempt 长 prompt 虽然释放 blocks 多，但恢复时间长，对其他请求影响更大。

5. **Prefix Cache 与 Preemption 的交互理解不足**
   - **误区**：把 prefix cache 当作纯粹的"省显存"机制。
   - **正确理解**：共享 blocks（refcount>1）在 preemption 时既不能释放也不能 swap。
   - **工程后果**：preemption 策略如不考虑共享 blocks，可能因有效回收不足而触发连锁 preemption。

---

## 5. 温故知新：历史不明白知识回炉

### 5.1 Swap vs Recompute 的 crossover 分析

**一句话直觉**：Swap 像搬家（东西越多搬越久），Recompute 像重做（从头来但别人不能用厨房）。选哪个取决于"搬家时间 vs 重做时间"和"搬家期间别人能不能继续用厨房"。

**公式推导**：
```
swap_time = N × KV_bytes_per_token / PCIe_bw = N × 16.4μs
recompute_time = N / prefill_throughput = N × 6.7μs

crossover: swap_time = recompute_time → N(crossover) ≈ 123 blocks
但 4K prompt 至少 256 blocks，所以始终在 recompute 优势区
```

**应用场景**：
- 短 prompt + 少 decode：recompute 优
- 长 prompt + 多 decode + prefill 空闲：recompute 优（系数小 2.46×）
- 长 prompt + 多 decode + prefill 拥堵：swap 可能优（不抢 prefill capacity）
- 无 Chunked Prefill：swap 优（避免阻塞全部 decode）

**常见坑**：
- ❌ 认为 recompute 只算 prompt 长度
- ❌ 认为 swap 一定慢
- ❌ 用固定阈值代替动态决策

### 5.2 Chunked Prefill 的核心本质

**一句话直觉**：不切块的 prefill 像一次倒一桶水（快但别人得等着），切块的 prefill 像用水杯一杯杯倒（慢一点但别人能同时用水龙头）。

**机制对比**：
```
无 Chunked Prefill:  Prefill ────→ Decode ────→ Prefill ────→ Decode
                     (prefill 阶段所有 decode 暂停)

有 Chunked Prefill:  [prefill_chunk + decode_A] → [prefill_chunk + decode_B] → ...
                     (每 step 既做 prefill 也做 decode)
```

**chunk size 选择光谱**：
- 大 chunk（如 1024）：TTFT 好，TPOT 差
- 小 chunk（如 64）：TTFT 差，TPOT 好
- GQA（8 KV heads vs MHA 32 heads）：KV Cache 减 4× → 更大 chunk size 选择空间

### 5.3 Prefix Cache 的 eviction 连锁反应

**一句话直觉**：Prefix cache 像共享书架上的书。被 evict 后第一个来借的人需要重新买一本放回去，后面的人就能直接借到了。

**恢复机制**：Evict → A1 miss → A1 re-prefill 2K → prefix 挂回 RadixTree → A2/A3... cache hit

**什么时候持续劣化**：evict 频率 > prefix 恢复速度（B 持续到达反复触发 preemption）

**常见坑**：
- ❌ "evict 一次，所有后续 A 都需要重新 prefill"（实际第一个 A 恢复后全部 hit）
- ❌ 忽略 RadixCache 的自动恢复能力

### 5.4 Preemption 代码中的状态管理

**一句话直觉**：Preemption 不只是"释放内存"，而是"把请求从一个队列移到另一个队列"——放错队列比不放更糟。

**正确状态流转**：
```
running ──→ swapped（swap 后必须移出 running）
running ──→ waiting（recompute 后放队首，不是队尾）
swapped ──→ running（swap in 恢复后放回 running）
```

### 5.5 Preemption 的根因分析：Admission Control > Eviction Strategy

**一句话直觉**：preemption 像电影院超卖座位——问题不是"怎么赶人走"，而是"一开始就不该卖那么多票"。

**核心逻辑链**：
```
max_num_seqs=96 × 平均~421 blocks/请求 > 36,500 总 blocks
  → scheduler admit 满 96 个请求
    → blocks 不够 → preemption 触发
      → 释放 blocks 给新请求
        → 新请求 admit 后 blocks 又不够 → 再 preempt...
```

**工程启示**：
- Admission control 是第一道防线，eviction 策略优化是第二道防线
- 两道防线都要做，但不能顺序搞反

---

## 6. 下周学习建议

1. **深入 Preemption 的生产实践**：从代码层面验证 vLLM 实际的 preemption 实现（`vllm/v1/core/scheduler.py` 中的 `_preempt()` 逻辑）。

2. **补齐 06/13~06/14 的题目**：本周已完整回顾 06/08~06/12 共 5 道题，06/13~06/14 的题目尚未查看。建议从这两天的题目入手，检验对 PD 分离、CUDA Graph 等高阶主题的理解。

3. **尝试 PD 分离专题**：Preemption 策略在 PD 分离架构下会有本质不同（KV Cache 传输受网络带宽约束）。

4. **算子基础补强**：从 Triton 入手，用 `llm-inference-learning` 的 L009/L010 题目做填空练习。

5. **验证 PCIe 带宽实测**：在集群上用 `nvbandwidth` 或 `cudaMemcpy` 实测 GPU↔CPU 的 PCIe 有效带宽。

---

## 7. 下周推荐练习题

1. **L007 - 调度方案设计（⭐⭐⭐）**
   - 综合运用本周 preemption 策略思考，将 swap/recompute 权衡落到方案设计

2. **L014 - 显存优化方案设计（⭐⭐⭐）**
   - 巩固 KV Cache 预算估算能力，理解 max_num_seqs/gpu_memory_utilization 配置逻辑

3. **L020 - CUDA Graph 捕获失败排查（⭐⭐⭐⭐）**
   - 涉及 CUDA Graph 捕获的常见失败模式，是通往底层优化的第一步

---

## 8. 记录信息

- **目标仓库**：`https://github.com/zccjjj/inferenceEx`
- **建议文件路径**：`weekly-reviews/2026-W24-weekly-review.md`
- **提交方式**：用户手动提交

```bash
git add weekly-reviews/2026-W24-weekly-review.md
git commit -m "add weekly inference review 2026-W24"
git push
```
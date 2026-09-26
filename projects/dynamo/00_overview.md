# Overview — dynamo（ARCH-2026-09-20-001）

## 项目一句话
NVIDIA 开源数据中心级推理栈 Dynamo（v1.6.0，Apache-2.0）——推理引擎（SGLang/TRT-LLM/vLLM）**之上**的编排层：KV 感知路由 + 调度 + 多层级 KV 缓存管理 + 自动扩缩，把 GPU 集群变成协调推理系统。**Rust 性能核心 + Python 扩展层 + K8s 部署层**。

## 本次考古核心命题
**"agent 的推理成本与延迟如何成为可工程化的系统问题？"**——agentic 工作负载（coding agent 序列模式 / multi-agent fan-out）与 chat 负载特征不同，round-robin 路由"盲目"；KV cache 价值异质、生命周期短暂（subagent/thinking）；harness 拥有基础设施看不到的全局上下文。

## 关键发现（Top 5）
1. **Agent Hints（nvext）**：priority / osl / speculative_prefill 三信号让 harness 上下文第一次跨 API 边界进入路由与缓存决策——"匿名 token 请求是对 agent 负载的信息浪费"
2. **KV 价值分层**：系统提示（最高）vs thinking tokens（占生成 40% 但循环关闭即死）——uniform LRU 对 agentic 负载结构性失效；双结构 evictor（LRU + priority queue）+ session tagging
3. **KV 全局共享化**：4 层内存层级（GPU→CPU→NVMe→NIXL RDMA）+ write-through + sequence hash 去重——subagent 冷启动从 4 次冗余 prefill 变 1 次 compute + 3 次 load
4. **Flash Indexer**：六次迭代到 170M ops/s 的并发全局 KV 块索引（压缩 radix 家族：split 不 merge）
5. **调度双路径 + DRR**：direct path（never-queues/零 worker）vs policy-queue path（BinaryHeap 优先级 + 加权轮转 + deadline 语义）；NoEndpoints 诚实失败

## 质量指标
Facts 35 ｜ EK 34（≥68 边，0 游离）｜ KO 9（L2×1/L3×6/L4×2，aggregation 100%）｜ Candidates 8 ｜ Flows 7/7 ｜ 反例 22

## 交付
- run_id: ARCH-2026-09-20-001
- repository: https://github.com/ai-dynamo/dynamo.git
- commit: `6562e7d04b04ad177459f8a5b720718de71e1a6a`（2026-09-19）
- skill_version: knowledge-archaeology v3.2
- 与 corpus 对照：hermes-agent（harness 侧 prompt cache 神圣性）↔ Dynamo（推理侧 cache 感知调度）——KO-04 上下文经济学双面闭合；letta-code（fail-closed 治理）↔ Dynamo（诚实失败 NoEndpoints）——KO-03 同构

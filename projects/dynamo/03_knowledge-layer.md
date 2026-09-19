# Knowledge Layer — dynamo（ARCH-2026-09-20-001）

> KO 从 EK Graph 簇生成，每条声明 aggregation_rule（R1-R4）+ 反例预算（L3+ ≥3 定向反例，0 反例须给出搜索证据）。Epistemic 状态严格区分：Fact/Observation/Hypothesis/Pattern/Cognitive Model/Principle。

## KO-01 块价值分层（R3 不变量簇）——L3 Pattern
**一句话**：推理缓存的统一 LRU 对 agentic 负载结构性失效——KV 块价值高度异质（系统提示每 turn 复用 vs thinking 循环关闭即死），必须按价值分层保留。
- aggregation_rule: R3（EK-14/EK-15/EK-17/EK-29 汇聚到"块价值≠recency"不变量）
- 解释范围扩大：从 Dynamo 的 KVBM 到任何带缓存的 agent 推理系统（SGLang HiCache / vLLM prefix cache / Anthropic prompt caching）
- 反例：①单 GPU 单模型场景引擎足够（README 明示）②thinking 占生成 40% 但价值近零——高生成量≠高价值 ③LRU 对 chat 负载（短会话少复用）并非失效
- Epistemic: **Pattern**（Dynamo 生产验证 + 同类 SGLang/TRT-LLM 对照，跨项目结构一致）

## KO-02 记忆面与执行面解耦（R4 主题簇）——L4 Cognitive Model
**一句话**：在 agent 推理栈中，"知道什么（KV 记忆）"与"在哪执行（worker 调度）"必须解耦为独立维度，由全局索引统一编排。
- aggregation_rule: R4（EK-01/EK-02/EK-03/EK-12/EK-13/EK-19 围绕"编排层把记忆与执行分开治理"主题）
- 解释范围扩大：Dynamo（KV 全局索引 vs worker 选择）↔ hermes-agent（记忆冻结 vs 即时生效）↔ letta-code（记忆文件树 vs 执行权限）——三项目独立收敛到同一认知
- 反例：①hermes 把记忆写盘与 prompt 投影耦合（快照模式不投影）②letta 记忆与 git 耦合（每 agent 一仓库）——耦合是现实，解耦是目标 ③单 worker 部署无解耦需求
- Epistemic: **Cognitive Model**（三项目独立验证，但跨项目对照为归纳，标注 pending）

## KO-03 诚实失败不变量（R3 不变量簇）——L3 Pattern
**一句话**：调度/索引子系统对"无可用资源/无法满足"必须显式报错（NoEndpoints），绝不假装成功或无限等待。
- aggregation_rule: R3（EK-21/EK-22/EK-33 汇聚"诚实状态报告"不变量）
- 解释范围扩大：与 letta-code fail-closed（无审批回退路径则内核强制拒绝）同构——"系统不知道怎么办时必须明说"
- 反例：①cache_control 兼容但明确不支持（兼容≠承诺）②scaled-to-zero 报真实条件而非降级 ③sgl-router 的 approximate cache history 被替换为 authoritative device-KV overlap——用真实数据替代近似
- Epistemic: **Pattern**（Dynamo + letta 双项目，跨项目结构一致）

## KO-04 Agent 上下文跨 API 边界（R4 主题簇）——L4 Cognitive Model
**一句话**：推理基础设施的效率天花板取决于 harness 全局上下文（谁阻塞/剩余 turn/价值高低）能否结构化地跨越 API 边界——匿名 token 请求是对 agent 负载的信息浪费。
- aggregation_rule: R4（EK-16/EK-17/EK-18/EK-19/EK-20 围绕"harness 知识入基础设施"主题）
- 解释范围扩大：nvext.agent_hints（priority/osl/speculative_prefill）是 MCP/A2A 之外的第三协议面——"执行协议"面；对照 hermes 把 prompt cache 神圣化——两边都承认"上下文是稀缺资产"
- 反例：①cache_control 不被接受（不是所有 harness 信号都被采纳）②agent_hints 是 v1 社区共设计（未稳定）③prefetch hooks 还在建（"We are building"）
- Epistemic: **Cognitive Model**（Dynamo 明确论证 + 与 hermes/letta 上下文经济学交叉印证）

## KO-05 radix 压缩家族（R1 机制簇）——L3 Pattern
**一句话**：KV 前缀索引的演进是"radix 压缩 + 并发化 + 不合并"三个意图的迭代组合——压缩减少节点、并发化支撑 170M ops/s、不合并保简单性。
- aggregation_rule: R1（EK-04/EK-23/EK-25/EK-26 共享 radix/压缩机制，跨 indexer/pruning/cleanup 三子系统）
- 解释范围扩大：RadixTree→RadixTreeIndex→BranchSharded→ConcurrentRadixTreeCompressed 六代同族；与 vLLM prefix cache 的 radix 选择同构
- 反例：①split 永不 merge（对高碎片负载是代价）②sticky internal nodes 保留已删 fanout 点（空间代价）③ThreadPoolIndexer 是"非 radix"替代路径
- Epistemic: **Pattern**（Dynamo 内多实例 + vLLM 同类）

## KO-06 编排层哲学：不做引擎，做协调（R4 主题簇）——L3 Pattern
**一句话**：推理编排层通过"增量价值"定位（disagg/路由/KV 共享/自动扩缩）而非替代引擎赢得生态位——单 GPU 用户明确不需要它。
- aggregation_rule: R4（EK-01/EK-09/EK-27/EK-28/EK-34 围绕"编排层职责"主题）
- 解释范围扩大：与 hermes 的"多前端单核心"（不重造 CLI 生态）、letta 的"不重造 agent runtime 而做记忆层"同模式——**分层产品以互补而非替代进入生态**
- 反例：①NAT bandit 路由计划内置（编排层也在向上吃策略空间）②ai-dynamo wheel 依赖 transformers/kubernetes（与引擎层有耦合）
- Epistemic: **Pattern**（Dynamo/hermes/letta 三项目）

## KO-07 KV 全局共享化（R2 因果链簇）——L3 Pattern
**一句话**：把 KV cache 从"每 worker 本地临时资源"提升为"全局共享不可变寻址资源"，用一次 compute+多次 load 消除 subagent 冷启动冗余 prefill。
- aggregation_rule: R2（EK-11→EK-12→EK-13→EK-24→EK-31 因果链：条件拆分布置→4 层 write-through→去重注册→disagg 一致性→传输协议）
- 解释范围扩大：与分布式系统"共享不可变数据"通用模式（内容寻址存储）同构；对照 SGLang HiCache
- 反例：①4 层层级仍在建设中（KVBM 是 toward 4-tier）②NIXL RDMA 依赖硬件 ③lead 79.4% vs explore 91.3% 的差距没有全消除
- Epistemic: **Pattern**（设计意图明确 + 部分实现，标记实现中）

## KO-08 性能工程方法论（R4 主题簇）——L3 Pattern
**一句话**：把数据结构做到物理极限（170M ops/s）的方法是"测量驱动迭代"——六代索引每次解决一个可测量的瓶颈，直到网络/hashing 成为瓶颈。
- aggregation_rule: R4（EK-03/EK-09/EK-27/EK-28 围绕"性能目标驱动设计"主题）
- 解释范围扩大：与 letta 的 benchmark 门、hermes 的 WAL 基准同方法论——**性能是设计输入不是验收后验证**
- 反例：①bench 环境依赖（170M ops/s 复现性未独立验证）②部分优化路径（jump-optimized spatial）复杂度高
- Epistemic: **Pattern**（方法论在 Dynamo 内多次实例化）

## KO-09 调度数据结构族（R4 主题簇）——L2 Engineering Knowledge
**一句话**：Dynamo 调度 = 四容器类队列（pending/ready_by_worker/blocked/heads）+ BinaryHeap 优先级 + DRR 加权轮转 + deadline 语义的组合。
- aggregation_rule: R4（EK-06/EK-08/EK-09/EK-10/EK-30/EK-32 围绕"调度数据结构"主题）
- 反例：①direct path 绕过队列 ②threshold 以下 bypass ③单类 profile 仍走 PolicyQueue（DRR 无跨类效果）
- Epistemic: **L2 工程知识**（不升维——这是实现组合，跨项目可迁移性未验证）

---

## 层级分布
| KO | Layer | Epistemic | aggregation_rule |
|---|---|---|---|
| KO-01 | L3 | Pattern | R3 |
| KO-02 | L4 | Cognitive Model（cross-project pending） | R4 |
| KO-03 | L3 | Pattern | R3 |
| KO-04 | L4 | Cognitive Model | R4 |
| KO-05 | L3 | Pattern | R1 |
| KO-06 | L3 | Pattern | R4 |
| KO-07 | L3 | Pattern（实现中） | R2 |
| KO-08 | L3 | Pattern | R4 |
| KO-09 | L2 | Engineering Knowledge | R4 |

## 反例预算
- 每个 L3+ KO ≥3 定向反例 ✓（KO-01~08 均列出 ≥3）
- 0 反例 KO：无

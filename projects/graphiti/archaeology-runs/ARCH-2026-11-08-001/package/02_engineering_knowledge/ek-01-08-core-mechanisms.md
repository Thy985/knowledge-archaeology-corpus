# Engineering Knowledge · Core Mechanisms (EK-01 ~ EK-08)

## EK-01 写入是固定七阶段的编排管线，而非"一次 LLM 调用后写库"
- **type**: core-mechanism · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: `add_episode` 严格按 Scope→Context→Episode→Extract Nodes→Resolve Nodes→Extract+Resolve Edges→Persist 推进；每个阶段的产物是下一阶段的输入，节点/边解析结果通过 `uuid_map` 与 `node_episode_index_map` 贯穿。
- **evidence**: E-11, E-12, E-13, E-14, E-15, E-16, E-17
- **links**: `subsystem:EK-02`, `subsystem:EK-03`, `subsystem:EK-04`, `subsystem:EK-07`, `causal:EK-13`
- **note**: 顺序不可打乱——边解析依赖节点 uuid_map；属性抽取依赖新边集合。

## EK-02 Episode 是唯一的 provenance 根，事实不直接来自 LLM 文本
- **type**: core-mechanism · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: 原始内容保存在 `EpisodicNode.content`，不被改写；派生实体/边通过 `EpisodicEdge(MENTIONS)` 与 `EntityEdge.episodes[]` 双向回溯到 episode。因此"这条事实是谁说的/从哪来"始终可查。
- **evidence**: E-05, E-10
- **links**: `constraint:EK-12`, `constraint:EK-24`, `mechanism:EK-03`

## EK-03 事实用四个时间字段表达"双时间 + 逻辑删除"
- **type**: core-mechanism · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: 四个时间字段中，`valid_at/invalid_at` 是业务时间窗，`created_at/expired_at` 是系统时间；`expired_at` 只表示"不再作为当前事实"，不删数据。事实变更 = 新增一条边 + 旧边置 `invalid_at`。
- **audit_note**: "双时间（bi-temporal）"是分析术语，仓库未使用该词；此处描述的是字段结构，不等同于学术定义。
- **evidence**: E-10, E-13
- **links**: `causal:EK-04`, `mechanism:EK-12`, `contrast:EK-31`

## EK-04 失效判定是"四边界比较"，且只在特定顺序下生效
- **type**: core-mechanism · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: `_apply_temporal_invalidation` 先按 `valid_at` 升序排候选；若候选起始**晚于**新边起始，则新边被置 `invalid_at=candidate.valid_at` 并 `expired_at=now`。`resolve_edge_contradictions` 仅在"候选在新区间之外"或"候选更早"时把候选置失效；两段事实时间重叠时**不互相失效**。
- **evidence**: E-26, E-27, E-28
- **links**: `dependency:EK-03`, `causal:EK-11`, `constraint:EK-28`
- **note**: 所有比较前要求"调用方必须先给边打时间戳"（代码注释原文），时间戳由 LLM 抽取（E-28）——这是整条链最脆弱的输入点。

## EK-05 请求级作用域：每次调用持有自己的 driver/clients，禁止改写共享 `self.driver`
- **type**: core-mechanism · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: `_resolve_request_scope` 对非默认 group 返回 `driver.clone(database=gid)` + `clients.model_copy(update={'driver':driver})`，并沿调用链显式传递；默认库路径直接复用共享实例（不 clone、不 copy）。
- **evidence**: E-18, E-19
- **links**: `constraint:EK-06`, `mechanism:EK-06`, `causal:EK-33`

## EK-06 `group_id` 既是逻辑租户，也是物理分库键（仅 FalkorDB）
- **type**: core-mechanism · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: 所有节点/边带 `group_id`；Neo4j/Neptune 在同库内按 `group_id` 过滤，FalkorDB 则"一个 group 一个 graph"，因此任何读写都必须把 driver 路由到对应 graph，否则静默返回空结果（#1659）。
- **evidence**: E-04, E-09, E-18, E-19
- **links**: `subsystem:EK-05`, `dependency:EK-17`, `contrast:EK-08`

## EK-07 实体去重是"确定性优先、LLM 兜底"的四段阶梯
- **type**: core-mechanism · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: 候选三路合并（identity properties / 精确归一化 name / 语义 cosine≥0.6）→ 逐节点先 identity 匹配、再余弦匹配；两者都未决的才进 LLM 批量裁决；无候选则视为新实体。LLM **不是**首选裁判。
- **evidence**: E-20, E-22, E-23, E-24
- **links**: `causal:EK-08`, `constraint:EK-09`, `contrast:EK-10`
- **note**: 这是"成本/延迟优先"的设计：LLM 调用按"未决集合"批量发生，而非每实体一次。

## EK-08 去重有明确预算：候选 15、余弦 0.6，且能力缺失时逐驱动降级
- **type**: core-mechanism · **layer**: L2 · **value**: B · **confidence**: high
- **statement**: `NODE_DEDUP_CANDIDATE_LIMIT=15` 限制每个实体的候选池，`NODE_DEDUP_COSINE_MIN_SCORE=0.6` 决定进入候选的语义门槛。identity lookup / 语义检索 / batch embedding 在未实现时抛 `NotImplementedError`，被捕获后返回空集或逐条退化——功能降级而非错误上抛。
- **evidence**: E-21, E-22, E-41
- **links**: `constraint:EK-07`, `mechanism:EK-17`, `dependency:EK-17`

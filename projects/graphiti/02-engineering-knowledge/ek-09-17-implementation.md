# Engineering Knowledge · Key Implementation (EK-09 ~ EK-17)

## EK-09 边类型签名：要么丢弃，要么降级为通用关系（两种失败策略由开关决定）
- **type**: implementation · **layer**: L2 · **value**: B · **confidence**: high
- **statement**: `edge_type_map` 以 `(源类型, 目标类型)` 为键。若边名属于"已声明签名的类型"但不在该端点对允许集合内：`strict_edge_types=True` → **丢弃该边**；否则 → 改名 `DEFAULT_EDGE_NAME` 并清空 `attributes`。未声明签名的类型任意实体间可用。
- **evidence**: E-25
- **links**: `constraint:EK-10`, `contrast:EK-10`, `subsystem:EK-01`
- **note**: 默认 `False`（宽松），意味着"模型抽错关系名"不会被丢弃，而是丢掉结构属性、保留一条语义模糊的边——精度换召回。

## EK-10 矛盾消解只在"时间不重叠"时执行，重叠即共存
- **type**: implementation · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: 两条互斥事实若**无法证明先后**（区间重叠或时间缺失），系统不裁决，两者都保留为"并存事实"；只有当候选明确更早、或其区间明确在新区间之外时才触发失效。
- **audit_note**: "重叠即共存"是简化表述；实现的判据是"无法证明先后 → 不裁决"。
- **evidence**: E-26, E-27
- **links**: `contrast:EK-04`, `contrast:EK-09`, `dependency:EK-03`
- **note**: 这是保守策略：宁可保留冲突，也不凭"更新更可信"武断判定——把裁决权交回读取端/应用。

## EK-11 边时间戳由 LLM 从 episode + reference_time 抽取
- **type**: implementation · **layer**: L2 · **value**: B · **confidence**: high
- **statement**: `_extract_edge_timestamps` 在边抽取后单独调用 LLM 推断 `valid_at/invalid_at`，再交给失效算法。时间正确性依赖模型对相对时间（"上周""as of March"）的解析。
- **evidence**: E-28
- **links**: `causal:EK-04`, `dependency:EK-12`
- **note**: 与"LLM 只做未决裁决"（EK-07）相反，时间抽取**完全依赖 LLM**，且没有确定性校验——这是双时间模型的主要失真来源。

## EK-12 Saga：用 episode 分组 + 增量总结压缩长历史
- **type**: implementation · **layer**: L2 · **value**: B · **confidence**: high
- **statement**: `saga` 可以是名字或 `SagaNode`；同 group 同名复用。episode 通过 `HAS_EPISODE` 挂到 saga，连续 episode 通过 `NEXT_EPISODE` 串联。`summarize_saga` 更新 `summary` 并记录 `last_summarized_at` / `last_summarized_episode_valid_at`；SagaNode 内置 fetched/selected/skipped 计数与字节预算。
- **audit_note**: "只总结新增 episode" 的增量语义**未验证**（字段命名暗示，但乱序 episode 下是否成立未知）→ 降为 `Hypothesis`，见 CAND-06。
- **evidence**: E-08, E-29, E-30
- **links**: `mechanism:EK-13`, `dependency:EK-02`, `constraint:EK-16`

## EK-13 实体摘要与属性是"增量 hydrate"，且只喂新边
- **type**: implementation · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: 每次 add_episode 对涉及节点重新生成/更新 `summary` 与 `attributes`；摘要生成只接收**本次新产生的边**（注释：避免把图里已有的事实重复喂给模型）。
- **evidence**: E-06, E-16
- **links**: `causal:EK-01`, `contrast:EK-12`, `dependency:EK-02`
- **note**: 摘要不是全量重算，因此摘要反映的是"累积演化的区域摘要"，不保证与全图一致。

## EK-14 检索是"配方制"：16 个预置 SearchConfig，按调用意图切换
- **type**: implementation · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: 共 16 个常量：`COMBINED_HYBRID_SEARCH_{RRF,MMR,CROSS_ENCODER}`（3）、`EDGE_HYBRID_SEARCH_{RRF,MMR,NODE_DISTANCE,EPISODE_MENTIONS,CROSS_ENCODER}`（5）、`NODE_*`（5）、`COMMUNITY_HYBRID_SEARCH_{RRF,MMR,CROSS_ENCODER}`（3）。三种对象 × 多种融合策略：RRF（倒数排名融合）、MMR（多样性重排）、NODE_DISTANCE（中心节点距离）、EPISODE_MENTIONS（提及热度）、CROSS_ENCODER（重排模型）。默认 `EDGE_HYBRID_SEARCH_RRF`；传 `center_node_uuid` 则切 `NODE_DISTANCE`。
- **evidence**: E-31, E-32, E-34, E-35
- **links**: `subsystem:EK-15`, `contrast:EK-15`, `dependency:EK-17`
- **note**: 检索不依赖 LLM 摘要（README 明示），这是与 GraphRAG 的核心差异。

## EK-15 图遍历（BFS）是与向量检索并列的一等方法，但必须有起点
- **type**: implementation · **layer**: L2 · **value**: B · **confidence**: high
- **statement**: `bfs` 是 edge/node search 的一等方法；未给 `bfs_origin_node_uuids` 时代码有专门分支处理（不能凭空全图 BFS）。`center_node_uuid` 则用于"围绕某实体的邻近事实"重排。
- **evidence**: E-34, E-35, E-32
- **links**: `mechanism:EK-14`, `constraint:EK-14`

## EK-16 分块器对 JSON 结构敏感：按元素/字段切分并带 overlap
- **type**: implementation · **layer**: L2 · **value**: B · **confidence**: high
- **statement**: `chunk_json_content` 用 `CHARS_PER_TOKEN=4` 估算；JSON 数组按元素切、JSON 对象按字段切，块间保留 `overlap_tokens` 的元素（`_get_overlap_elements`）；非 JSON 走文本切分。
- **evidence**: E-47
- **links**: `mechanism:EK-12`, `dependency:EK-17`
- **note**: 4 chars/token 是跨 provider 的保守估值，对中文/代码会低估 token 数；分块阈值因此偏保守。

## EK-17 驱动能力是"按需实现"的：缺失能力用异常表达，由上层降级
- **type**: implementation · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: `GraphDriver` 是 ABC（session/execute/transaction/clone/build_fulltext_query）；后端未实现的能力抛 `NotImplementedError`。上层在 identity lookup、语义检索、batch embedding、`clone` 等点捕获该异常并回退（空集/逐条/不路由）。
- **evidence**: E-38, E-39, E-40, E-41
- **links**: `mechanism:EK-08`, `constraint:EK-06`, `contrast:EK-22`
- **note**: 优点是新后端可渐进接入；代价是"能力缺失"与"真的没有结果"在返回值上不可区分（都表现为空列表）。

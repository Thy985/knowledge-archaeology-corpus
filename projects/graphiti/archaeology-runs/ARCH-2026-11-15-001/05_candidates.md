# 05 · Candidates — Hypotheses, Boundaries, Cross-Project Candidates

> **纪律**：以下内容不进入 KO。任何 `Hypothesis` 都必须带“缺失证据 + 验证路径”。不得把 Cross-project Candidate 写成已验证 Principle。

## 1. Runtime Hypotheses（本项目，未运行验证）

### C-01 · Equal-`valid_at` contradictions remain jointly current

- **Type**: Boundary / Truth maintenance
- **Status**: `Hypothesis (logic-derived, unexecuted)`
- **Claim**: 当新旧事实的 `valid_at` 完全相同，`resolve_edge_contradictions` 的两个跳过条件均成立，双方都不会被置 `invalid_at`，因此同一时刻可存在两条互斥“当前事实”。
- **Evidence**: EV-036/037/039; EK-08.
- **Missing evidence**: 没有构造同 `valid_at` 的冲突边并检查查询结果。
- **Validation path**: 用内存/mock 图或 Neo4j 测试容器，写入两条同端点、同 `valid_at`、语义互斥的边，执行 `search` 并检查两条边的 `invalid_at`。
- **Impact**: 若成立，KO-02 的“代码裁决”存在未定义区间；需要在时间相等时增加 tie-break 规则（如 `created_at`/置信度/人工优先级）。
- **Target**: KO-02, KO-03.

### C-02 · Default top-k can surface invalid facts ahead of current ones

- **Type**: Retrieval / Truth
- **Status**: `Hypothesis`
- **Claim**: 默认检索不过滤 `invalid_at`，且排序信号（bm25/cosine/RRF）不包含“当前有效性”，因此过期事实可能进入 top-k 并占据上下文。
- **Evidence**: EV-066/063; EK-36/37.
- **Missing evidence**: 没有构造“旧事实高分、新事实低分”的数据集并比较 top-k。
- **Validation path**: 构造 20 条旧事实 + 1 条新事实，旧事实与 query 高度相似；比较默认 search 与加 `valid_at` 过滤的 search 返回集。
- **Impact**: 若成立，KO-03 的“读时投影”需要默认 current 模式，或在排序中加入有效性惩罚。
- **Target**: KO-03.

### C-03 · LLM context lacks provenance IDs, weakening grounded citation

- **Type**: Evidence / Agent behavior
- **Status**: `Hypothesis`（机制为 Fact，行为影响为 Hypothesis）
- **Claim**: 上下文只含 `fact`、`valid_at`、`invalid_at`，不含 edge UUID/episode UUID；模型无法在回答中引用具体证据，也难以自行核验来源。
- **Evidence**: EV-113/114; EK-57.
- **Missing evidence**: 没有测量模型在有/无 provenance 时的引用准确性。
- **Validation path**: 同一组事实，分别提供“仅日期”与“附 episode UUID/source_description”的上下文，比较模型回答的引用准确率与拒答率。
- **Impact**: 若成立，KO-07 的证据链在“模型可见层”断裂；应在 context 中加入可点击/可查询的 source 标识。
- **Target**: KO-07.

### C-04 · `store_raw_episode_content=False` makes later audit impossible

- **Type**: Evidence / Retention
- **Status**: `Hypothesis`
- **Claim**: 关闭原文后，episode 只保留 `name`/`source_description`/时间/边引用，无法重建原始文本；对需要事后审计的系统，这等价于不可逆证据丢失。
- **Evidence**: EV-016/030; EK-29/30.
- **Missing evidence**: 没有验证是否存在其他副本（外部源、日志、对象存储）。
- **Validation path**: 以 `False` 写入一条 episode，读取 `get_episode`/`search` 的 episode 内容，确认为空；检查 `source_description` 是否足以定位原文。
- **Impact**: 若成立，KO-07 需要求“至少存内容哈希 + 来源 URI”。
- **Target**: KO-07.

### C-05 · MCP restart drops accepted-but-unprocessed episodes

- **Type**: Durability
- **Status**: `Hypothesis`
- **Claim**: `asyncio.Queue` 只在进程内；若进程在 worker 处理前退出，调用方已收到成功，但 episode 从未写入图。
- **Evidence**: EV-100/102/104; EK-24/26/27.
- **Missing evidence**: 没有做“提交后立即 kill 进程”的实验。
- **Validation path**: 启动 MCP server，提交一条会触发慢 LLM 的 episode，立即重启进程，再查询该 episode 是否存在。
- **Impact**: 若成立，KO-05 的“ack 持久性”必须改为持久 outbox。
- **Target**: KO-05.

### C-06 · `add_triplet` can create facts without an episode anchor

- **Type**: Evidence
- **Status**: `Hypothesis`（代码路径为 Fact，数据后果为 Hypothesis）
- **Claim（已按 Independent Validation 修正）**: 不传 episode 时，`add_triplet` 构造一个**未持久化**的 synthetic `EpisodicNode`（默认生成新 UUID），`append_episode_provenance` 会把该 UUID 追加到 `edge.episodes`。因此 edge 的 `episodes` 不是空列表，而是包含一个**悬空引用**：指向图中不存在的 episode。该事实没有可回读的源 episode，但看起来像有溯源。
- **Evidence**: EV-024/050; EK-35; `edge_operations.py:884-890`; `graphiti.py:2140-2156`.
- **Missing evidence**: 没有实际写入并查询 `edge.episodes`，也没有验证 `get_by_uuids` 对悬空 UUID 的返回。
- **Validation path**: 调用 `add_triplet` 不传 episode，随后 `get_by_uuid` 读取 edge，检查 `episodes` 字段；再用该 UUID 调 `EpisodicNode.get_by_uuid`，应观察到 `NodeNotFound`。
- **Impact**: KO-07 的“证据锚”必须由写入 API 强制；当前行为是**伪溯源**，比空列表更危险。
- **Target**: KO-07.

### C-07 · `remove_episode` leaves derived temporal invalidation in place

- **Type**: Lifecycle / Rollback
- **Status**: `Hypothesis`
- **Claim**: 删除一个曾导致旧事实失效的 episode，不会恢复旧事实的 `invalid_at`/`expired_at`；旧事实仍是“历史”，不会回到“当前”。
- **Evidence**: EV-051/052; EK-33.
- **Missing evidence**: 没有构造“A 使 B 失效 → 删除 A → 查询 B”的序列。
- **Validation path**: 写入 B，写入 A 使 B 失效，删除 A，检查 B 的时间字段与 search 可见性。
- **Impact**: 若成立，KO-03/KO-07 需要“删除时重算失效闭包”或明确声明删除不回滚。
- **Target**: KO-03, KO-07.

### C-08 · Community rebuild can leave an empty community layer

- **Type**: Failure / Availability
- **Status**: `Hypothesis`
- **Claim**: `build_communities` 先 `remove_communities`，再构建；若构建/LLM 失败，目标分组的社区已被删除且未重建。
- **Evidence**: EV-054/055; EK-22.
- **Missing evidence**: 没有注入 LLM 失败。
- **Validation path**: mock LLM 抛错，调用 `build_communities`，随后查询 community 节点数量。
- **Impact**: 若成立，KO-06 需要“先构建到临时命名空间，再原子切换”或保留旧社区直到新社区成功。
- **Target**: KO-06.

### C-09 · Bulk phase-2 failure leaves orphan episodes

- **Type**: Failure / Recovery
- **Status**: `Hypothesis`
- **Claim**: phase-1 已写入 episodes；phase-2 抽取/写入失败后，这些 episode 仍会被 `retrieve_episodes` 当作前序上下文，但缺少实体/边。
- **Evidence**: EV-091/092/070; EK-19/20.
- **Missing evidence**: 没有注入 phase-2 失败。
- **Validation path**: mock `resolve_extracted_edges` 抛错，执行 `add_episode_bulk`，再查询 episodes 与 entities/edges 数量。
- **Impact**: 若成立，KO-06 需要“可恢复的两阶段状态机 + 孤儿清理”。
- **Target**: KO-06.

### C-10 · Atomicity claim is driver-dependent

- **Type**: Portability
- **Status**: `Hypothesis`（代码差异为 Fact，实际部分写入为 Hypothesis）
- **Claim**: 在 FalkorDB/Kuzu/Neptune 上，`execute_write` 内多条语句中途失败可能留下部分写入；只有 Neo4j 有真正回滚。
- **Evidence**: EV-080-084; EK-18.
- **Missing evidence**: 没有在非 Neo4j 后端注入错误。
- **Validation path**: 对每个后端构造“第一条语句成功、第二条语句失败”的批量写，检查第一条是否持久化。
- **Impact**: 若成立，KO-06 的“能力矩阵”必须成为发布门槛。
- **Target**: KO-06.

### C-11 · `group_id` as database name can collide with existing databases

- **Type**: Scope / Configuration
- **Status**: `Hypothesis`
- **Claim**: 非默认 `group_id` 会被用作 database/graph 名；若与运维已有的数据库同名，可能写入非预期库。
- **Evidence**: EV-006/043/078; EK-43.
- **Missing evidence**: 没有检查部署层的 database 命名策略。
- **Validation path**: 在 Neo4j 中预建同名 database，传入对应 `group_id`，观察写入位置。
- **Impact**: 若成立，KO-04 需要求 `group_id → database` 映射表与保留名策略。
- **Target**: KO-04.

### C-12 · LLM false-negative contradictions leave stale facts current

- **Type**: Model error
- **Status**: `Hypothesis`
- **Claim**: prompt 再严格也是软约束；模型漏判时，代码不会失效旧事实，旧事实继续为“当前”。
- **Evidence**: EV-043-045; EK-10.
- **Missing evidence**: 没有做对抗样本/漏判率测试。
- **Validation path**: 构造 50 对明确矛盾，测量 `contradicted_facts` 召回；对漏判样本检查 `invalid_at`。
- **Impact**: 若成立，KO-02 需要“规则兜底 + 周期性一致性扫描”。
- **Target**: KO-02.

### C-13 · Concurrent same-group episodes can race on previous context

- **Type**: Concurrency
- **Status**: `Hypothesis`
- **Claim**: `_resolve_request_scope` 修了“不同 group 覆写 driver”，但同一 `group_id` 的并发 `add_episode` 仍可能同时读取同一前序窗口，随后各自失效，产生顺序不确定。
- **Evidence**: EV-006/007; EK-49/50.
- **Missing evidence**: 没有并发写入同一 group 的测试。
- **Validation path**: 并发提交两条互相矛盾的同 group episode，重复 100 次，检查最终 `invalid_at` 是否稳定。
- **Impact**: 若成立，KO-08 需要“同键串行/乐观锁”，不能只靠请求级 scope。
- **Target**: KO-08.

### C-14 · `previous_episode_uuids` bypasses reference-time filtering

- **Type**: Context integrity
- **Status**: `Hypothesis`
- **Claim**: 显式 episode UUID 列表直接 `get_by_uuids`，不检查 `valid_at <= reference_time`，可把“未来” episode 注入过去时点的抽取上下文。
- **Evidence**: EV-008/070/074; EK-51.
- **Missing evidence**: 没有构造 reference_time 早于 episode valid_at 的案例。
- **Validation path**: 写入 valid_at=2026-01 与 2026-12 的 episode，用 reference_time=2026-06 并显式传入 2026-12 episode，检查抽取上下文。
- **Impact**: 若成立，KO-08 的“调用方负责上下文”需要加校验/警告。
- **Target**: KO-08.

### C-15 · Historical edge with explicit `invalid_at` is marked `expired_at=now` at ingest

- **Type**: Temporal semantics
- **Status**: `Hypothesis`
- **Claim**: 新抽取边若自带 `invalid_at` 且无 `expired_at`，代码立即把 `expired_at` 设为 `now`。这表示“系统刚入库就记录其已失效”，对回填历史数据可能是预期，也可能是语义混淆。
- **Evidence**: EV-034; EK-09.
- **Missing evidence**: 没有回填历史数据的用例/测试断言。
- **Validation path**: 写入一条 `valid_at=2020-01`、`invalid_at=2021-01` 的边，检查 `created_at`/`expired_at` 与查询可见性。
- **Impact**: 若语义不符，KO-01 需区分“入库即历史”与“入库后被取代”。
- **Target**: KO-01.

### C-16 · Request-scoped driver is applied only to two ingestion APIs

- **Type**: Concurrency / Scope
- **Status**: `Fact`（代码路径）+ `Hypothesis`（跨库后果）
- **Claim**: `_resolve_request_scope` 只在 `add_episode` 与 `add_episode_bulk` 调用。`add_triplet`、`remove_episode`、`get_nodes_and_edges_by_episode` 与默认 `search` 仍使用 `self.driver`/`self.clients`；在 `group_id` 映射到独立 database 的部署中，这些 API 会读/写/删默认库。
- **Evidence**: EV-006; `graphiti.py:2020-2029`, `2048-2171`, `2174-2251`, `1913-1962`; Independent IF-16/17/18/19.
- **Missing evidence**: 没有多 database 部署的端到端测试。
- **Validation path**: 建两个 Neo4j database，group A 写默认库、group B 写克隆库；分别用 `add_triplet`、`remove_episode`、`get_nodes_and_edges_by_episode`、`search` 操作 B，检查目标库。
- **Impact**: KO-08 必须从“请求级状态所有权”收紧为“两个 ingestion API 的请求级状态所有权”；否则会误以为所有 API 都安全。
- **Target**: KO-08, KO-04.

### C-17 · `add_triplet` writes to the default database when `edge.group_id` is non-default

- **Type**: Scope / Data placement
- **Status**: `Hypothesis`（代码路径为 Fact）
- **Claim**: `add_triplet` 的节点解析、既有边查询、duplicate 搜索与最终 `add_nodes_and_edges_bulk` 都用 `self.driver`/`self.clients`；若 `edge.group_id` 不是默认 database，边会被写进默认库，而 `group_id` 字段仍标为 B，形成“分区标签与物理库不一致”。
- **Evidence**: `graphiti.py:2048-2171`; EV-006.
- **Missing evidence**: 没有多库写入实验。
- **Validation path**: 设 `edge.group_id='tenant_b'`，调用 `add_triplet`，分别查询默认库与 tenant_b 库的同 UUID 边。
- **Impact**: 若成立，KO-04/KO-08 需要“写入 API 必须显式解析数据库”的约束。
- **Target**: KO-04, KO-08.

### C-18 · `remove_episode` cannot delete episodes in a cloned database

- **Type**: Lifecycle / Scope
- **Status**: `Hypothesis`（代码路径为 Fact）
- **Claim**: `remove_episode(episode_uuid)` 无 driver/group 参数，直接用 `self.driver` 查询；非默认 group 的 episode 在克隆库中，默认库查不到，删除失败。
- **Evidence**: `graphiti.py:2174-2176`; EV-006.
- **Missing evidence**: 没有跨库删除实验。
- **Validation path**: 在 tenant_b 库写入 episode，调用 `remove_episode`，检查是否 `NodeNotFound` 或删错默认库同名 UUID。
- **Impact**: 若成立，删除 API 必须接受 `group_id` 或 driver，否则多租户删除不可用。
- **Target**: KO-08.

### C-19 · `search` does not auto-resolve `group_ids` to databases

- **Type**: Retrieval / Scope
- **Status**: `Hypothesis`（代码路径为 Fact）
- **Claim**: `Graphiti.search` 只在调用方显式传 `driver` 时切换库；默认 `driver=None` 走 `self.clients`（默认库）。若 `group_ids=['tenant_b']` 且数据在 tenant_b 库，默认检索返回空或不完整。
- **Evidence**: `graphiti.py:1913-1962`, `search.py:98-135`; EV-006.
- **Missing evidence**: 没有多库检索实验。
- **Validation path**: 写入 tenant_b 库后，分别用默认 `search` 与显式 driver 的 `search` 比较结果。
- **Impact**: 若成立，KO-04 需要“检索路由与写入路由必须对称”。
- **Target**: KO-04, KO-08.

### C-20 · `remove_episode` ignores request-scoped LLM/embedder overrides

- **Type**: Configuration / Consistency
- **Status**: `Fact`（代码路径）+ `Hypothesis`（行为差异）
- **Claim**: summary 重建用 `self.llm_client` 与 `self.clients`，而不是请求级 `clients`；若调用方传入自定义 LLM/embedder，删除时的摘要重建仍走默认模型。
- **Evidence**: `graphiti.py:2229`; `GraphitiClients`.
- **Missing evidence**: 没有自定义客户端 + 删除的组合测试。
- **Validation path**: 传入一个记录调用次数的自定义 LLM，删除 episode，检查调用是否命中自定义客户端。
- **Impact**: KO-08 的“请求级依赖束”在删除路径上不成立。
- **Target**: KO-08.

## 2. Cross-Project Candidates（未验证，禁止升格为 Principle）

| ID | Candidate | Evidence in Graphiti | Missing cross-project evidence | Possible validation | Status |
|---|---|---|---|---|---|
| XP-01 | Agent memory systems commonly conflate retention with validity | KO-03; Graphiti stores invalid edges and returns them by default | 需对照 `mem0`、`memmachine`、`letta-code` 的读默认 | 三项目同一 query 的 top-k 中过期事实占比 | `Cross-project Candidate` |
| XP-02 | Async memory ingestion acknowledgements are often less durable than the graph | KO-05; MCP in-memory queue | 需检查其他 MCP memory server 的队列实现 | 对 3 个 MCP server 做提交后重启实验 | `Cross-project Candidate` |
| XP-03 | Tenant/group keys are often mistaken for authorization boundaries | KO-04; no auth in reference server | 需对照多租户 memory 服务的鉴权模型 | 代码审计 + 越权读写测试 | `Cross-project Candidate` |
| XP-04 | Memory systems increasingly separate LLM semantic judgment from deterministic temporal bookkeeping | KO-02; prompt + `_apply_temporal_invalidation` | 需对照其他系统的 contradiction resolver | 对比冲突处理的代码路径 | `Cross-project Candidate` |
| XP-05 | Evidence-anchored derived facts are a common weak point in graph memory | KO-07; triplet without episode | 需对照其他 graph memory 的直接写 API | 检查所有写 API 是否强制 episode | `Cross-project Candidate` |

> 上述五项均不得写入 Generalized Knowledge 的 KO；它们只在未来跨项目考古中作为对照假设。

## 3. Needs Human Review

| ID | Issue | Why human | Suggested owner action |
|---|---|---|---|
| HR-01 | Equal-`valid_at` tie-break semantics | 需要产品语义决定：同一时刻两条互斥事实，哪条为主？ | 定义 business key + 优先级/置信度 |
| HR-02 | Whether invalid facts should be filtered by default | 审计需求 vs 对话准确性冲突 | 决定默认读模式，并提供配置 |
| HR-03 | Whether `add_triplet` should require an episode | 程序化写入的便利 vs 证据完整性 | 强制或标记 `derived_without_source` |
| HR-04 | Whether `remove_episode` should roll back invalidation | 删除语义：删数据还是撤销影响 | 明确生命周期契约 |
| HR-05 | Whether community rebuild should be atomic | 可用性 vs 实现复杂度 | 决定是否引入 staging/versioned communities |
| HR-06 | Whether `group_id` maps to database names in production | 运维安全/命名冲突 | 发布映射与保留名策略 |
| HR-07 | Whether `add_triplet` / `remove_episode` / `get_nodes_and_edges_by_episode` / `search` should all auto-resolve group databases | 当前只有两个 ingestion API 做请求级解析；其余 API 的契约未定义 | 明确“group_id 是否等于 database”，并为所有公开 API 统一路由或显式要求 driver |

## 4. Explicitly Not Candidates

以下内容有证据但不需要“候选”身份，已作为 Fact/Observation 进入 EK：
- `group_id` ASCII 校验（Fact）
- reference server 无认证（Fact）
- Neo4j 真事务 / 其他驱动无事务包装（Fact）
- 默认上下文 10 条（Fact）
- `max_coroutines` 非全局上限（Fact）

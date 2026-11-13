# Candidates — 未确认假说与跨项目候选

> 纪律：这里的每一条都**不是已验证知识**。`Hypothesis` 不得在 03/02 中被当作 Fact 引用；`Cross-project candidate` 不得写成 Principle。
> 每条包含：当前证据 / 缺失证据 / 验证路径 / 若成立的影响。

---

## A. 项目内候选（C-01 ~ C-08）

### C-01 `-inf` sentinel 在 event + euclidean 路径会把“不过滤”变成“全部丢弃”
- **Status**: **Strong candidate defect**（代码路径闭合，未运行验证）
- **Hypothesis**: `SearchMemoriesSpec.score_threshold=None` → `service._search_target_memories` 传 `-float("inf")` → event `_search_scored_event` 在 `similarity_metric=EUCLIDEAN` 且 `reranker=None` 时按 `score > -inf` 判断 → 所有有限距离都“高于阈值” → 全部丢弃，返回空。
- **当前证据**: EV-055（API 默认 None）、EV-056（service 显式 -inf）、EV-057（docstring 警告旧 sentinel）、EV-061（event 阈值方向）、EV-062（测试覆盖 euclidean 正阈值）；`test_memmachine_mock.py:506/584` 仍传 `-inf`。
- **缺失证据**: 没有实际启动 event+euclidean+无 reranker 的服务跑一次搜索；没有确认 `similarity_metric` 默认值是否可能落在 euclidean。
- **验证路径**: 用 `sample_configs/configuration.event.yml` + `similarity_metric=euclidean` + 不配 reranker 启动，POST `/v2/.../search` 不传 `score_threshold`，观察是否空结果；再用 `None` 直连 LTM 对照。
- **若成立的影响**: 默认 API 调用在特定配置下“查不到任何东西”；属于回归级缺陷，不是设计选择。
- **关联**: KO-04、EK-36/37/38。

### C-02 “部分写失败”是否有调用方可见的修复入口
- **Status**: Hypothesis
- **Hypothesis**: 调用方收到 `_raise_add_episode_errors` 后，只能整批重试；系统没有按 episode_id 查询“哪些已写/哪些未写”的接口。
- **当前证据**: EV-016/017；`add_episodes` 返回 `AddEpisodesResult`，但错误路径抛异常，不返回部分结果。
- **缺失证据**: 未检查 client SDK 是否在异常中携带 `episode_ids` 或 `batch_id`；未检查是否有内部 repair CLI。
- **验证路径**: 搜索 `add_episodes` 的所有调用方与 client 重试逻辑；构造部分失败注入。
- **若成立的影响**: 重试可能重复写；不重试则永久缺失。
- **关联**: KO-02。

### C-03 直接暴露 REST 的部署是否有“默认安全”假设
- **Status**: Hypothesis（部署事实，不是代码事实）
- **Hypothesis**: 项目文档假定 REST 部署在可信网关后；若用户按 `docker-compose` 直接映射端口到公网，则任何知道 org/project 的人可读写/删除记忆。
- **当前证据**: EV-012（无 auth）、EV-083（只校验字符）、`docker-compose.yml` 暴露端口（EV-073）。
- **缺失证据**: 未读 deployments/helm 的 ingress/auth 配置；未确认 docs 是否明确要求网关。
- **验证路径**: 检查 `deployments/helm/` 的 ingress/network policy；搜索文档中的 “auth/reverse proxy/security”。
- **若成立的影响**: 默认 compose 是“开发用”，但缺少显著警告。
- **关联**: KO-05。

### C-04 配置 API 更新后，已缓存实例是否使用新配置
- **Status**: Hypothesis
- **Hypothesis**: `ConfigService` 更新 `Configuration` 与资源管理器，但 `MemoryInstanceCache` 中已缓存的 `EpisodicMemory` 仍持有旧 params；直到实例被 evict/重启才生效。
- **当前证据**: EV-075/076（配置更新路径）；EV-049/050（实例缓存按 session 缓存，构造时注入 params）。
- **缺失证据**: 未逐行确认 `ConfigService._apply_*` 是否主动 invalidate cache；未确认 `max_lifetime` 默认值。
- **验证路径**: 读 `config_service.py` 全部更新分支，搜索 `instance_cache`/`evict` 调用；运行期改配置后查询同一 session。
- **若成立的影响**: “配置已保存”与“行为已改变”不一致；运维误判。
- **关联**: KO-03、KO-08。

### C-05 逻辑删除后立即重建同名 session
- **Status**: Hypothesis
- **Hypothesis**: `delete_session` 返回后 status=Deleted 但物理数据未清；若同一 org/project 立刻 `add_episodes`，`create_or_validate_session` 可能因配置不匹配/状态冲突拒绝，或新数据与旧数据在同一 partition 混合。
- **当前证据**: EV-042/043（延迟删除）；EV-041（create_or_validate_session）。
- **缺失证据**: 未读 `create_or_validate_session` 对 Deleted 状态的处理；未验证 event partition 的 incarnation 逻辑。
- **验证路径**: 读 `session_data_manager.py` 状态机；构造删除后立即写入的集成测试。
- **若成立的影响**: 删除语义对调用方不清晰；可能出现“删除后仍可搜到旧记忆”。
- **关联**: KO-08。

### C-06 ~~feature 删除留下 citation 孤儿~~（已被独立验证降级）
- **Status**: **Hypothesis（条件性；作为“强缺陷”已被 CONTRADICTED）**
- **原假设**: PG 后端 `delete_features` 只删 Feature，不删 `citations`，形成孤儿证据。
- **独立验证结论**: `citations.feature_id` 是 `ForeignKey("feature.id", ondelete="CASCADE")`（EV-091）；PG 在删除 Feature 时由数据库级联删除关联行。Neo4j `DETACH DELETE f` 删节点及属性。**两个后端都不会留下孤儿 citation**。
- **保留的条件性假设**: 只有当某个后端不支持 FK 级联（或 ORM 绕过 DB 级联）时才可能成立；当前代码不支持这个前提。
- **缺失证据**: 未运行 PG 删除验证；但 schema 已闭合，除非 DB 未启用 FK。
- **验证路径**: 创建 feature+citation → `delete_features` → 查 `citations`；预期 0 行。
- **若成立的影响**: 若未来换存储或禁用 FK，才可能出现孤儿证据。
- **关联**: KO-07（“过度归属”仍成立，但与删除无关）。

### C-07 `max_features_per_update=50` 是否会导致“看不见的重复”
- **Status**: Hypothesis
- **Hypothesis**: 当某 set/category 的 feature 超过 50 条，LLM 只看到前 50 条；新增 feature 可能与未展示的旧 feature 语义重复，长期膨胀。
- **当前证据**: EV-047（固定 50）。
- **缺失证据**: 未确认 50 条的选取顺序（按创建时间/相关度）；未统计真实 set 的 feature 分布。
- **验证路径**: 读 `get_features` 的排序；用 >50 条 feature 的合成 set 跑 ingestion。
- **若成立的影响**: consolidation 阈值 20 可能永远触发不到“全量去重”；成本上升。
- **关联**: KO-06、KO-07。

### C-08 STM summary 的“陈旧”没有可观测信号
- **Status**: Hypothesis
- **Hypothesis**: 摘要失败保留旧 summary，但 `get_short_term_memory_context` 仍返回旧 summary + 新对话；调用方无法知道 summary 已落后多少轮。
- **当前证据**: EV-036/037。
- **缺失证据**: 未确认 summary 是否带 `last_summarized_at`/`episode_count` 元数据；未检查 metrics 是否有 summary staleness。
- **验证路径**: 读 `ShortTermMemory` 的 summary 结构与 metrics。
- **若成立的影响**: 长对话中“记忆摘要”可能悄悄停留在早期；影响回答质量且难以诊断。
- **关联**: KO-08。

---

## B. 跨项目候选（C-09 ~ C-12）

> **Cross-project candidate = 假设，不是 Principle。** 需要至少 2 个其他项目的独立证据才能升级。

### C-09 “原文保留程度”是记忆系统的分层属性，不是单一卖点
- **Status**: Cross-project candidate（待 corpus 对照）
- **Hypothesis**: 一个记忆系统可以在 episodic 层保留原文（ground-truth-preserving），同时在 semantic/profile 层重度 LLM 抽取；因此“是否 ground-truth-preserving”必须按层回答，不能用产品级口号概括。
- **MemMachine 证据**: 候选卡称 MemMachine 为 “ground-truth-preserving：存原始 episode、少做 LLM 抽取”（source_entry）；但代码显示 semantic ingestion 用 LLM 生成 ADD/DELETE、consolidation 用 LLM 合并（EV-046/048/050），declarative derivative 也加前缀/按句切（EV-027）。
- **对照项目（待验证）**: `memevolve`（corpus 2026-11-13）、`letta`（2026-11-09）、`graphiti`（2026-11-08）、`hy-memory`（2026-11-10）。
- **缺失证据**: 上述项目的原文保留层与抽取层事实尚未在本轮读取；不能引用其结论。
- **验证路径**: 在 corpus 对应 `02-engineering-knowledge` 中检索 “raw/episode/extraction/LLM”；建立三层对照表（episodic / semantic / consolidation）。
- **若成立的影响**: 选型时不能用“保原文”作为整体判断；要问“哪一层保原文、哪一层抽取、citation 是否可回链”。

### C-10 “删除 = 先断索引，后删权威”可能是记忆系统的通用删除顺序
- **Status**: Cross-project candidate
- **Hypothesis**: 多表示记忆系统在删除时普遍先删检索索引、再删权威存储，以避免“搜到但读不到”的坏读；写入时顺序相反。
- **MemMachine 证据**: event 写 segment→vector（EK-21），删 vector→segment（EV-032）；declarative 删节点并行、无显式顺序（EV-028）。
- **对照项目**: `memevolve`、`openviking`、`semanticrun`（均涉及多存储/索引）。
- **缺失证据**: 对照项目的删除顺序未读。
- **验证路径**: 在 corpus 各项目 `04-flow-atlas` 搜 delete/purge/order；记录“先索引/先权威”。
- **若成立的影响**: 可作为设计检查项，但当前只有 MemMachine 单项目证据，不能写成 Principle。

### C-11 “身份是命名空间”在 agent memory 项目中可能是常态而非缺陷
- **Status**: Cross-project candidate
- **Hypothesis**: 多数 agent-memory 服务把 org/project/user 作为命名空间，把认证交给宿主应用/网关；因此“无 auth”要按部署形态判定，不能一律视为漏洞。
- **MemMachine 证据**: EV-012/013/083。
- **对照项目**: `letta`（服务化部署）、`graphiti`（图服务）、`memevolve`。
- **缺失证据**: 对照项目的 auth 实现未读。
- **验证路径**: 在 corpus 各项目 `01-project-layer/architecture` 搜 auth/token/tenant；建立“内建认证 / 外部网关 / 单租户”三分类。
- **若成立的影响**: 安全评估应问“部署边界”，而不是只看代码有没有 `Depends(Security)`。

### C-12 “合并即证据重写”：consolidation 可能普遍损害 provenance 粒度
- **Status**: Cross-project candidate
- **Hypothesis**: 任何用 LLM 做记忆合并的系统，若不保留“新条目 ↔ 旧证据子集”的映射，都会出现 provenance 膨胀；这是结构性问题，不是 MemMachine 独有。
- **MemMachine 证据**: EV-052（全量 citation 挂每个新 feature）。
- **对照项目**: `memevolve`（进化/合并语义）、`hy-memory`（层次记忆）、`letta`（记忆块重构）。
- **缺失证据**: 对照项目的 consolidation/provenance 实现未读。
- **验证路径**: 在 corpus 搜 consolidation/merge/citation/provenance；检查是否有“子集映射”或“证据版本”。
- **若成立的影响**: 记忆系统的“可解释性”会随合并次数下降；需要 provenance 版本化，而不是只存 citation 集合。

---

## C. 候选的处置规则（本轮）

| 候选 | 处置 | 理由 |
|---|---|---|
| C-01 | 保留为强候选缺陷，进 Validation 重点攻击 | 代码路径闭合，但缺运行证据 |
| C-02~C-08 | 保留为项目内 Hypothesis | 均有代码线索，但缺调用方/实现细节 |
| C-09~C-12 | 保留为 Cross-project candidate | 只有 MemMachine 单项目证据 |
| 任何候选 | **不得**升级为 KO/Principle | 未满足跨项目或运行验证 |

## D. 与既有 Corpus 的连接（待下一轮验证）

- `memevolve`（2026-11-13）：可对照“记忆进化/合并”与 C-12。
- `letta`（2026-11-09）：可对照“服务化记忆的认证边界”与 C-11。
- `graphiti`（2026-11-08）：可对照“图索引 + 原文”与 C-09/C-10。
- `openviking`（2026-11-11）：可对照“多存储删除顺序”与 C-10。
- `hy-memory`（2026-11-10）：可对照“层次记忆的抽取层”与 C-09。
- `semanticrun`（2026-11-05）：可对照“运行期语义记忆”与 C-09。

> 以上连接仅为**候选检索路径**；本轮没有读取这些项目的产物内容，故不引用其结论。

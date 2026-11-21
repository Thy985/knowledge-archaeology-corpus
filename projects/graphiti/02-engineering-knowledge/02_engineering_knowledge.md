# 02 · Engineering Knowledge — Evidence + EK Graph

> 本文件先列可追溯 Evidence，再在其上建立 Engineering Knowledge。EK 不是模块说明；每条 EK 至少声明一条边。

## 0. Evidence Index（EV-001–EV-114）

**证据强度**：S3 = 已在 `ed23196` 代码/配置中实现（静态读取）；S2 = 文档中的设计声明。**本 Package 无 S4/S5**：未运行测试、未启动服务。

### A. Facade / Lifecycle（EV-001–EV-028）

| ID | Fact | Evidence | Strength |
|---|---|---|---|
| EV-001 | `Graphiti.__init__` 默认 `store_raw_episode_content=True` | `graphiti_core/graphiti.py:218` | S3 |
| EV-002 | 未传 `graph_driver` 时必须提供 `uri`，默认构造 `Neo4jDriver` | `graphiti.py:209-390` | S3 |
| EV-003 | 未传 `llm_client` / `embedder` / `cross_encoder` 时默认 `OpenAIClient` / `OpenAIEmbedder` / `OpenAIRerankerClient` | `graphiti.py:209-390` | S3 |
| EV-004 | `max_coroutines` 被传入 `GraphitiClients`，作为每次 `semaphore_gather` 的上限 | `graphiti.py:209-390`; `helpers.py:132-142` | S3 |
| EV-005 | 依赖 bundle 为 `GraphitiClients(driver, llm_client, embedder, cross_encoder, tracer, prompt_library, llm_runtime, max_coroutines)` | `graphiti.py:209-390` | S3 |
| EV-006 | `_resolve_request_scope` 返回请求级 driver/clients，避免并发不同 `group_id` 覆写共享 `self.driver` | `graphiti.py:1304-1332`（issue #1676 注释） | S3 |
| EV-007 | `add_episode` 默认取最近 10 条前序 episode（`RELEVANT_SCHEMA_LIMIT=10`） | `graphiti.py:1415-1445`; `search_utils.py:64` | S3 |
| EV-008 | 传入 `previous_episode_uuids` 时完全覆盖自动检索 | `graphiti.py:1430-1445` | S3 |
| EV-009 | 新建 episode 时 `created_at=now`、`valid_at=reference_time` | `graphiti.py:1455-1475` | S3 |
| EV-010 | `add_episode` docstring 明确建议后台队列，并按 episode 顺序 `await`，并发/乱序会破坏前序上下文 | `graphiti.py:1334-1414` | S2 |
| EV-011 | `_extract_and_resolve_edges` 只对 `materialized_nodes` 合并端点，避免丢失 dedupe 的 `uuid_map` | `graphiti.py:916-982` | S3 |
| EV-012 | 提取边后先 `resolve_edge_pointers`，再 `resolve_extracted_edges` | `graphiti.py:916-982` | S3 |
| EV-013 | `extract_attributes_from_nodes` 在边解析后执行 | `graphiti.py:1480-1540` | S3 |
| EV-014 | `_process_episode_data` 先构建 episodic edges | `graphiti.py:1020-1045` | S3 |
| EV-015 | 每个 episode 的 `entity_edges` 记录该 episode 涉及的所有 entity edge UUID | `graphiti.py:1031-1035` | S3 |
| EV-016 | `store_raw_episode_content=False` 时仅把 `ep.content` 置空 | `graphiti.py:1034-1035` | S3 |
| EV-017 | 主图节点/边/episode 通过一次 `add_nodes_and_edges_bulk` 写入 | `graphiti.py:1037-1045` | S3 |
| EV-018 | saga 关系在主编写入之后创建/更新 | `graphiti.py:1047-1095` | S3 |
| EV-019 | `update_communities=True` 时社区更新发生在主写入之后 | `graphiti.py:1540-1588` | S3 |
| EV-020 | `add_episode_bulk` 先单独保存所有 episodes（phase 1） | `graphiti.py:1690-1702` | S3 |
| EV-021 | bulk 的节点/边在第二次 `add_nodes_and_edges_bulk` 写入（phase 2） | `graphiti.py:1776-1785` | S3 |
| EV-022 | bulk 路径同样经过 `extract_edges` / `resolve_extracted_edges`，会持久化 invalidated edges | `graphiti.py:1651-1660` | S2 |
| EV-023 | `add_triplet` 直接构造/解析源目标节点，不经过节点/边抽取 | `graphiti.py:2033-2173` | S3 |
| EV-024 | `add_triplet` 未传 episode 时构造一个 `content=''` 的防御性 `EpisodicNode`，但不写入图 | `graphiti.py:2100-2125` | S3 |
| EV-025 | `remove_episode` 从 `episode.entity_edges` 取边 | `graphiti.py:2174-2195` | S3 |
| EV-026 | 若 edge 没有其他 episode 支持则删除；否则移除当前 episode 并保存 | `graphiti.py:2174-2200` | S3 |
| EV-027 | 只被该 episode `MENTIONS` 的节点会被删除（`episode_count == 1`） | `graphiti.py:2190-2210` | S3 |
| EV-028 | 幸存节点重建 summary 失败时恢复旧 summary 并记录异常，不抛出 | `graphiti.py:2210-2251` | S3 |

### B. Temporal / Edge Maintenance（EV-029–EV-060）

| ID | Fact | Evidence | Strength |
|---|---|---|---|
| EV-029 | `EntityEdge` 含 `created_at`、`valid_at`、`invalid_at`、`expired_at`、`reference_time` | `graphiti_core/edges.py` | S3 |
| EV-030 | `valid_at` 字段说明为“fact became true”，未来计划以 episode reference time 为准 | `edges.py` 字段描述 | S3 |
| EV-031 | `invalid_at` 字段说明为“fact stopped being true”；软截止日期不写 `invalid_at` | `edges.py` 字段描述 | S3 |
| EV-032 | `expired_at` 字段说明为“record was invalidated” | `edges.py` 字段描述 | S3 |
| EV-033 | `EpisodicNode.valid_at` 表示原始文档创建时间 | `nodes.py` 字段描述 | S3 |
| EV-034 | `_apply_temporal_invalidation` 在 resolved edge 自带 `invalid_at` 且无 `expired_at` 时，立即设 `expired_at=now` | `edge_operations.py:722-748` | S3 |
| EV-035 | 若新边无 `expired_at`，按 `valid_at` 排序候选；若候选 `valid_at >` 新边 `valid_at`，把**新边** `invalid_at` 设为候选 `valid_at`、`expired_at=now`，并 break | `edge_operations.py:722-748` | S3 |
| EV-036 | `resolve_edge_contradictions` 对每个候选：若候选 `invalid_at <= 新边 valid_at`，或新边 `invalid_at <= 候选 valid_at`，视为时间不重叠并跳过 | `edge_operations.py:748-785` | S3 |
| EV-037 | 若候选 `valid_at < 新边 valid_at`，候选 `invalid_at=新边 valid_at`、`expired_at=now`，加入 `invalidated_edges` | `edge_operations.py:748-785` | S3 |
| EV-038 | 失效路径不删除旧边，只写 `invalid_at` / `expired_at` | `edge_operations.py:748-785` | S3 |
| EV-039 | 若候选与新边 `valid_at` 完全相同，上述两个跳过条件均成立，双方都不会被置为失效 | `edge_operations.py:748-785`（逻辑推论） | S3 |
| EV-040 | `resolve_extracted_edges` 逐边返回 `(resolved, invalidated, duplicates)` | `edge_operations.py:424-646` | S3 |
| EV-041 | 边去重候选限定为相同端点对（either orientation） | `edge_operations.py:424-646` | S3 |
| EV-042 | 去重 prompt 要求不得因数值/日期/关键限定词差异判重 | `graphiti_core/prompts/dedupe_edges.py` `resolve_edge` | S3 |
| EV-043 | 去重 prompt 的系统消息：只有新事实显式使旧事实为假时才可标矛盾 | `prompts/dedupe_edges.py` system message | S3 |
| EV-044 | prompt 明确禁止仅因 recency、episode 日期、weekly snapshot、section heading 判矛盾 | `prompts/dedupe_edges.py` user message | S3 |
| EV-045 | prompt 要求 contradiction 有显式终止/替换/转移/互斥当前状态语言 | `prompts/dedupe_edges.py` user message | S3 |
| EV-046 | 重复事实返回已存在的 `related_edges[idx]`，并追加当前 episode 到 `episodes`  provenance | `edge_operations.py:930-1030` | S3 |
| EV-047 | 节点解析顺序为 identity → similarity → LLM | `node_operations.py:816-874` | S3 |
| EV-048 | `add_triplet` 用 `EDGE_HYBRID_SEARCH_RRF` 搜同端点候选和 broader existing edges | `graphiti.py:2100-2145` | S3 |
| EV-049 | `add_triplet` 仍调用 `resolve_extracted_edge`，因此会做去重与时间失效 | `graphiti.py:2140-2173` | S3 |
| EV-050 | `add_triplet` 最终通过 `add_nodes_and_edges_bulk` 写 nodes+edges | `graphiti.py:2150-2173` | S3 |
| EV-051 | `remove_episode` 不恢复被该 episode 触发失效的其他边（代码无此路径） | `graphiti.py:2174-2251` | S3 |
| EV-052 | `remove_episode` 删除前不检查 edge 的 `invalid_at` 来源；只按 `episodes` 支持数决定删/留 | `graphiti.py:2174-2200` | S3 |
| EV-053 | `remove_episode` 对幸存节点 summary 按批次重建，失败回滚 summary 文本 | `graphiti.py:2210-2251` | S3 |
| EV-054 | `build_communities` 先 `remove_communities` 清空目标分组，再重建 | `graphiti.py:1867-1895` | S3 |
| EV-055 | 社区节点/边的 save 与 embedding 生成是后续独立步骤，不与主图写入同一事务 | `graphiti.py:1895-1912` | S3 |
| EV-056 | 社区构建并发受 `max_coroutines or MAX_COMMUNITY_BUILD_CONCURRENCY` 限制 | `community_operations.py:256-296` | S3 |
| EV-057 | 社区节点写入前生成 name embedding | `graphiti.py:1895-1912` | S3 |
| EV-058 | `add_episode` 对 saga 新建节点时以最早 episode `valid_at` 为 `created_at` | `graphiti.py:1795-1810` | S3 |
| EV-059 | `NEXT_EPISODE` 链按 `valid_at` 排序构造 | `graphiti.py:1810-1835` | S3 |
| EV-060 | `HAS_EPISODE` 连接 saga 与 episode | `graphiti.py:1047-1095` | S3 |

### C. Search / Context（EV-061–EV-078）

| ID | Fact | Evidence | Strength |
|---|---|---|---|
| EV-061 | `search()` 第一步 `validate_group_ids(group_ids)` | `search/search.py:98-230` | S3 |
| EV-062 | 空 `group_ids` 或 `['']` 归一为 `None` | `search/search.py:98-230` | S3 |
| EV-063 | `search()` 并发发出 edge / node / episode / community 四类查询 | `search/search.py:98-230` | S3 |
| EV-064 | `COMBINED_HYBRID_SEARCH_RRF` 对 edge/node 使用 bm25 + cosine，episode 仅 bm25，community bm25 + cosine，最后 RRF | `search/search_config_recipes.py` | S3 |
| EV-065 | `COMBINED_HYBRID_SEARCH_CROSS_ENCODER` 在混合检索中加入 BFS 扩展 | `search/search_config_recipes.py` | S3 |
| EV-066 | `SearchFilters` 字段默认 `None`；无默认 `valid_at`/`invalid_at`/`expired_at` 过滤 | `search/search_filters.py` | S3 |
| EV-067 | `format_edge_date_range` 输出 `valid_at - invalid_at(或 Present)` | `search/search_helpers.py` | S3 |
| EV-068 | `search_results_to_context_string` 把事实组织为 `{fact, valid_at, invalid_at or 'Present'}` | `search/search_helpers.py` | S3 |
| EV-069 | context 文本声明“事实在 valid_at 与 invalid_at 之间有效，invalid_at=Present 视为有效” | `search/search_helpers.py` | S3 |
| EV-070 | `retrieve_episodes` 只取 `valid_at <= reference_time` 的 episode | `graph_data_operations.py:67-160` | S3 |
| EV-071 | episode 查询按 `valid_at DESC, created_at DESC` 取前 N，然后反转成时间正序 | `graph_data_operations.py:67-160` | S3 |
| EV-072 | `RELEVANT_SCHEMA_LIMIT=10`；`EPISODE_WINDOW_LEN=3` 是另一个默认 | `search_utils.py:64`; `graph_data_operations.py:29` | S3 |
| EV-073 | 前序上下文默认只按 `group_ids=[group_id]` 隔离 | `graphiti.py:1430-1445` | S3 |
| EV-074 | `previous_episode_uuids` 直接 `get_by_uuids`，不再按 `reference_time` 过滤 | `graphiti.py:1430-1445` | S3 |
| EV-075 | `SearchFilters` 支持 `node_labels`、`edge_types`、`valid_at`、`invalid_at` 等过滤条件 | `search/search_filters.py` | S3 |
| EV-076 | `semaphore_gather` 每次调用新建 `asyncio.Semaphore(max_coroutines or SEMAPHORE_LIMIT)` | `helpers.py:132-142` | S3 |
| EV-077 | `DEFAULT_SEMAPHORE_LIMIT=20`，可由 `SEMAPHORE_LIMIT` 环境变量覆盖 | `helpers.py:46-47` | S3 |
| EV-078 | `validate_group_id` 仅允许 ASCII 字母数字、`-`、`_`；空字符串允许 | `helpers.py:145-174` | S3 |

### D. Drivers / Persistence（EV-079–EV-093）

| ID | Fact | Evidence | Strength |
|---|---|---|---|
| EV-079 | `GraphProvider` 枚举 Neo4j/FalkorDB/Kuzu/Neptune | `driver/driver.py:55-70` | S3 |
| EV-080 | 基类 `GraphDriver.transaction()` docstring：有原生事务的驱动 commit/rollback；无原生事务的驱动只是“thin wrapper where queries execute immediately” | `driver/driver.py:136-160` | S3 |
| EV-081 | Neo4j `transaction()` 真正 `begin_transaction`，异常 `rollback`，正常 `commit` | `driver/neo4j_driver.py:149-165` | S3 |
| EV-082 | FalkorDB `execute_write` 直接 `await func(self, ...)`，无事务包装 | `driver/falkordb_driver.py:107-110` | S3 |
| EV-083 | Kuzu `execute_write` 直接 `await func(self, ...)`，无事务包装 | `driver/kuzu_driver.py:281-284` | S3 |
| EV-084 | Neptune `execute_write` 直接 `await func(self, ...)`，无事务包装 | `driver/neptune_driver.py:381-384` | S3 |
| EV-085 | `add_nodes_and_edges_bulk` 在一个 `driver.session()` 上调用 `execute_write(add_nodes_and_edges_bulk_tx)` | `utils/bulk_utils.py:128-150` | S3 |
| EV-086 | 批量事务内先为缺 embedding 的 nodes/edges 生成 embedding | `bulk_utils.py:151-240` | S3 |
| EV-087 | `edge_data` 写入 `uuid/source/target/name/fact/group_id/episodes/created_at/expired_at/valid_at/invalid_at/reference_time/fact_embedding` | `bulk_utils.py:151-240` | S3 |
| EV-088 | 合并 `attributes` 时显式字段优先，防止 attributes 里的 stale string 覆盖 typed datetime | `bulk_utils.py:151-240` | S3 |
| EV-089 | Kuzu 路径按 provider 对 datetime/attributes 做字符串/JSON 适配 | `bulk_utils.py`; `driver/kuzu/` | S3 |
| EV-090 | `add_nodes_and_edges_bulk` 在 `finally` 中关闭 session | `bulk_utils.py:128-150` | S3 |
| EV-091 | `add_episode_bulk` 为 episode 使用独立 phase-1 写入 | `graphiti.py:1690-1702` | S3 |
| EV-092 | phase-2 写入节点/边，若失败，phase-1 episodes 已持久化 | `graphiti.py:1776-1785` | S3 |
| EV-093 | phase-2 写入与 phase-1 不在同一 `execute_write` 调用内 | `graphiti.py:1690-1702`; `1776-1785` | S3 |

### E. Server / MCP / Authority（EV-094–EV-112）

| ID | Fact | Evidence | Strength |
|---|---|---|---|
| EV-094 | REST server 路由含 `/messages`、`/entity-node`、`/search`、`/get-memory`、`/clear`、`/healthcheck` | `server/graph_service/main.py:23-30`; routers | S3 |
| EV-095 | REST server 未挂载认证/授权中间件 | `server/graph_service/main.py` | S3 |
| EV-096 | `SECURITY.md` 明确“servers have no authentication and no authorization” | `SECURITY.md:16` | S2 |
| EV-097 | `SECURITY.md` 明确“no prompt-injection protection”，episode 内容与 tool input 直接送 LLM | `SECURITY.md:17` | S2 |
| EV-098 | `SECURITY.md` 明确“do not isolate callers from each other”，任何调用方可读写任意 `group_id` | `SECURITY.md:18` | S2 |
| EV-099 | `SECURITY.md` 明确同一 `group_id` 内所有 episode 同一信任级，新 episode 可使旧事实失效 | `SECURITY.md:23` | S2 |
| EV-100 | MCP `add_memory` docstring：立即返回，后台处理 | `mcp_server/src/graphiti_mcp_server.py:402-470` | S2 |
| EV-101 | MCP 同 `group_id` 的 episode 顺序处理，避免 race | `mcp_server/.../graphiti_mcp_server.py:402-470`; `queue_service.py:12-48` | S3 |
| EV-102 | `QueueService` 使用 `dict[str, asyncio.Queue]` 保存每个 group 的队列 | `mcp_server/src/services/queue_service.py:12-48` | S3 |
| EV-103 | 队列 worker 捕获异常后仅 `logger.error`，`finally` 中 `task_done()`，无重试 | `queue_service.py:49-78` | S3 |
| EV-104 | 队列无持久化；进程退出后 `asyncio.Queue` 内容消失 | `queue_service.py:12-78`（结构推论） | S3 |
| EV-105 | MCP 暴露 `delete_episode`、`clear_graph` 等写操作 | `mcp_server/src/graphiti_mcp_server.py` | S3 |
| EV-106 | `docker-compose.yml` 提供 Neo4j/FalkorDB 部署，`docker-compose.test.yml` 提供测试栈 | 仓库根目录 | S3 |
| EV-107 | `server/.env.example` 配置 OpenAI key/base/model 与 Neo4j/FalkorDB 连接 | `server/.env.example` | S3 |
| EV-108 | `AGENTS.md`、`CLAUDE.md`、`CONTRIBUTING.md`、`SECURITY.md`、`Zep-CLA.md` 存在 | 仓库根目录 | S3 |
| EV-109 | `pyproject.toml` 依赖 `pydantic>=2.11.5`、`neo4j>=5.26.0`、`openai>=1.91.0`、`tenacity>=9.0.0`、`numpy`、`python-dotenv`、`posthog`、`certifi` | `pyproject.toml` | S3 |
| EV-110 | `pyproject.toml` 版本 `0.31.2`，`requires-python >=3.10,<4`，`license Apache-2.0` | `pyproject.toml` | S3 |
| EV-111 | 81 个 `test_*.py`，910 个静态 `def test_` | 本轮静态计数 | S3 |
| EV-112 | 时间语义测试文件存在：`test_temporal_edge_utils.py`、`test_edge_operations.py`、`test_edge_dedup_scenarios.py` | `tests/utils/maintenance/` | S3 |
| EV-113 | `search_results_to_context_string` 的 fact JSON 只含 `fact`/`valid_at`/`invalid_at`，不含 edge UUID 或 `episodes` 溯源列表 | `search/search_helpers.py` | S3 |
| EV-114 | context 的 EPISODES 段只含 `source_description`/`content`，不含 episode UUID | `search/search_helpers.py` | S3 |

> Canonical count: **114 Evidence facts** (EV-001 … EV-114).

## 1. Engineering Knowledge（EK-01–EK-56）

**Layer**：L1 = 工程事实型知识；L2 = 工程经验/机制型知识。**Epistemic**：Fact / Observation。**Value**：A/B/C/D。

### 1.1 Temporal Model（EK-01–EK-12）

| ID | Layer | Category | Statement | Evidence | Links | Epistemic | Value |
|---|---|---|---|---|---|---|---|
| EK-01 | L1 | RUNTIME/MEMORY | Graphiti 的事实边同时维护四条时间轴：`created_at`、`valid_at`、`invalid_at`、`expired_at` | EV-029,030,031,032,033 | mechanism EK-02/03/04/05; subsystem EK-55 | Fact | A |
| EK-02 | L2 | MEMORY | `valid_at` 以 `reference_time` 为锚，未来计划使用“计划被创建的时间”而不是未来发生日期 | EV-030,033 | mechanism EK-01; causal EK-02→EK-09 | Observation | A |
| EK-03 | L2 | MEMORY | `created_at` 是知识时间：系统何时知道这条事实 | EV-009,029 | mechanism EK-01; contrast EK-03↔EK-05 | Fact | A |
| EK-04 | L2 | MEMORY | `invalid_at` 是现实时间：事实何时不再为真；软截止日期不写此字段 | EV-031 | causal EK-04→EK-09; mechanism EK-01 | Fact | A |
| EK-05 | L2 | RUNTIME | `expired_at` 是系统记录时间：这条记录何时被系统判定为失效 | EV-032,034 | contrast EK-05↔EK-03; causal EK-05→EK-09 | Fact | A |
| EK-06 | L2 | MEMORY | 若新抽取事实比已有候选更旧，新事实会被直接结束在候选的 `valid_at`，不会反向覆盖更新的既有事实 | EV-035 | causal EK-09→EK-06; constraint EK-06→EK-07 | Observation | A |
| EK-07 | L2 | MEMORY | 失效是软失效：写 `invalid_at`/`expired_at`，不删除旧边 | EV-038,037 | causal EK-07→EK-08; contrast EK-07↔EK-32 | Fact | A |
| EK-08 | L2 | MEMORY | 当新旧事实的 `valid_at` 完全相同，时间窗口判定不会使任何一方失效，双方可同时保持“当前” | EV-039 | constraint EK-09→EK-08; contrast EK-08↔EK-06 | Observation (logic-derived) | B |
| EK-09 | L2 | MEMORY | 旧事实被新事实取代时，旧 `invalid_at=新 valid_at`，旧 `expired_at=now` | EV-036,037 | causal EK-09→EK-07; dependency EK-09→EK-10 | Fact | A |
| EK-10 | L2 | AGENT/MEMORY | 只有 LLM 标记为 `contradicted_facts` 的候选才进入时间失效算术；未标记的矛盾不会自动失效 | EV-040,043,045 | causal EK-10→EK-09; dependency EK-10→EK-11 | Observation | A |
| EK-11 | L2 | AGENT | 去重 prompt 禁止仅凭 recency、episode 日期、weekly snapshot、section heading 判矛盾 | EV-044 | constraint EK-11→EK-10 | Fact | A |
| EK-12 | L2 | AGENT | 矛盾需要显式终止/替换/转移/互斥当前状态语言；仅“同一实体有活动/进展”不算矛盾 | EV-043,045 | constraint EK-12→EK-10 | Fact | B |

### 1.2 Resolution / Dedupe（EK-13–EK-16）

| ID | Layer | Category | Statement | Evidence | Links | Epistemic | Value |
|---|---|---|---|---|---|---|---|
| EK-13 | L1 | ENGINEERING | 节点解析是 identity → similarity → LLM 的级联，不是直接让 LLM 合并全部节点 | EV-047 | subsystem EK-13↔EK-14; causal EK-13→EK-15 | Fact | A |
| EK-14 | L2 | ENGINEERING | 边去重候选只看相同端点对，避免全图比较；LLM 只在候选集内判断重复/矛盾 | EV-041,042,046 | mechanism EK-14↔EK-13; causal EK-14→EK-15 | Fact | A |
| EK-15 | L2 | EVIDENCE | 识别为重复的事实不会新建边，而是复用已有 edge uuid，并把当前 episode 追加进 `episodes` 溯源列表 | EV-046 | causal EK-15→EK-30; dependency EK-15→EK-16 | Fact | A |
| EK-16 | L2 | ENGINEERING | `add_triplet` 绕过抽取但仍走 dedupe + temporal invalidation；它可被用作程序化写边入口 | EV-023,048,049,050 | contrast EK-16↔EK-17; subsystem EK-16↔EK-14 | Fact | B |

### 1.3 Persistence / Transaction（EK-17–EK-23）

| ID | Layer | Category | Statement | Evidence | Links | Epistemic | Value |
|---|---|---|---|---|---|---|---|
| EK-17 | L1 | RUNTIME | 单 episode 的节点/边/episode 主写入发生在同一个 `execute_write` 调用中 | EV-017,085 | causal EK-17→EK-18; subsystem EK-17↔EK-19 | Fact | A |
| EK-18 | L2 | RUNTIME | “原子写”仅在 Neo4j 成立；FalkorDB/Kuzu/Neptune 的 `execute_write` 只是立即执行函数，无事务回滚 | EV-080,081,082,083,084 | dependency EK-17→EK-18; contrast EK-18↔EK-46 | Fact | A |
| EK-19 | L1 | RUNTIME/WORKFLOW | 批量导入分两阶段：先保存 episodes，再抽取并写入节点/边 | EV-091,092,093 | causal EK-19→EK-20; subsystem EK-19↔EK-17 | Fact | A |
| EK-20 | L2 | FAILURE | 若 phase-2 失败，已写入的 episodes 仍可被后续检索当作前序上下文，形成“有 episode 无实体/边”的孤儿状态 | EV-092,070 | causal EK-20→EK-30; constraint EK-20→EK-19 | Observation | A |
| EK-21 | L2 | FAILURE | saga 关系在主编写入之后创建；主编成功、saga 失败会留下无 saga 连接的事实 | EV-018,059,060 | causal EK-21→EK-17; dependency EK-21→EK-18 | Fact | B |
| EK-22 | L2 | FAILURE | community 更新在主写入之后；且 `build_communities` 先清空目标分组再重建，重建失败会留下“社区为空”的中间态 | EV-019,054,055 | causal EK-22→EK-19; contrast EK-22↔EK-07 | Fact | B |
| EK-23 | L2 | RUNTIME | 嵌入生成发生在写入事务内部/之前，外部 LLM/embedding 失败会阻断主写入 | EV-086,017 | dependency EK-23→EK-17 | Observation | B |

### 1.4 Async / MCP（EK-24–EK-28）

| ID | Layer | Category | Statement | Evidence | Links | Epistemic | Value |
|---|---|---|---|---|---|---|---|
| EK-24 | L1 | AGENT/RUNTIME | MCP `add_memory` 立即返回，实际抽取在后台任务中进行 | EV-100 | causal EK-24→EK-25 | Fact | A |
| EK-25 | L2 | RUNTIME | 后台处理按 `group_id` 串行；同一分组不会并发处理 episode | EV-101,102 | causal EK-25→EK-26; constraint EK-25→EK-24 | Fact | A |
| EK-26 | L2 | RUNTIME/FAILURE | 队列是进程内 `asyncio.Queue`，无持久化、无重试、无死信；失败只记日志并 `task_done` | EV-102,103 | causal EK-26→EK-27; contrast EK-26↔EK-17 | Fact | A |
| EK-27 | L2 | FAILURE | 进程退出后未处理的 episode 任务消失，但调用方已收到“已接收”的返回 | EV-100,104 | causal EK-27→EK-24; dependency EK-27→EK-26 | Observation | A |
| EK-28 | L2 | MEMORY | 同一 `group_id` 内所有 episode 共享信任级；新 episode 可以改变/失效旧事实 | EV-099 | dependency EK-28→EK-45 | Fact (doc) | A |

### 1.5 Evidence / Provenance（EK-29–EK-35）

| ID | Layer | Category | Statement | Evidence | Links | Epistemic | Value |
|---|---|---|---|---|---|---|---|
| EK-29 | L1 | EVIDENCE | 默认保存原始 episode 内容；`store_raw_episode_content=False` 时只清空 `content` | EV-001,016 | mechanism EK-29↔EK-30 | Fact | A |
| EK-30 | L2 | EVIDENCE | 关闭原始内容后，episode 的 `name`/`source_description`/`valid_at`/`entity_edges` 仍在，但原始文本不可回读 | EV-016,015 | causal EK-30→EK-31; constraint EK-30→EK-29 | Observation | A |
| EK-31 | L2 | EVIDENCE | `episode.entity_edges` 是 episode→entity-edge 的显式溯源清单；`remove_episode` 依赖它决定删/留边 | EV-015,025,026 | dependency EK-31→EK-15; causal EK-31→EK-32 | Fact | A |
| EK-32 | L2 | EVIDENCE | `MENTIONS` 边把 episode 连接到 entity；节点是否删除按 `MENTIONS` 计数判定 | EV-027 | subsystem EK-32↔EK-31; causal EK-32→EK-33 | Fact | B |
| EK-33 | L2 | FAILURE | `remove_episode` 只按 `episodes` 支持数删/留边，不恢复被该 episode 触发的 `invalid_at`/`expired_at` | EV-051,052 | causal EK-33→EK-31; constraint EK-33→EK-07 | Observation | A |
| EK-34 | L2 | FAILURE | 节点 summary 重建失败时恢复旧 summary 并吞掉异常，系统保留可能已过时的摘要 | EV-028,053 | causal EK-34→EK-33; dependency EK-34→EK-35 | Fact | B |
| EK-35 | L2 | EVIDENCE | `add_triplet` 无真实 episode 时构造一个未持久化的 synthetic `EpisodicNode`；`append_episode_provenance` 把它的 UUID 追加到 `edge.episodes`，因此边带的是**悬空引用**，不是空列表 | EV-024,050 | causal EK-35→EK-31; contrast EK-35↔EK-17 | Observation | A |

### 1.6 Search / Truth in Context（EK-36–EK-41）

| ID | Layer | Category | Statement | Evidence | Links | Epistemic | Value |
|---|---|---|---|---|---|---|---|
| EK-36 | L1 | SEARCH | 检索 API 支持 `valid_at`/`invalid_at` 等时间过滤，但默认过滤器为空 | EV-066,075 | causal EK-36→EK-37 | Fact | A |
| EK-37 | L2 | MEMORY/TRUTH | 默认检索不自动排除 `invalid_at`/`expired_at` 的边；旧事实可被返回 | EV-066,063 | causal EK-37→EK-38; contrast EK-37↔EK-07 | Observation | A |
| EK-38 | L2 | TRUTH | 返回给 LLM 的上下文以 `valid_at - invalid_at/Present` 表达区间，并附自然语言“仅在此区间有效” | EV-067,068,069 | dependency EK-38→EK-37 | Fact | A |
| EK-39 | L2 | SEARCH/SCOPE | `group_id` 过滤贯穿 edge/node/episode/community 四类检索 | EV-061,073 | subsystem EK-39↔EK-40 | Fact | A |
| EK-40 | L2 | SEARCH | 空 `group_ids`/`['']` 被归一为“无过滤”，即退化为跨分组检索 | EV-062 | causal EK-40→EK-39; constraint EK-40→EK-44 | Fact | A |
| EK-41 | L2 | SEARCH | `COMBINED_HYBRID_SEARCH_RRF` 对不同 scope 使用不同信号组合；episode 只用 bm25，不做向量检索 | EV-064 | subsystem EK-41↔EK-42 | Fact | B |
| EK-41b | L2 | SEARCH | `COMBINED_HYBRID_SEARCH_CROSS_ENCODER` 额外加入 BFS 邻居扩展，扩大候选集 | EV-065 | mechanism EK-41b↔EK-41 | Fact | B |

### 1.7 Scope / Authority / Security（EK-42–EK-48）

| ID | Layer | Category | Statement | Evidence | Links | Epistemic | Value |
|---|---|---|---|---|---|---|---|
| EK-42 | L1 | PERMISSION | `group_id` 是图分区键，读写都按它过滤 | EV-061,073 | dependency EK-42→EK-39; contrast EK-42↔EK-46 | Fact | A |
| EK-43 | L2 | PERMISSION | 当 `group_id` 不等于默认数据库时，**只有 `add_episode` 与 `add_episode_bulk`** 会 clone driver 并把 `group_id` 当作 database 名；其余公开 API 仍走 `self.driver` | EV-006,078; IF-16–19 | causal EK-43→EK-42; constraint EK-43→EK-44 | Observation | A |
| EK-44 | L2 | PERMISSION | `group_id` 仅做 ASCII `[A-Za-z0-9_-]+` 校验；它阻止注入，不做身份/授权校验 | EV-078 | constraint EK-44→EK-43; contrast EK-44↔EK-45 | Fact | A |
| EK-45 | L2 | AUTHORITY | reference server 无认证/授权；任何调用方可读写任意 `group_id` | EV-095,096,098 | causal EK-45→EK-46 | Fact | A |
| EK-46 | L2 | AUTHORITY | 同一 `group_id` 内所有 episode 同信任级；一条新 episode 可影响旧事实 | EV-099 | causal EK-46→EK-45; dependency EK-46→EK-28 | Fact (doc) | A |
| EK-47 | L2 | SECURITY | reference server 无 prompt-injection 防护，episode 内容与 tool input 直接进入 LLM | EV-097 | causal EK-47→EK-46 | Fact (doc) | A |
| EK-48 | L2 | AUTHORITY | 安全边界明确被推给部署方：认证、授权、网络控制由外部提供 | EV-096,098 | constraint EK-48→EK-45 | Fact (doc) | A |

### 1.8 Concurrency / Context Window / Test（EK-49–EK-56）

| ID | Layer | Category | Statement | Evidence | Links | Epistemic | Value |
|---|---|---|---|---|---|---|---|
| EK-49 | L2 | RUNTIME | `max_coroutines` 是每个 `semaphore_gather` 调用点的上限，不是全局并发上限；多个调用点可叠加并发 | EV-004,076,077 | constraint EK-49→EK-50 | Fact | B |
| EK-50 | L2 | CONTEXT | 单 episode 默认前序窗口是 10 条，按 `reference_time` 过滤后取最近，再转正序 | EV-007,070,071,072 | causal EK-50→EK-51 | Fact | A |
| EK-51 | L2 | CONTEXT | `previous_episode_uuids` 可覆盖自动窗口，且绕过 `reference_time` 过滤；调用方对上下文负责 | EV-008,074 | contrast EK-51↔EK-50 | Fact | A |
| EK-52 | L1 | TESTING | 仓库存在专门的时间语义/边去重测试文件 | EV-112 | dependency EK-52→EK-53 | Fact | B |
| EK-53 | L2 | TESTING | 81 个测试文件、910 个静态测试函数；本轮未执行，不能把测试存在当作行为已验证 | EV-111 | mechanism EK-53↔EK-52 | Observation | A |
| EK-54 | L1 | GOVERNANCE | `SECURITY.md` 的“无认证/无隔离”是设计声明，且有代码缺失佐证 | EV-095,096,097,098 | constraint EK-54→EK-45 | Fact | A |
| EK-55 | L2 | MEMORY | `GraphitiClients` 把 driver/LLM/embedder/reranker/tracer/prompt library 绑定为一个依赖束，请求级 scope 复制的是这个束 | EV-005,006 | subsystem EK-55↔EK-01; causal EK-55→EK-43 | Fact | C |
| EK-56 | L2 | TESTING | 集成测试与 mock 测试并存，说明项目同时验证单点行为与跨组件装配；但本轮均未执行 | EV-112; `tests/test_graphiti_mock.py`; `tests/test_edge_int.py` | mechanism EK-56↔EK-53 | Observation | B |
| EK-57 | L2 | EVIDENCE/TRUTH | LLM 上下文只暴露事实文本与有效期，不暴露 edge UUID/episode UUID，因此模型侧无法回溯证据；溯源链止于 API 层 | EV-067,068,113,114 | causal EK-57→EK-38; dependency EK-57→EK-31; contrast EK-57↔EK-15 | Fact | A |

## 2. EK Graph

### 2.1 Edge summary

| Edge type | Count | Example |
|---|---|---|
| mechanism | 9 | EK-01↔02/03/04/05; EK-14↔13 |
| subsystem | 9 | EK-17↔19; EK-39↔40 |
| causal | 35 | EK-09→07; EK-45→46 |
| dependency | 15 | EK-17→18; EK-10→11 |
| constraint | 14 | EK-11→10; EK-44→43 |
| contrast | 14 | EK-03↔05; EK-42↔46 |
| **Total** | **96** | — |

**Graph-level metrics**
- EK total: **58**（EK-01..EK-57，含 EK-41b）
- EK with ≥1 declared link: **58 / 58 (100%)**
- Isolated EK: **0**
- Average declared links per EK: **96 / 58 ≈ 1.66**
- Aggregation coverage: all EK appear in at least one KO cluster.

### 2.2 Key paths

**Path A — 事实失效**
`EK-01 (四时间轴) → EK-02/03/04/05 (语义) → EK-10 (LLM 标记矛盾) → EK-09 (时间算术) → EK-07 (软失效) → EK-37 (默认检索仍返回) → EK-38 (上下文标注) → EK-33 (删除不恢复)`

**Path B — 写路径完整性**
`EK-17 (主写入同一 execute_write) → EK-18 (仅 Neo4j 原子) → EK-19 (bulk 两阶段) → EK-20 (孤儿 episode) → EK-21/22 (saga/community 后置)`

**Path C — 作用域/权威**
`EK-42 (分区) → EK-43 (driver 路由) → EK-44 (ASCII 校验) → EK-45 (无认证) → EK-46 (同组同信任) → EK-47 (无注入防护) → EK-48 (边界外推)`

**Path D — 接收与持久性**
`EK-24 (立即返回) → EK-25 (按组串行) → EK-26 (内存队列/无重试) → EK-27 (重启丢失)`

**Path E — 证据链**
`EK-15 (重复复用 edge + provenance) → EK-31 (entity_edges) → EK-32 (MENTIONS) → EK-33 (删除不恢复失效) → EK-35 (悬空 synthetic UUID 的 triplet) → EK-57 (上下文无 UUID)`

## 3. EK → Category Coverage

| Category | EK IDs | Count |
|---|---|---|
| MEMORY | 01-10, 28, 37, 55 | 13 |
| RUNTIME | 01,05,17,19,24,25,26,49,55 | 9 |
| ENGINEERING | 13,14,16,41 | 4 |
| EVIDENCE | 15,29,30,31,32,33,35,57 | 8 |
| FAILURE | 20,21,22,26,27,33,34 | 7 |
| SEARCH | 36,39,40,41,41b | 5 |
| PERMISSION/AUTHORITY/SECURITY | 42-48,54 | 8 |
| CONTEXT/TESTING/GOVERNANCE | 50-53,56 | 5 |

> Category counts overlap: EK-01 is both MEMORY and RUNTIME, so the category sum (59) is one greater than the unique EK total (58).

## 4. EK Quality Notes

- **Not module summaries**: each EK states a behavior/decision with evidence; links explain why it matters to another EK.
- **No forced EK**: 58 is within the 40–60 band for a 77k-line codebase; none was created merely to fill quota.
- **D-Level none**: all EK are at least B or C; C only for the dependency-bundle structure (EK-55), which is local but useful for request-scope reasoning.
- **What is deliberately not EK**: raw file lists, directory names, README marketing claims, and single test names without behavioral content.

## 5. Reconciliation Corrections (Independent Validation)

These corrections are merged into the statements above; they are retained here so the audit trail is explicit.

| ID | Correction | Source | Affected EK |
|---|---|---|---|
| RC-01 | `add_triplet` 的 `episodes` 不是空列表，而是包含一个未持久化 synthetic episode 的 UUID（悬空引用） | IF-13/14; `edge_operations.py:884-890`; `graphiti.py:2140-2156` | EK-35 |
| RC-02 | “请求级 driver”只覆盖 `add_episode` 与 `add_episode_bulk`；`add_triplet`/`remove_episode`/`get_nodes_and_edges_by_episode`/默认 `search` 仍用 `self.driver` | IF-16–19 | EK-43, EK-55 |
| RC-03 | `remove_episode` 使用 `self.llm_client` 重建 summary，不使用请求级 `clients.llm_client` | IF-33 | EK-34, EK-55 |
| RC-04 | `add_triplet` 在非默认 `group_id` 下仍写入默认数据库，造成分区标签与物理库不一致 | IF-34 | EK-43 |
| RC-05 | `build_communities` 是“先删后建”，且 driver 可外部传入；不是原子切换 | IF-20/24 | EK-22 |

**Updated statement for EK-55**: `GraphitiClients` 是一个依赖束，但“请求级复制”只发生在两个 ingestion API；它不是所有公开 API 的通用保证。

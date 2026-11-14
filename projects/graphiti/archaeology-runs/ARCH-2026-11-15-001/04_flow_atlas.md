# 04 · Flow Atlas — Graphiti

> 七类流均从 `ed23196` 的真实代码导出。每条 edge 都给出 symbol/file、条件、状态转移与 EV 证据；无运行时观测，`runtime_verified: false`。

## 0. Summary

| Flow | Edges | Runtime verified |
|---|---|---|
| Control | 19 | 否 |
| State | 16 | 否 |
| Data | 12 | 否 |
| Evidence | 10 | 否 |
| Authority | 13 | 否 |
| Memory | 10 | 否 |
| Policy | 9 | 否 |
| **Total** | **89** | **0** |

## 1. Control Flow

| ID | From → To | Symbol / File | Condition | State transition | EV |
|---|---|---|---|---|---|
| F-C-01 | `Graphiti.add_episode` entry → request scope | `_resolve_request_scope`; `graphiti.py:1423` | always | shared clients → request-scoped bundle | EV-006 |
| F-C-02 | scope → previous-episode branch | `add_episode`; `graphiti.py:1426-1445` | `previous_episode_uuids is None` | none | EV-007/008 |
| F-C-02b | scope → explicit previous episodes | `EpisodicNode.get_by_uuids`; `graphiti.py:1445` | `previous_episode_uuids is not None` | context replaced | EV-008 |
| F-C-03 | previous context → episode object | `EpisodicNode(...)`; `graphiti.py:1455-1475` | always | new episode `created_at=now`, `valid_at=reference_time` | EV-009 |
| F-C-04 | episode → node extraction | `extract_nodes`; `graphiti.py:1475-1500` | always | raw content → extracted node list | EV-011 |
| F-C-05 | extracted nodes → node resolution | `resolve_extracted_nodes`; `graphiti.py:1500-1515` | always | duplicate entities collapsed | EV-047 |
| F-C-06 | resolved nodes → edge pipeline | `_extract_and_resolve_edges`; `graphiti.py:1515-1525` | always | nodes → edge candidates | EV-011/012 |
| F-C-07 | edge pipeline → merge endpoints | `_merge_materialized_endpoint_nodes`; `graphiti.py:916-982` | only materialized nodes | endpoint UUIDs aligned | EV-011 |
| F-C-08 | merged endpoints → pointer resolution | `resolve_edge_pointers`; `graphiti.py:916-982` | always | edge source/target UUID set | EV-012 |
| F-C-09 | pointers → edge resolution | `resolve_extracted_edges`; `graphiti.py:916-982` | always | candidates → resolved/duplicate/invalidated | EV-040 |
| F-C-10 | resolved edges → node attributes | `extract_attributes_from_nodes`; `graphiti.py:1480-1540` | always | node/edge attributes filled | EV-013 |
| F-C-11 | attributes → episode data | `_process_episode_data`; `graphiti.py:984-1095` | always | episode edges + entity_edges constructed | EV-014/015 |
| F-C-12 | episode data → bulk write | `add_nodes_and_edges_bulk`; `graphiti.py:1037` | always | in-memory graph objects → persisted rows | EV-017 |
| F-C-13 | bulk write success → saga | `_get_or_create_saga`; `graphiti.py:1047-1095` | `saga is not None` | saga chain updated | EV-018/058/059 |
| F-C-14 | saga success → communities | `update_community`; `graphiti.py:1540-1588` | `update_communities is True` | community summaries updated | EV-019 |
| F-C-15 | any exception → error span → raise | `add_episode` `except Exception`; `graphiti.py:1575-1588` | exception | operation aborted, prior writes stay | EV-017/019 |
| F-C-16 | bulk phase 1 → phase 2 | `add_episode_bulk`; `graphiti.py:1690-1785` | always | episodes persisted before entities/edges | EV-020/021/091-093 |
| F-C-17 | add_triplet → dedupe → write | `add_triplet`; `graphiti.py:2033-2173` | caller invokes | direct edge candidate → resolved → persisted | EV-023/048-050 |
| F-C-18 | build_communities → remove → build → save | `build_communities`; `graphiti.py:1867-1912` | always | old communities deleted; new ones built | EV-054/055/056/057 |
| F-C-19 | remove_episode → classify → delete/update → summaries | `remove_episode`; `graphiti.py:2174-2251` | caller invokes | episode deleted; edges/nodes pruned | EV-025-028 |

**Control flow notes**
- `add_episode` has no retry loop; any exception exits via `raise` (F-C-15).
- `add_episode_bulk` phase separation is the only deliberate multi-phase control path (F-C-16).
- `build_communities` is destructive first, constructive second (F-C-18).

## 2. State Flow

| ID | State A → State B | Symbol / File | Trigger / Condition | Transition detail | EV |
|---|---|---|---|---|---|
| F-S-01 | no group → default group | `_resolve_request_scope`; `graphiti.py:1304-1332` | `group_id is None` | default provider group; shared driver | EV-006 |
| F-S-02 | default DB → per-group DB | `driver.clone(database=group_id)`; same | `group_id != driver._database` | clone driver; copy clients | EV-006/043 |
| F-S-03 | no episode → episode row | `EpisodicNode(...)`; `graphiti.py:1455-1475` | new UUID or existing | `created_at=now`; `valid_at=reference_time` | EV-009 |
| F-S-04 | raw content present → blank | `_process_episode_data`; `graphiti.py:1034` | `store_raw_episode_content=False` | `content=''` only | EV-016 |
| F-S-05 | extracted node → identity-resolved | `resolve_extracted_nodes`; `node_operations.py:816-874` | exact identity match | UUID reused | EV-047 |
| F-S-06 | identity miss → similarity-resolved | same | similarity threshold hit | entity merged | EV-047 |
| F-S-07 | similarity miss → LLM-resolved | same | LLM decides merge/new | node created or merged | EV-047 |
| F-S-08 | extracted edge → duplicate edge | `resolve_extracted_edge`; `edge_operations.py:834-930` | LLM `duplicate_facts` hit | existing UUID kept; provenance appended | EV-046 |
| F-S-09 | old edge current → invalid | `_apply_temporal_invalidation`; `edge_operations.py:722-748` | old `valid_at < new valid_at` | old `invalid_at=new.valid_at`; `expired_at=now` | EV-037 |
| F-S-10 | new backdated edge → historical | same | candidate `valid_at > new valid_at` | new `invalid_at=candidate.valid_at`; `expired_at=now` | EV-035 |
| F-S-11 | contradictory pair → both current | `resolve_edge_contradictions`; `edge_operations.py:748-785` | equal `valid_at` | no invalidation edge fires | EV-039 |
| F-S-12 | episode present → unsupported edge deleted | `remove_episode`; `graphiti.py:2174-2200` | `edge.episodes` becomes empty | edge row deleted | EV-026 |
| F-S-13 | episode present → supported edge survives | same | other episode UUIDs remain | edge re-saved without current episode | EV-026 |
| F-S-14 | source episode deleted → invalidation retained | `remove_episode`; `graphiti.py:2174-2251` | any | `invalid_at`/`expired_at` unchanged | EV-051/052 |
| F-S-15 | communities present → cleared → rebuilt | `build_communities`; `graphiti.py:1867-1912` | always | old community nodes/edges removed | EV-054/055 |
| F-S-16 | MCP task queued → running → done/failed | `QueueService`; `queue_service.py:49-78` | worker picks task | `task_done()` in finally; failure logged | EV-101-103 |

## 3. Data Flow

| ID | Input → Output | Symbol / File | Transformation | EV |
|---|---|---|---|---|
| F-D-01 | raw episode (`text/json/message`) → extraction prompt | `extract_nodes` / `extract_edges`; prompts | content + context serialized to LLM | EV-011 |
| F-D-02 | prompt → extracted nodes | `extract_nodes`; `graphiti_core/prompts/extract_nodes.py` | LLM returns entity candidates | EV-011 |
| F-D-03 | prompt → extracted edges | `extract_edges`; prompts | LLM returns fact candidates | EV-011 |
| F-D-04 | entity node → name embedding | `generate_name_embedding`; `nodes.py` | text → vector | EV-057/086 |
| F-D-05 | entity edge → fact embedding | `generate_embedding`; `edges.py` | fact text → vector | EV-086 |
| F-D-06 | resolved edge → `edge_data` row | `add_nodes_and_edges_bulk_tx`; `bulk_utils.py:151-240` | object → typed row params | EV-087 |
| F-D-07 | episode + entity edges → `entity_edges` UUID list | `_process_episode_data`; `graphiti.py:1031-1035` | edge objects → UUID references | EV-015 |
| F-D-08 | `edge_data` + nodes + episodes → graph DB | `execute_write`; `bulk_utils.py:128-150` | rows persisted | EV-085 |
| F-D-09 | query text → query embedding | `search()`; `search.py` | text → vector | EV-063 |
| F-D-10 | search hits → context JSON | `search_results_to_context_string`; `search_helpers.py` | objects → facts/entities/episodes/communities | EV-067/068/113/114 |
| F-D-11 | context JSON → LLM prompt | caller / server | prompt assembly | EV-069 |
| F-D-12 | community summaries → community node embedding | `build_communities`; `graphiti.py:1895-1912` | summary → vector | EV-057 |

**Data flow notes**
- `attributes` merge is field-protected: explicit typed fields are not overwritten by stale attribute strings (EV-088).
- Kuzu serializes datetime/attributes differently (EV-089).
- The context string deliberately omits edge and episode UUIDs (EV-113/114), so data flow to LLM is **content + time**, not **provenance IDs**.

## 4. Evidence Flow

| ID | Evidence Link | Symbol / File | Meaning | EV |
|---|---|---|---|---|
| F-E-01 | EpisodicNode → `MENTIONS` → EntityNode | `build_episodic_edges`; `graphiti.py:1020-1045` | episode mentions entity | EV-027/032 |
| F-E-02 | EpisodicNode → `entity_edges` → EntityEdge | `_process_episode_data`; `graphiti.py:1031-1035` | episode supports fact edge | EV-015 |
| F-E-03 | duplicate edge → existing `episodes` list append | `_dedupe_extracted_edge`; `edge_operations.py:930-1030` | multiple episodes support same edge | EV-046 |
| F-E-04 | EntityEdge → search context fact | `search_results_to_context_string`; `search_helpers.py` | fact text + validity window | EV-067/068 |
| F-E-05 | search context → no edge UUID | same | model cannot identify edge record | EV-113 |
| F-E-06 | search context → no episode UUID | same | model cannot identify source episode | EV-114 |
| F-E-07 | `add_triplet` no episode → edge with dangling synthetic UUID | `add_triplet`; `graphiti.py:2140-2156`; `append_episode_provenance`; `edge_operations.py:884-890` | `edge.episodes` contains an unsaved episode UUID | EV-024/050; IF-13/14 |
| F-E-08 | unsupported edge deleted → provenance lost | `remove_episode`; `graphiti.py:2174-2200` | deleting last support deletes edge | EV-026 |
| F-E-09 | supported edge survives → provenance shrinks | same | episode UUID removed, edge remains | EV-026 |
| F-E-10 | source episode deleted → invalidation remains | same | derived temporal state not rolled back | EV-051/052 |

**Evidence flow conclusion (Observation)**
Graphiti 在图内保存了 `episode → edge → entity` 的链，但这条链在三个位置可被削弱：
1. 配置 `store_raw_episode_content=False` 使原文不可回读；
2. `add_triplet` 在无真实 episode 时写入一个指向不存在 episode 的 UUID（伪溯源，不是“无 episode”）；
3. LLM 上下文不暴露 UUID，模型侧无法回溯。
因此“可追溯”是**API/图数据库层的能力**，不是“LLM 上下文层”的保证。

## 5. Authority Flow

| ID | Authority Source → Target | Symbol / File | Enforcement | EV |
|---|---|---|---|---|
| F-A-01 | caller `group_id` → ASCII validator | `validate_group_id`; `helpers.py:145-174` | reject invalid chars | EV-078 |
| F-A-02 | validated `group_id` → driver/database | `_resolve_request_scope`; `graphiti.py:1304-1332` | clone driver for non-default — **only `add_episode`/`add_episode_bulk`** | EV-006/043; IF-16–19 |
| F-A-03 | HTTP caller → REST router | `server/graph_service/main.py` | route dispatch | EV-094 |
| F-A-04 | REST request → handler | routers | **no auth middleware** | EV-095/096 |
| F-A-05 | MCP client → MCP tool | `graphiti_mcp_server.py` | tool dispatch | EV-100/105 |
| F-A-06 | MCP request → handler | same | **no auth** | EV-096/098 |
| F-A-07 | episode content/tool input → LLM | extraction/search context | **no prompt-injection filter** | EV-097 |
| F-A-08 | LLM → duplicate/contradiction labels | `prompts/dedupe_edges.py` | model judgment | EV-042-045 |
| F-A-09 | labels → deterministic invalidation | `_apply_temporal_invalidation` | code applies time arithmetic | EV-036/037 |
| F-A-10 | `add_triplet` caller → graph | `add_triplet`; `graphiti.py:2033-2173` | direct write bypasses extraction | EV-023/050 |
| F-A-11 | MCP admin caller → graph deletion | `clear_graph` tool | delete all data | EV-105 |
| F-A-12 | deployment gateway → Graphiti | external | optional auth at network edge | EV-096/098 |
| F-A-13 | `add_triplet`/`remove_episode`/`get_nodes_and_edges_by_episode` → default database | those methods; `graphiti.py:2020-2251` | no request-scope resolution; non-default group can read/write/delete wrong DB | IF-16–19 |

**Authority flow conclusion (Fact + Observation)**
- 项目内部唯一的“权限”是 `group_id` 的输入校验与分区路由。
- 认证/授权/调用方隔离没有任何代码层执行点；`SECURITY.md` 明确把这一层外推给部署方。
- LLM 对持久状态只有“提名权”，执行权在代码；但直接写路径（`add_triplet`）绕过抽取，仍受 dedupe/时间算术约束，却不受认证约束。

## 6. Memory Flow

| ID | Memory Operation | Symbol / File | Scope | EV |
|---|---|---|---|---|
| F-M-01 | retrieve last 10 episodes | `retrieve_episodes`; `graphiti.py:1426` | per `group_id` | EV-007/072 |
| F-M-02 | filter `valid_at <= reference_time` | `graph_data_operations.py:95-120` | per reference time | EV-070 |
| F-M-03 | order `valid_at DESC, created_at DESC`, then reverse | same | per group | EV-071 |
| F-M-04 | explicit `previous_episode_uuids` override | `get_by_uuids`; `graphiti.py:1445` | caller-selected | EV-008/074 |
| F-M-05 | saga `NEXT_EPISODE` chain | `_get_or_create_saga`; `graphiti.py:1047-1095` | per saga | EV-058/059 |
| F-M-06 | community summaries | `build_communities`; `graphiti.py:1867-1912` | per group | EV-019/054 |
| F-M-07 | edge search scope | `search()`; `search.py:264+` | group filter | EV-061/063 |
| F-M-08 | episode search scope | `search()`; `search.py:681+` | group filter | EV-061/063 |
| F-M-09 | community search scope | `search()`; `search.py:783+` | group filter | EV-061/063 |
| F-M-10 | context facts with validity windows | `search_results_to_context_string` | query result set | EV-067-069 |

**Memory flow observation**
- The “memory” given to the LLM is a **query-time projection**, not a stable per-agent state.
- Context includes invalid facts if they rank in top-k; the only guard is the textual validity window.
- Saga provides narrative ordering; communities provide cluster summaries; neither replaces provenance.

## 7. Policy Flow

| ID | Policy Artifact → Behavior | Symbol / File | Enforcement | EV |
|---|---|---|---|---|
| F-P-01 | “no authn/authz” → deploy behind own controls | `SECURITY.md:16,21` | documentation only | EV-096 |
| F-P-02 | “same trust in group_id” → app decides content | `SECURITY.md:23` | documentation only | EV-099 |
| F-P-03 | no recency-based contradiction → LLM prompt | `prompts/dedupe_edges.py` | soft prompt constraint | EV-044 |
| F-P-04 | explicit contradiction language → LLM prompt | same | soft prompt constraint | EV-045 |
| F-P-05 | group id ASCII → validator | `helpers.py:145-174` | hard input check | EV-078 |
| F-P-06 | raw content retention → flag | `store_raw_episode_content` | hard config switch | EV-001/016 |
| F-P-07 | retention/deletion policy → external | no code path | unenforced | EV-051/052 |
| F-P-08 | community rebuild → no approval gate | `build_communities` | hard destructive call | EV-054 |
| F-P-09 | delete episode → no invalidation rollback | `remove_episode` | unenforced lifecycle rule | EV-051/052 |

**Policy flow conclusion**
Graphiti 的 Policy Flow **不是一个闭环**：它有“策略声明”（SECURITY.md、prompt 规则、配置开关），但缺少“审批 → 强制执行 → 未来决策”的闭环。`group_id` 校验与 `store_raw` 是硬执行点；认证、授权、删除回滚、保留策略都外推给部署方。这不是缺陷断言，而是边界事实。

## 8. Flow → KO Cross-Check

| KO | Flow edges | Status |
|---|---|---|
| KO-01 | F-C-03, F-S-03, F-S-09, F-S-10, F-D-06 | 一致 |
| KO-02 | F-A-08, F-A-09, F-S-09, F-C-09 | 一致 |
| KO-03 | F-S-09, F-E-04, F-M-10, F-S-14 | 一致 |
| KO-04 | F-A-01…F-A-06, F-A-12 | 一致 |
| KO-05 | F-S-16, F-A-05, F-A-06 | 一致（运行时未验证） |
| KO-06 | F-C-12, F-C-16, F-D-08 | 一致（按驱动差异修正） |
| KO-07 | F-E-01…F-E-10, F-D-11 | 一致 |
| KO-08 | F-C-01, F-C-02, F-M-01…F-M-04, F-S-02 | 一致 |

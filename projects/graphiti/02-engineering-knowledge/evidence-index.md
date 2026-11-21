# Evidence Index

> 所有行号基于 `747b480ec6b12ecb718f1e2bc588b20ee36ea9c2`。V = verified（直接读到代码/文档）；I = inferred（由代码推导，未经运行验证）。

| ID | Fact | Source | Strength |
|---|---|---|---|
| E-01 | 包 `graphiti-core` 版本 0.31.2，Python `>=3.10,<4`，Apache-2.0 | `pyproject.toml:2,5,9` | V·S3 |
| E-02 | HEAD `747b480e…`，main，2026-11-06 04:49:50 +0000，CLA 签名 commit | `git log -1` | V·S3 |
| E-03 | 386 文件 / 300 py / 76,764 py 行 / test_*.py 81 个 / 15 workflow（排除 `.git`，可复现口径） | `os.walk` 复核 | V·S3 |
| E-04 | `Node` 基类：`uuid,name,group_id,labels,created_at` | `nodes.py:93-98` | V·S3 |
| E-05 | `EpisodicNode`：`source,source_description,content,valid_at,entity_edges,episode_metadata` | `nodes.py:318-329` | V·S3 |
| E-06 | `EntityNode`：`name_embedding,summary,attributes` | `nodes.py:499-502` | V·S3 |
| E-07 | `CommunityNode`：`name_embedding,summary` | `nodes.py:727-729` | V·S3 |
| E-08 | `SagaNode`：`summary,first/last_episode_uuid,last_summarized_at,last_summarized_episode_valid_at` + 4 个私有计数 | `nodes.py:907-918` | V·S3 |
| E-09 | `Edge` 基类：`uuid,group_id,source/target_node_uuid,created_at` | `edges.py:49-54` | V·S3 |
| E-10 | `EntityEdge`：`name,fact,fact_embedding,episodes,expired_at,valid_at,invalid_at,reference_time,attributes` | `edges.py:263-288` | V·S3 |
| E-11 | `add_episode` 签名含 `group_id,uuid,update_communities,entity_types,excluded_entity_types,previous_episode_uuids,edge_types,edge_type_map,custom_extraction_instructions,saga,saga_previous_episode_uuid` | `graphiti.py:1334-1352` | V·S3 |
| E-12 | 取 previous episodes：`retrieve_episodes(reference_time, last_n=RELEVANT_SCHEMA_LIMIT, group_ids=[group_id], source=source)`，或按显式 UUID | `graphiti.py:1450-1458` | V·S3 |
| E-13 | Episode 以 `created_at=now`、`valid_at=reference_time` 构造 | `graphiti.py:1458-1475` | V·S3 |
| E-14 | 节点抽取：`extract_nodes(clients, episode, previous_episodes, entity_types, excluded_entity_types, custom_extraction_instructions)` | `graphiti.py:1477-1484` | V·S3 |
| E-15 | 节点解析：`resolve_extracted_nodes(...)` 返回 nodes/uuid_map | `graphiti.py:1486-1493` | V·S3 |
| E-16 | 边抽取与属性抽取"并行"（注释：只把新边给摘要，避免重复既有事实） | `graphiti.py:1495-1530` | V·S3 |
| E-17 | `_process_episode_data(..., saga, saga_previous_episode_uuid, node_episode_index_map)` 负责落库与 saga 关联 | `graphiti.py:1528` | V·S3 |
| E-18 | `_resolve_request_scope` 返回 per-call driver + `clients.model_copy(update={'driver':driver})`；默认库路径不 clone 不 copy | `graphiti.py:1304-1330` | V·S3 |
| E-19 | 注释说明历史 bug：共享 `self.driver` 被并发请求中途改写 → episode 静默写入错误 graph（issue #1676） | `graphiti.py:1311-1319, 1428-1431` | V·S3 |
| E-20 | 候选三路采集：identity property / exact name / semantic，合并后按顺序去重 | `node_operations.py:456-480` | V·S3 |
| E-21 | 去重候选上限 `NODE_DEDUP_CANDIDATE_LIMIT=15`、余弦下限 `NODE_DEDUP_COSINE_MIN_SCORE=0.6` | `node_operations.py:67-68` | V·S3 |
| E-22 | 语义候选：批量 embedding → `node_similarity_search(..., 15, 0.6)`；`create_batch` 未实现时退化逐条 `create` | `node_operations.py:585-635` | V·S3 |
| E-23 | 仅"identity 与 similarity 都未决"的节点进入 `_resolve_with_llm` 批量裁决 | `node_operations.py:868-883` | V·S3 |
| E-24 | 解析顺序：identity→similarity→LLM；无候选者原样作为新节点 | `node_operations.py:816-910` | V·S3 |
| E-25 | `enforce_edge_type_signatures`：签名类型但端点对不匹配 → strict 时丢弃，否则改名 `DEFAULT_EDGE_NAME` 并清空 attributes | `edge_operations.py:671-714` | V·S3 |
| E-26 | `_apply_temporal_invalidation`：自身带 `invalid_at` 且无 `expired_at` → 视为历史；候选起始晚于新边起始 → 新边失效 | `edge_operations.py:715-747` | V·S3 |
| E-27 | `resolve_edge_contradictions`：仅当"候选在新区间之外"或"候选更早"时把候选置失效 | `edge_operations.py:748-786` | V·S3 |
| E-28 | 边时间戳由 LLM 抽取（`_extract_edge_timestamps`） | `edge_operations.py:786-833` | V·S3 |
| E-29 | `saga` 参数语义：字符串按名字取/建；episode 由 HAS_EPISODE 连，连续 episode 由 NEXT_EPISODE 连 | `graphiti.py:1392-1400` | V·S3 |
| E-30 | `summarize_saga` 存在；SagaNode 记录 `last_summarized_at` / `last_summarized_episode_valid_at` | `graphiti.py:590`、`nodes.py:907` | V·S3 |
| E-31 | 检索配方 **16** 个：COMBINED（RRF/MMR/CROSS_ENCODER）+ EDGE/NODE（RRF/MMR/NODE_DISTANCE/EPISODE_MENTIONS/CROSS_ENCODER）+ COMMUNITY（RRF/MMR/CROSS_ENCODER） | `search_config_recipes.py:34-217` | V·S3（独立审计修正） |
| E-32 | `Graphiti.search` 默认 `EDGE_HYBRID_SEARCH_RRF`；`center_node_uuid` 存在时改 `EDGE_HYBRID_SEARCH_NODE_DISTANCE`，并 `limit=num_results` | `graphiti.py:1913-1970` | V·S3 |
| E-33 | `Graphiti._search` 标注 **DEPRECATED**，转调 `search_` | `graphiti.py:1974-1988` | V·S3 |
| E-34 | `search` 按 config 分派 edge/node/community；方法含 bm25/cosine_similarity/bfs；BFS 需 origin 节点 | `search.py:98-341` | V·S3 |
| E-35 | `node_search` 支持 bm25/cosine/bfs，BFS 无 origin 时明确处理 | `search.py:473-553` | V·S3 |
| E-36 | `episode_search` 独立实现 | `search.py:676-682` | V·S3 |
| E-37 | `SearchFilters` 支持属性/时间过滤，用于检索裁剪 | `search_filters.py`、`graphiti.py:1935` | V·S3 |
| E-38 | `GraphDriver(QueryExecutor, ABC)`：session/execute/transaction 等抽象 | `driver.py:90-129` | V·S3 |
| E-39 | `GraphDriver.clone(database)` 基类默认 `NotImplementedError`；各后端覆写 | `driver.py:131-142` | V·S3 |
| E-40 | 4 个后端目录：neo4j / falkordb / kuzu / neptune | `driver/` | V·S3 |
| E-41 | 能力缺失用 `NotImplementedError` 逐驱动降级（identity lookup、similarity search、batch embed） | `node_operations.py:516-525, 585-600` | V·S3 |
| E-42 | Namespace 层 733 行：`nodes/edges` 各类型命名空间 | `graphiti_core/namespaces/` | V·S3 |
| E-43 | spec 目标三层：Client → Namespace（薄编排）→ Operations ABC → Driver；Phase 1 非破坏 | `spec/driver-operations-redesign.md` | V·S2（设计） |
| E-44 | spec LLMRuntime：必需 default model，可选 prompt routes / overrides，schema 固化 | `spec/llm-runtime.md:1-45` | V·S2（设计） |
| E-45 | `LLMRuntime.resolve_model/resolve_builder/complete` 实现 | `llm_client/llm_runtime.py:54-167` | V·S3 |
| E-46 | 测试 `test_concurrent_completes_do_not_serialize_or_clobber_model` 验证并发不串模型 | `tests/llm_client/test_llm_runtime.py:271` | V·S4 |
| E-47 | 分块：`CHARS_PER_TOKEN=4`；`chunk_json_content(chunk_size_tokens, overlap_tokens)`；JSON 数组按元素、对象按字段切分并保留 overlap | `content_chunking.py:36,215-310` | V·S3 |
| E-48 | MCP **13** 个 `@mcp.tool()`：add_memory/search_nodes/search_memory_facts/delete_entity_edge/delete_episode/get_entity_edge/get_episodes/summarize_saga/build_communities/add_triplet/get_episode_entities/clear_graph/get_status | `mcp_server/src/graphiti_mcp_server.py:403-1146` | V·S3（独立审计修正） |
| E-49 | `clear_graph` 直接按 group 清空，无任何权限检查 | `mcp_server.py:1101-1143` | V·S3 |
| E-50 | REST 10 端点；`POST /messages` 返回 **202** 并入 `async_worker.queue` 串行处理 | `routers/ingest.py:51-72`、`routers/retrieve.py:17-44` | V·S3 |
| E-51 | `SECURITY.md`：参考服务无认证/无授权/无 prompt-injection 防护/不隔离调用方/默认凭据/同 group 内 episode 同等信任，且列为 out of scope | `SECURITY.md:1-40` | V·S3 |
| E-52 | README：Graphiti = 开源框架（BYO graph DB / LLM / embedder，自定义 tenancy、access control、ops）；Zep = 统一治理层 + Konig | `README.md:80-120` | V·S3 |
| E-53 | AGENTS.md：`make test` 隐式需 Neo4j；`test_add_triplet.py` 11 个既有失败且 CI 不跑；FalkorDB 并发断连 | `AGENTS.md`（Tests / Backend gotcha 段） | V·S3 |

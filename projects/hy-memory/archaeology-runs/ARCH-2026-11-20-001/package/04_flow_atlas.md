# 04 Flow Atlas — Hy-Memory 2.0

- run_id: `ARCH-2026-11-20-001`
- source: `hy_memory/` at `d8138b501a755f662a024f7e8eb879613eabf1d3`; every edge is read from code, not inferred from README
- edge schema: `id | from → to | condition | symbol / file | evidence`
- seven classes: Control / State / Data / Evidence / Authority / Memory / Policy
- notation: `CF` control, `SF` state, `DF` data, `EF` evidence, `AF` authority, `MF` memory, `PF` policy

## 1. Control Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| CF-01 | `HyMemoryClient.add` → `async_add` → `_async_add_impl` | sync public call; async call enters directly | `client.py:1729`, `:2875`, `:3010` | code |
| CF-02 | `_async_add_impl` → `_normalize_add_request` | always | `client.py` add prologue | code |
| CF-03 | `_async_add_impl` → chat branch | `scene_forced == "chat"` after tool stripping | `client.py` routing block | code |
| CF-04 | `_async_add_impl` → `classify_messages_scene` | `scene is None`, `content_type=="messages"`, mode != lite, `config.coding` present/enabled, and tool message present | `client.py:3035-3125`; `coding/judge.py` | code |
| CF-05 | `classify_messages_scene` → chat fallback | classifier raises, returns unknown, or explicit `scene="coding"` has no tool message | `client.py:3035-3125` | code |
| CF-06 | `_async_add_impl` → `MemoryWriter.write` (chat) | resolved route is chat (explicit chat, no tool, or judge says chat) | `writer.py:2570` | code |
| CF-07 | `_async_add_impl` → coding writer | `coding_available` (mode != lite, messages input, `config.coding.enable`) and (`scene_forced=="coding"` or tool message) and `is_coding`; explicit coding without tools falls back to chat | `client.py:3035-3125`; `coding/writer.py` | code |
| CF-08 | `_async_add_impl` → working writer | resolved scene == working | `client.py` working branch | code |
| CF-09 | `MemoryWriter.write` → `_run_agent` → extractor | mode != lite and L1 upsert did not hard-fail | `writer.py:2570-2760`, `:3265` | code |
| CF-10 | `_run_agent` → `reconcile` | extraction produced at least one item | `writer.py`; `agent/reconciler.py:397` | code |
| CF-11 | `_reconcile_and_store` → `_reconcile_and_store_locked` | always; lock acquired per user | `writer.py:1237-1259` | code |
| CF-12 | `_reconcile_and_store_locked` → reconciler | lock held | `writer.py:1259`; `reconciler.py:397` | code |
| CF-13 | `_reconcile_and_store_locked` → persist operations | `reconcile.success == True` | `writer.py:1260-1310` | code |
| CF-14 | `_reconcile_and_store_locked` → `ReconcileFailure` | `success == False` | `writer.py:1300-1315` | code |
| CF-15 | `MemoryWriter.write` → `_rollback_failed_l1` | extraction/agent failure | `writer.py:3118` | code |
| CF-16 | `_rollback_failed_l1` → `vector_store.delete` → `get_by_id` confirm | always on failure path | `writer.py:3118-3175` | code |
| CF-17 | `writer` → graph projection | graph enabled and projection scheduled | `writer.py:1059` `_schedule_graph_projection` | code |
| CF-18 | `writer` → `_schedule_episode_summary` | session/dynamic/series episode enabled | `writer.py:990` | code |
| CF-19 | `writer` → tag/entity index | entity extraction enabled and non-empty | `writer.py:_maybe_index_entities` | code |
| CF-20 | `HyMemoryClient.search` → `async_search` → `_async_search_impl` | sync/async public entry | `client.py:1847`, `:3949`, `:4024` | code |
| CF-21 | `_async_search_impl` → `_search_all_scenes` | `scene == "all"` | `client.py:3830` | code |
| CF-22 | `_search_all_scenes` → chat/working/coding lanes | always, via `gather` | `client.py:3830-3875` | code |
| CF-23 | `_async_search_impl` → single reader | explicit scene or resolved scene | `client.py` routing block | code |
| CF-24 | `BaseReadPipeline.read` → `_read_once` | always; optional underflow second pass | `reader_base.py:1057`, `:1110` | code |
| CF-25 | `_read_once` → VDB semantic | dense channel always available | `reader_base.py` channels | code |
| CF-26 | `_read_once` → BM25 channel | Qdrant text index or rank-bm25 enabled | `reader_base.py`; `data/vector_store_qdrant.py` | code |
| CF-27 | `_read_once` → graph schema/intention side channel | graph available and scene ultra/pro-with-graph | `reader_base.py` graph block | code |
| CF-28 | `read` → `_apply_graph_rerank` | `graph_rerank_enabled` and graph available | `reader_base.py:379`, `:599` | code |
| CF-29 | `search` → `_spawn_search_dedup` | dedup enabled and L2/L4 results non-empty | `client.py:3617`, `:4552` | code |
| CF-30 | `evolve` → `System2Writer.evolve` | ultra mode and explicit call | `client.py:2389`; `system2_writer.py:1172` | code |
| CF-31 | `evolve` → `run_system2_agent` | clusters exist | `system2_writer.py:1205` | code |
| CF-32 | `run_system2_agent` → `complete_messages` | always, exactly one call | `system2_agent.py:1938-2025` | code |
| CF-33 | `run_system2_agent` → `System2ToolExecutor.execute` | JSON parsed with ≥1 operation | `system2_agent.py`; `system2_tools.py:419` | code |
| CF-34 | `evolve` → `_run_cross_domain_sweeper` | `should_continue == False` | `system2_writer.py:1205-1265` | code |
| CF-35 | `ComponentFactory.get_reader` → `MemoryReader.initialize` | cache miss | `registry.py:121-171` | code |

## 2. State Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| SF-01 | L1 `ACTIVE` → `SHADOW` | agent run succeeded, regardless of extraction count | `writer.py:3589-3615` | code |
| SF-02 | L1 `ACTIVE` → deleted | extraction failed and delete confirmed | `writer.py:3118-3175` | code |
| SF-03 | L1 `ACTIVE` → `ACTIVE` | extraction failed and rollback ambiguous | `writer.py` 409 path | code |
| SF-04 | old L2 `ACTIVE` → `SUPERSEDED` | reconcile chooses SUPERSEDE | `writer.py` persist ops; `reconciler.py:82` | code |
| SF-05 | old L2 `ACTIVE` → `SHADOW` | reconcile chooses UPDATE | `writer.py` UPDATE branch | code |
| SF-06 | old node `is_latest=True` → `False` | supersede/update | `writer.py` chain update | code |
| SF-07 | `supersedes` / `superseded_by` edge created | new version persisted | `writer.py` chain block | code |
| SF-08 | no stored memory | reconcile failure | `writer.py:1300-1315` | code |
| SF-09 | episode state not advanced | reconcile failure | `writer.py` comment at `:1237` | code |
| SF-10 | episode finalized | `finalize_session` true and finalize succeeds | `writer.py` finalize block | code |
| SF-11 | base memory persisted + 409 | dynamic/series finalize fails | `writer.py`; `client.py:1823-1845` | code |
| SF-12 | `last_accessed_at` updated | strength enabled and search hit | `client.py:4552-4580` | code |
| SF-13 | L7 `ACTIVE` → L2 | intention expired on read | `_retrieval/intention.py:243` | code |
| SF-14 | duplicate L7 → deleted | cosine ≥ 0.95 and same day/valid_until | `intention.py` `_partition_duplicates` | code |
| SF-15 | L2/L4 → `SHADOW` + `shadow_reason=dedup:*` | background dedup winner chosen | `dedup.py:415-453` | code |
| SF-16 | chain node IDs also shadowed | dedup loser is a chain head | `dedup.py` `chain_node_ids` | code |
| SF-17 | working fact superseded → summary preserved | new fact replaces old | `coding/store.py` `fact_chain_summary` | code |
| SF-18 | coding diff thread append | `observe()` called with a patch | `coding/chain_store.py:180` | code |
| SF-19 | graph node created | System 2 `create_graph_node` operation | `system2_tools.py:599` | code |
| SF-20 | graph node retired | `retire_schema` operation | `system2_tools.py:790-915` | code |

## 3. Data Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| DF-01 | message content → embedding vector | every write in every mode | `core/embed_service.py:482` | code |
| DF-02 | message content → `MemoryNode.content` | L1 node construction | `writer.py:2570-2700` | code |
| DF-03 | recent L1 history → extractor prompt | mode != lite | `writer.py:2739`, `:2977` | code |
| DF-04 | extractor JSON → `MemoryNode[]` | parse success | `writer.py:_run_agent` | code |
| DF-05 | extracted item → reconcile candidate query | per item | `reconciler.py:397` | code |
| DF-06 | candidate vector + BM25 → candidate list | candidate retrieval | `reconciler.py` hybrid block | code |
| DF-07 | chain head content → reconcile prompt | chain has versions | `reconciler.py` `trace_full_chains` | code |
| DF-08 | LLM ops JSON → ADD/SUPERSEDE/UPDATE nodes | parse success | `reconciler.py:1321` `_parse_ops` | code |
| DF-09 | new node → VDB upsert | persist | `writer.py` persist ops | code |
| DF-10 | new node → graph projection | graph write enabled | `writer.py:1059` | code |
| DF-11 | query text → query embedding | search | `client.py` search prep; `embed_service.embed` | code |
| DF-12 | query text → BM25 terms | BM25 channel active | `reader_base.py` keyword channel | code |
| DF-13 | semantic/BM25/tag scores → additive final score | normal/hybrid read | `reader_base.py:1157-1199` | code |
| DF-14 | lane ranks → RRF score | `scene=all` | `client.py` `_rrf_fuse_lanes` | code |
| DF-15 | search results → L2/L4 dedup embedding groups | dedup enabled | `client.py:3617` | code |
| DF-16 | coding query vector → scoped business IDs → filtered hits | coding scope supplied | `coding/store.py:577` | code |
| DF-17 | patch → thread/round record | `observe()` | `coding/chain_store.py:180` | code |
| DF-18 | full prompt + full response → `pipeline_logs_v2` | trace enabled (default) | `data/cache_sqlite.py:67-85` | code |
| DF-19 | old/new content → `history.db` | history enabled (default) | `data/history_store.py:record_*` | code |
| DF-20 | schema evidence IDs → `VdbRef` → `DERIVED_FROM` | `create_graph_node` / `add_evidence` | `system2_tools.py:599-790`, `:915-974` | code |
| DF-21 | cluster facts → System 2 prompt | clusters exist | `system2_writer.py` preprocessing | code |
| DF-22 | System 2 JSON → graph ops + stats | parse success | `system2_tools.py:419` | code |

## 4. Evidence Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| EF-01 | extracted memory → `evidence_chain` | extraction carries source IDs | `models/memory.py:223-270` | code |
| EF-02 | L1 raw ID → L2/L4 `source_raw_memory_id` | extraction from raw | `models/memory.py`; `writer.py` | code |
| EF-03 | old version → new version via `supersedes` | SUPERSEDE/UPDATE | `writer.py` chain block | code |
| EF-04 | new version → old via `superseded_by` | chain persistence | `writer.py` chain block | code |
| EF-05 | episode → `source_session_id` / `source_turn_index` | episode summary written | `models/memory.py`; `episode_summary.py` | code |
| EF-06 | graph node → `VdbRef` | graph node created | `system2_tools.py:ensure_vdbref` | code |
| EF-07 | graph node → `DERIVED_FROM` edge | evidence ID supplied | `system2_tools.py:add_derived_from` | code |
| EF-08 | evidence ID → owner check | `add_evidence` only | `system2_tools.py:915-974` | code |
| EF-09 | evidence ID → no owner/existence check | `create_graph_node` path | `system2_tools.py:630-735` | code |
| EF-10 | search result → dedup `dedup_kept_id` | dedup loser shadowed | `dedup.py:415-453` | code |
| EF-11 | delete → history `record_delete(content=old_content)` | VDB delete success and history enabled | `client.py:5450-5465` | code |
| EF-12 | ADD/UPDATE → history full new/old content | history enabled | `history_store.py:record_add/update` | code |
| EF-13 | LLM call → pipeline log prompt/response | trace enabled | `pipeline_observability.py:9-10`; `cache_sqlite.py` | code |
| EF-14 | search → `RECONCILE_CANDIDATES` log fields | candidate pool built | `reconciler.py` logging | code |
| EF-15 | coding chain → `seq` call count | `observe()` | `coding/chain_store.py:180` | code |

## 5. Authority Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| AF-01 | public `search` → `user_ids` validation | always, empty list rejected | `client.py:4125-4129` | code |
| AF-02 | public `search` → `_build_isolation_params` | always, after validation | `reader_base.py:3643` | code |
| AF-03 | `_build_isolation_params` → `user_ids` only | no agent/session filter | `reader_base.py:3643-3740` | code |
| AF-04 | `_build_isolation_params` → exact `user::agent` | `request.agent_id` present; no `agent_ids` and no `session_ids` | `reader_base.py:3643` | code |
| AF-05 | `_build_isolation_params` → `user::agent::session` combinations | sessions supplied | `reader_base.py` | code |
| AF-06 | `_build_isolation_params` → `error_msg` | multiple agent_ids and multiple session_ids | `reader_base.py` | code |
| AF-07 | vector store → no user condition | isolation_keys, isolation_key, user_ids all absent | `vector_store_qdrant.py:1195-1245` | code |
| AF-08 | `get` / `update` / `delete` → node by ID | caller supplies a memory_id only | `client.py:2016`, `:2042`, `:2072` | code |
| AF-09 | `delete_all` → agent×session rounds | only sessions supplied | `client.py:5581-5720` | code |
| AF-10 | `add_evidence` → schema/evidence owner check | both sides must match user and agent | `system2_tools.py:915-974` | code |
| AF-11 | `revise_schema` / `retire_schema` → old schema owner check | always | `system2_tools.py:790-915` | code |
| AF-12 | `create_graph_node` → no evidence owner check | evidence ID supplied | `system2_tools.py:630-735` | code |
| AF-13 | `add_edge` → no endpoint scope check | edge operation executed | `system2_tools.py:990-998` | code |
| AF-14 | L0 profile → user-level node | any agent | `agent/tools/basic_profile.py:53-80` | code |
| AF-15 | source filtering → `mixed` excluded | any include/exclude active and mixed not explicit | `utils/source_filter.py` | code |
| AF-16 | history → `user:agent` key | history write | `client.py:852-859` | code |
| AF-17 | graph tag expansion → Cypher string interpolation | `expand_with_tags` called | `graph_store_kuzu.py:2307` | observation; no reader call site found |

## 6. Memory Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| MF-01 | conversation → L1_RAW | every write, before extraction | `writer.py:2570` | code |
| MF-02 | L1_RAW → SHADOW | agent success, including empty extraction | `writer.py:3589` | code |
| MF-03 | extraction → L2_FACT / L4_IDENTITY | extraction output | `writer.py:_run_agent` | code |
| MF-04 | reconcile ADD → new L2/L4 ACTIVE node | no candidate or LLM ADD | `reconciler.py`; `writer.py` | code |
| MF-05 | reconcile SUPERSEDE → new version + old SUPERSEDED | LLM SUPERSEDE | `reconciler.py:82`; `writer.py` | code |
| MF-06 | reconcile UPDATE → old SHADOW + new ACTIVE | LLM UPDATE | `reconciler.py`; `writer.py` | code |
| MF-07 | session summary → L3_SUMMARY | episode finalize | `episode_summary.py` | code |
| MF-08 | identity fact → L4_IDENTITY | extraction classifies identity | `models/memory.py`; prompts | code |
| MF-09 | schema → L6_SCHEMA | System 2 `create_graph_node` / `revise_schema` | `system2_tools.py` | code |
| MF-10 | intention → L7_INTENTION | extraction creates intention | `models/memory.py`; `_retrieval/intention.py` | code |
| MF-11 | expired intention → L2_FACT | read-time expiry | `intention.py:243` | code |
| MF-12 | working fact → `WorkingFact` row | working write | `coding/store.py` working tables | code |
| MF-13 | working experience → `Experience` row | confidence ≥ value threshold | `coding/store.py` working tables | code |
| MF-14 | coding patch → diff thread/round | `observe()` | `coding/chain_store.py:180` | code |
| MF-15 | coding experience → business row + `*_cv` vector | coding write | `coding/store.py`; `coding/writer.py` | code |
| MF-16 | graph projection → graph node/edge | graph write enabled | `writer.py:1059` | code |

## 7. Policy Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| PF-01 | mode `lite` → raw-only, zero LLM | always | `client.py:4171-4195`; README | code |
| PF-02 | mode `pro` → extraction + reconcile, graph default off | always unless env overrides | `config.py` `MEMORY_ENABLE_GRAPH`; `client.py` init | code |
| PF-03 | mode `ultra` → graph mandatory + System 2 available | always | `client.py` graph gate; README | code |
| PF-04 | reconcile LLM failure → zero writes, no ADD-all | `success=False` | `writer.py:1300-1315` | code |
| PF-05 | empty extraction → L1 SHADOW, not delete | agent success, `memory=[]` | `writer.py:3589-3615` | code |
| PF-06 | ambiguous rollback → 409, retry unsafe | delete unconfirmed or partial higher-level write | `writer.py:3118-3175` | code |
| PF-07 | dynamic/series finalize failure → 409, partial persist | finalize enabled and fails | `writer.py` finalize block | code |
| PF-08 | search dedup → background SHADOW | `MEMORY_SEARCH_DEDUP_ENABLED=true` | `client.py:3617`; `dedup.py` | code |
| PF-09 | access bump → off | `RecallConfig.strength_enabled=False` default | `config.py:330-370` | code |
| PF-10 | underflow fill → off | `MEMORY_HYBRID_V2_FILL_UNDERFLOW=0` default | `reader_base.py:1057` | code |
| PF-11 | coding chain block → off | `chain_block_enabled=False` default; `HCM_BLOCK_OFF=1` disables | `coding/chain_store.py` | code |
| PF-12 | history → on, full text | `HistoryConfig.enable=True` default | `config.py:1140-1160` | code |
| PF-13 | pipeline trace → on, full prompt/response | `MEMORY_PIPELINE_LOG_ENABLED` / `TRACE_ENABLED` default true | `utils/pipeline_observability.py:9-10` | code |
| PF-14 | memory operations audit → off | `MEMORY_MEMORY_OPERATIONS_ENABLED=false` default | observability config | code |
| PF-15 | schema dedup / compaction → off | `MEMORY_EVOLING_SCHEMA_DEDUP_ENABLED=false`; `..._COMPACTION_ENABLED=false` | `config.py`; `system2_writer.py` | code |
| PF-16 | graph rerank → off | `graph_rerank_enabled` default false | `config.py` `GraphStoreConfig` | code |
| PF-17 | local rerank → off | `local_rerank_enabled=False` default | `config.py:RecallConfig` | code |
| PF-18 | episode recall → off | `episode_recall_enabled=False`, `session/dynamic/series=false` | `config.py:RecallConfig` | code |

## 8. Edge classes at a glance

| class | count | dominant property |
|---|---:|---|
| Control | 35 | synchronous SDK calls bridged to two event loops; explicit routing branches |
| State | 20 | provisional raw + shadow lifecycle + evolution chains |
| Data | 22 | five independent stores; no global transaction |
| Evidence | 15 | raw→derived and node→VdbRef→DERIVED_FROM; asymmetric checks |
| Authority | 17 | distributed enforcement, ID-trust at record level |
| Memory | 16 | layered L0–L7 + working/coding two-tier |
| Policy | 18 | defaults materially narrow the advertised feature set |

**Total edges: 143.**

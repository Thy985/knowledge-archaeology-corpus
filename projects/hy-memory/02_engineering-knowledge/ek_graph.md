# 02 Engineering Knowledge — Hy-Memory 2.0 (EK Graph)

- run_id: `ARCH-2026-11-20-001`
- scope: `hy_memory/`, `tests/`, packaging/docs constraints; `memhub/` and `plugins/` as boundary only
- numbering: `EK-01…EK-53`
- edge types: `mechanism` / `subsystem` / `causal` / `dependency` / `constraint` / `contrast`
- rule: every EK declares at least one link; no isolated facts.
- evidence notation: `file:line` means the line was read in this run. `symbol` means the function/class was read but the exact line is not asserted.

## A. Product shape and identity

### EK-01 The unit of design is a scene-specific memory, not a generic vector table
- category: ARCHITECTURE / MEMORY
- layer: L1
- epistemic: Fact
- statement: The SDK routes writes and reads into three scenes: `chat` (hierarchical L0–L7), `working` (one session fact + experiences), and `coding` (working schema + code diff chain). Routing is based on presence of tool messages and an LLM classifier when `scene` is not explicit; no tool messages go straight to `chat`.
- evidence: `README.md` scene sections; `hy_memory/client.py:3010` `_async_add_impl`; `hy_memory/client.py:4024` `_async_search_impl`; `hy_memory/client.py:_scene_variant`.
- links: `subsystem`↔EK-02; `subsystem`↔EK-33; `contrast`↔EK-39.

### EK-02 Mode is a capability contract, not a performance knob
- category: ARCHITECTURE / CONFIG
- layer: L2
- epistemic: Fact
- statement: `lite` forces raw-only embedding, ignores `scene`/`layers`/`search_mode`, and makes no LLM calls. `pro` enables extraction + reconcile and leaves graph optional (default off). `ultra` forces graph on and enables explicit System 2 evolve. The same `MemoryWriter` serves all modes; mode changes orchestration and available features, not the writer class.
- evidence: `hy_memory/client.py:446-500` class docstring; `hy_memory/client.py:4171-4195` lite branch; `hy_memory/pipelines/registry.py:get_writer`; `hy_memory/config.py` graph defaults.
- links: `constraint`→EK-03; `dependency`→EK-41; `contrast`↔EK-21.

### EK-03 The taxonomy has eight concrete layers but is documented as seven
- category: MODEL / DOCUMENTATION
- layer: L1
- epistemic: Fact
- statement: `MemoryLayer` defines `L0_BASIC_INFO` through `L7_INTENTION` (8 values). The class docstring says “七层记忆模型” and lists only L1–L7. `all_layers()` returns 8. Consumers that iterate `MemoryLayer` therefore see L0; consumers that trust the prose may not.
- evidence: `hy_memory/models/memory.py:25-95`.
- links: `constraint`→EK-04; `contrast`↔EK-46.

### EK-04 The storage boundary is L0–L4 in VDB and L5–L7 in Graph, but L5 has no engine write path
- category: STORAGE / MODEL
- layer: L2
- epistemic: Fact
- statement: The enum docstring says L0–L4 live in the vector database and L5–L7 in the graph. The same docstring labels L5_KNOWLEDGE “暂未实现”; graph tools create only L6_SCHEMA/L7_INTENTION, and no engine write path for L5 was found. The L5 claim is therefore a code comment plus absence of writers in the audited engine, not a product-wide guarantee.
- evidence: `hy_memory/models/memory.py:31-42`; `hy_memory/pipelines/system2_tools.py:287-315` create tool enum; `hy_memory/data/graph_store_kuzu.py` node layer handling.
- links: `subsystem`↔EK-03; `dependency`→EK-42; `constraint`→EK-45.

### EK-05 SHADOW is one status with at least four causal meanings
- category: STATE / LIFECYCLE
- layer: L2
- epistemic: Fact
- statement: `SHADOW` is used for (a) an L1 raw node after any successful agent run, including an empty extraction; (b) the old node of a reconcile `UPDATE`; (c) a dedup loser; (d) low-confidence inferred content. It always means “not recalled,” but it does not encode why. Audit fields such as `shadow_reason` are written by some paths (dedup) and not others.
- evidence: `hy_memory/models/memory.py:121-130`; `hy_memory/pipelines/writer.py:3593-3612`; `hy_memory/pipelines/_retrieval/dedup.py:415-453`; `hy_memory/pipelines/writer.py:1636-1677`.
- links: `causal`→EK-12; `causal`→EK-19; `contrast`↔EK-28.

### EK-06 Canonical isolation is a three-part double-colon key
- category: PERMISSION / MODEL
- layer: L1
- epistemic: Fact
- statement: `MemoryNode.build_isolation_key(uid, agent_id, session_id)` returns `uid::agent_id::session_id`; empty session becomes `default_session`. Topic scope deliberately drops the session (`uid::agent_id`). `parse_isolation_key` accepts 2-, 3- and 4-part keys, so old app-scoped data remains readable.
- evidence: `hy_memory/models/memory.py:357-405`.
- links: `subsystem`↔EK-07; `constraint`→EK-24; `contrast`↔EK-35.

### EK-07 History uses a separate two-level key format
- category: AUDIT / PERMISSION
- layer: L1
- epistemic: Fact
- statement: `HyMemoryClient._build_isolation_key` returns `"{user_id}:{agent_id}"` (single colon, no session). This key is used for `HistoryStore.record_add/update/delete/search` and `get_recent`. The canonical node key is double-colon and includes session. The divergence is documented as “history two-level,” but it means a history lookup cannot address a single session.
- evidence: `hy_memory/client.py:852-859`; call sites `:3253`, `:4613`, `:5140`, `:5458`, `:8149`; `hy_memory/pipelines/base.py:59-63` has the same two-level shape but is unused.
- links: `contrast`↔EK-06; `dependency`→EK-33; `constraint`→EK-36.

### EK-08 `source` is orthogonal to agent identity and is fail-closed for mixed aggregates
- category: PERMISSION / DATA
- layer: L2
- epistemic: Fact
- statement: `source` (e.g. `openclaw`, `cursor`, `codex`) is normalized to lowercase and stored separately from `agent_id`. `normalize_source_filters` adds `mixed` to `exclude` whenever any include/exclude filter is active and the caller did not explicitly mention `mixed`. A composite/derived memory that merges multiple sources therefore cannot bypass a source blacklist.
- evidence: `hy_memory/utils/source_filter.py:normalize_source_filters`; `hy_memory/client.py:4114-4120`; `hy_memory/pipelines/writer.py` source merge calls.
- links: `constraint`→EK-29; `causal`→EK-31; `contrast`↔EK-35.

## B. Write and reconcile

### EK-09 Write routing has a zero-LLM fast path and an LLM classification path
- category: CONTROL / ROUTING
- layer: L2
- epistemic: Fact
- statement: If `scene` is explicit, no classifier runs. If the input has no tool messages, the write goes straight to `chat` even when `scene` is None. Only a tool-bearing write with `scene=None` invokes `classify_messages_scene`. A classifier exception is caught and falls back to `chat`; the caller is not told.
- evidence: `hy_memory/client.py:3010` `_async_add_impl` routing block; `hy_memory/coding/judge.py`.
- links: `mechanism`↔EK-10; `contrast`↔EK-21; `constraint`→EK-12.

### EK-10 L1 is written before extraction, so raw data survives a later LLM failure
- category: MEMORY / FAILURE
- layer: L1
- epistemic: Fact
- statement: `MemoryWriter.write` computes the embedding, constructs a `MemoryNode` with `ACTIVE`, `is_latest=True`, `source_type=EXPLICIT`, and upserts L1 before the agent/extractor runs. The raw node is provisional but durable; a later failure does not erase the conversation automatically.
- evidence: `hy_memory/pipelines/writer.py:2570-2700`; `:123` `MemoryWriter`.
- links: `causal`→EK-11; `causal`→EK-12; `contrast`↔EK-36.

### EK-11 Extraction receives recent raw history, not just the current message
- category: MEMORY / CONTEXT
- layer: L1
- epistemic: Fact
- statement: Before extraction, `_get_recent_history` reads L1_RAW nodes for the same user/agent, excludes the current L1 ID, sorts by `gmt_created`, and reconstructs messages. The default is `history_turns*2` messages (5 turns → 10 messages). This is the only internal path allowed to read raw layers.
- evidence: `hy_memory/pipelines/writer.py:2977-3115`; call site `:2739`; `allow_raw_layer=True` in `list_by_user`.
- links: `dependency`→EK-26; `causal`→EK-12; `contrast`↔EK-27.

### EK-12 Empty extraction is success, and it shadows L1 rather than deleting it
- category: MEMORY / STATE
- layer: L2
- epistemic: Fact
- statement: After a successful agent run, the L1 raw node is set to `SHADOW` whenever its layer is L1, regardless of how many memories were extracted. The code comment explicitly says an empty `memory=[]` is a normal conclusion (“nothing to remember”), and the earlier `stored_ids` guard caused raw dialogue to remain ACTIVE and recallable. Failures return earlier, so failed raw nodes stay ACTIVE for retry.
- evidence: `hy_memory/pipelines/writer.py:3589-3615`.
- links: `mechanism`↔EK-05; `causal`→EK-13; `contrast`↔EK-25.

### EK-13 Full-agent failure deletes L1 only when deletion is confirmed
- category: FAILURE / MEMORY
- layer: L2
- epistemic: Fact
- statement: `_rollback_failed_l1` calls `vector_store.delete`, then re-reads by ID to confirm absence. It records `l1_rollback.deleted`, `retry_safe`, and `partial_agent_persist`. If deletion is unconfirmed or any higher-level nodes were written, it returns 409 and says automatic retry is unsafe. It deliberately does not delete L2 changes, because reconcile may have updated pre-existing nodes.
- evidence: `hy_memory/pipelines/writer.py:3118-3175`.
- links: `causal`→EK-14; `constraint`→EK-15; `contrast`↔EK-20.

### EK-14 Reconcile failure stores zero new memories; there is no ADD-all fallback
- category: FAILURE / CONSISTENCY
- layer: L2
- epistemic: Fact
- statement: When `MemoryReconciler.reconcile` returns `success=False`, `_reconcile_and_store_locked` returns an empty stored list plus a `ReconcileFailure` and zero operation counts. The comment states that no new memory is stored and the caller should rerun; episode state has not advanced, so rerun does not double-count.
- evidence: `hy_memory/pipelines/writer.py:1237-1310`; `hy_memory/agent/reconciler.py:397`.
- links: `causal`→EK-15; `contrast`↔EK-19; `constraint`→EK-20.

### EK-15 Reconcile serialization is per-user but loop-local
- category: CONCURRENCY / CONSISTENCY
- layer: L2
- epistemic: Fact
- statement: `_reconcile_user_lock(user_id)` is a per-user `asyncio.Lock` held across the full reconcile+store block. It prevents concurrent same-user first-writes from both seeing an empty L2/L4 and taking the ADD fast path. Locks are kept per event loop in a `WeakKeyDictionary`; there is no cross-process or cross-loop lock.
- evidence: `hy_memory/pipelines/writer.py:_reconcile_user_lock`; `:1237-1245`.
- links: `mechanism`↔EK-14; `constraint`→EK-16; `contrast`↔EK-31.

### EK-16 Candidate retrieval is user-scoped and can include other agents
- category: RETRIEVAL / PERMISSION
- layer: L2
- epistemic: Fact
- statement: Reconcile candidates are retrieved at the `user_id` level, not narrowed to the current `agent_id`; the code comment says the candidate pool may mix other agents and dilute the current agent’s candidates, and logs `cross_agent_count`. This is intentional because user-owned memories can be shared across agents.
- evidence: `hy_memory/agent/reconciler.py` candidate search block; `RECONCILE_CANDIDATES` log fields.
- links: `constraint`→EK-06; `causal`→EK-17; `contrast`↔EK-24.

### EK-17 Reconcile candidates collapse evolution chains to chain heads
- category: RETRIEVAL / STATE
- layer: L2
- epistemic: Fact
- statement: `trace_full_chains` walks `supersedes`/`superseded_by` in both directions, deduplicates by chain head, and exposes only the chain head’s ID and content to the reconcile prompt. The prompt does not receive `history_versions`, preventing the LLM from operating on nodes already replaced. The old versions remain stored for audit.
- evidence: `hy_memory/agent/reconciler.py` chain expansion; `hy_memory/pipelines/writer.py` chain persistence.
- links: `mechanism`↔EK-16; `causal`→EK-18; `contrast`↔EK-19.

### EK-18 Canonical-state nodes bypass ordinary reconcile
- category: STATE / EXCEPTION
- layer: L1
- epistemic: Fact
- statement: Nodes with `custom.memory_kind == "canonical_state"` are excluded from candidate matching and are managed by System 2 via an exact key lifecycle. Normal reconcile never sees them, so a direct write cannot supersede or merge a canonical state through the usual ADD/SUPERSEDE path.
- evidence: `hy_memory/agent/reconciler.py` canonical-state skip; `hy_memory/pipelines/system2_tools.py` state tools; `hy_memory/pipelines/writer.py` canonical branch.
- links: `constraint`→EK-17; `subsystem`↔EK-44; `contrast`↔EK-19.

### EK-19 Dedup is logical deletion by SHADOW, including whole evolution chains
- category: STATE / DEDUP
- layer: L1
- epistemic: Fact
- statement: Dedup plans keep one item and mark losers `is_latest=False`, `status=SHADOW`, `shadow_reason=dedup:<trigger>`, `dedup_kept_id=<id>`. If a chain head is the loser, its `chain_node_ids` are shadowed too. The change uses `update_payload`, not `upsert`, to avoid rewriting vectors. The code comment explains that the previous hard delete could erase an entire evolution chain irreversibly.
- evidence: `hy_memory/pipelines/_retrieval/dedup.py:415-453`; `execute_dedup` `:456-533`.
- links: `mechanism`↔EK-05; `causal`→EK-27; `contrast`↔EK-18.

### EK-20 Finalize failure is a partial-success contract, not a rollback
- category: FAILURE / STATE
- layer: L2
- epistemic: Fact
- statement: When `finalize_session` is true and dynamic/series episode finalization fails, the writer returns 409 `DYNAMIC_EPISODE_FINALIZE_FAILED` with `partial_agent_persist=True`. The comment states base memories are already persisted, so retrying the whole block would repeat reconcile and is unsafe. Session episode finalization is idempotent but dynamic/series are not treated as such.
- evidence: `hy_memory/pipelines/writer.py` finalize block and response error mapping; `hy_memory/client.py:1823-1845`.
- links: `causal`→EK-13; `constraint`→EK-14; `contrast`↔EK-45.

## C. Read, scope and recall

### EK-21 Search routing is three-tier and lite is a hard override
- category: CONTROL / ROUTING
- layer: L2
- epistemic: Fact
- statement: Explicit `scene` bypasses the LLM. A single query with `search_mode=default` is fixed `normal`/chat and makes no LLM call. `search_mode=agentic` runs one LLM analysis for coding/normal classification, query rewrite and time parsing. `lite` forces raw-only normal search and ignores `scene`, `search_mode` and `layers`.
- evidence: `hy_memory/client.py:4024-4250` routing branches; `hy_memory/coding/judge.py:analyze_queries`.
- links: `contrast`↔EK-02; `causal`→EK-22; `constraint`→EK-23.

### EK-22 `scene="all"` fuses lanes with RRF, not score normalization
- category: RETRIEVAL / FUSION
- layer: L1
- epistemic: Fact
- statement: `_search_all_scenes` runs chat/working/coding concurrently and fuses by rank: `score = 1/(60+rank)`, rank from 1. Ties are broken by lane order chat→working→coding then lane rank. Items are not merged across namespaces by ID. A lane failure is collected in `errors` and does not fail the whole search.
- evidence: `hy_memory/client.py:3830-3875` `_search_all_scenes`, `_rrf_fuse_lanes`.
- links: `mechanism`↔EK-21; `contrast`↔EK-23; `constraint`→EK-34.

### EK-23 Reader scoring is additive dense+BM25+tag, with a documentation/code weight drift
- category: RETRIEVAL / SCORING
- layer: L2
- epistemic: Fact
- statement: The reader computes `final = sem + 1[bm25>0] * W_BM25 * bm25_norm + 1[tag_cos>0] * W_TAG * tag_cos`. Code defaults are `W_BM25=0.35`, `W_TAG=0.3`, tag threshold `0.5`. The module docstring says `w_bm25=0.2`, and an inline comment still says `0.2·bm25`. Operators tuning from the header will misread the actual default.
- evidence: `hy_memory/pipelines/reader_base.py:1-20` vs `:1156-1159`; `:1194` comment.
- links: `constraint`→EK-24; `contrast`↔EK-22; `dependency`→EK-29.

### EK-24 Isolation-parameter construction is combinatorial and refuses an ambiguous combination
- category: PERMISSION / RETRIEVAL
- layer: L2
- epistemic: Fact
- statement: `_build_isolation_params` returns:
  - no agent/session filters and `request.agent_id` present → exact `isolation_key = user::agent`;
  - `agent_ids` without sessions → `user_ids + agent_ids` filters;
  - session IDs → `isolation_keys` for all user×agent×session combinations;
  - `request.agent_id` empty with no agent/session filters → `user_ids` only (all agents of that user);
  - multiple agents + multiple sessions → an `error_msg` and refusal.
  The refusal exists because the cross-product would otherwise be ambiguous and could over-broaden scope.
- evidence: `hy_memory/pipelines/reader_base.py:3643` `_build_isolation_params`; `hy_memory/client.py:4125-4133` public validation.
- links: `mechanism`↔EK-06; `constraint`→EK-25; `contrast`↔EK-16.

### EK-25 The public SDK requires `user_ids`, but the vector store does not
- category: PERMISSION / BOUNDARY
- layer: L2
- epistemic: Fact
- statement: `_async_search_impl` raises `ValueError("user_ids is required and cannot be empty")`. At the storage layer, `_build_search_filter` adds a user condition only if `isolation_keys`, `isolation_key`, or `user_id/user_ids` is present; if all are absent it adds no user filter and searches the whole collection subject only to layer/status/latest filters. The security boundary therefore lives in the caller, not the store.
- evidence: `hy_memory/client.py:4125-4129`; `hy_memory/data/vector_store_qdrant.py:1195-1245`; `hy_memory/data/vector_store_base.py`.
- links: `constraint`→EK-24; `contrast`↔EK-35; `causal`→EK-36.

### EK-26 L1_RAW is excluded from all ordinary reads unless explicitly requested
- category: MEMORY / PRIVACY
- layer: L1
- epistemic: Fact
- statement: `list_by_user` defaults `allow_raw_layer=False`; vector search excludes L1 unless `layers` explicitly includes it. `client.list_memories` comments that “raw does not leave the store” and the old post-filter was removed. Only internal history reconstruction and explicit layer requests can read raw.
- evidence: `hy_memory/data/vector_store_qdrant.py` `allow_raw_layer` branch; `hy_memory/data/vector_store_base.py`; `hy_memory/client.py:2121-2200` list comments; `hy_memory/pipelines/writer.py:3011`.
- links: `mechanism`↔EK-11; `constraint`→EK-27; `contrast`↔EK-33.

### EK-27 Search itself mutates state: dedup shadows L2/L4 losers in the background
- category: STATE / READ-PATH
- layer: L2
- epistemic: Fact
- statement: After a successful search, `_spawn_search_dedup` takes L2_FACT and L4_IDENTITY results, loads their embeddings, groups by owner, and calls `execute_dedup(delete_from_store=True)` on the write loop. The loser is SHADOWed, not deleted. The trigger is `search`, the task is fire-and-forget, and failures are swallowed at debug level.
- evidence: `hy_memory/client.py:4500-4550`, `_spawn_search_dedup` `:3617-3735`; `hy_memory/pipelines/_retrieval/dedup.py:456-533`.
- links: `causal`→EK-19; `contrast`↔EK-28; `constraint`→EK-25.

### EK-28 Intention recall lazily converts expired L7 to L2 and persists it
- category: STATE / READ-PATH
- layer: L2
- epistemic: Fact
- statement: `recall_intentions` over-fetches ACTIVE L7 nodes, checks `valid_until`; if expired (`now > valid_until`) it calls `update_payload` to change `layer` to `l2_fact`, then removes the item from the intention result. The next normal search can see it as historical fact. Duplicate surviving intentions are grouped by user+agent+memory_at day+valid_until and cosine ≥0.95; duplicates are asynchronously deleted from VDB.
- evidence: `hy_memory/pipelines/_retrieval/intention.py:1-35`, `:243` `recall_intentions`; `_partition_duplicates`, `_spawn_async_delete`.
- links: `contrast`↔EK-27; `causal`→EK-05; `constraint`→EK-29.

### EK-29 Access-count writeback is opt-in and fire-and-forget
- category: STATE / READ-PATH
- layer: L1
- epistemic: Fact
- statement: After search, `access_count+1`/`last_accessed_at=now` is scheduled to the write loop only when `recall.strength_enabled` is true (default false) and `access_tracking_enabled` is true (default true). The bump reads the count already returned by the reader to avoid re-reading. Failures are silent.
- evidence: `hy_memory/client.py:4552-4580`; `hy_memory/config.py:RecallConfig.strength_enabled`.
- links: `dependency`→EK-23; `constraint`→EK-28; `contrast`↔EK-27.

### EK-30 Graph rerank is strictly additive and fail-open
- category: RETRIEVAL / GRAPH
- layer: L2
- epistemic: Fact
- statement: `_apply_graph_rerank` starts from the caller’s candidate list and returns it unchanged if any step fails. Graph edges are fetched only for node IDs already in the pool, so graph expansion cannot introduce an out-of-pool node; however `get_related_edges` itself does not filter by user. Graph rerank is disabled by default.
- evidence: `hy_memory/pipelines/reader_base.py:379` `_apply_graph_rerank`, `:599-601`; `hy_memory/config.py:GraphStoreConfig.graph_rerank_enabled`.
- links: `constraint`→EK-24; `contrast`↔EK-42; `dependency`→EK-04.

### EK-31 Underflow fill is a second read pass with side channels disabled
- category: RETRIEVAL / FILL
- layer: L1
- epistemic: Fact
- statement: When `MEMORY_HYBRID_V2_FILL_UNDERFLOW` is enabled (default 0) and normal results are below `limit`, `read` reuses the query vector for one more `_read_once` with `schema_limit=0`, `intention_limit=0`, no side channel, and `fill_normal_slots` only fills empty slots. Timeout/failure is recorded as `underflow_fill` and does not alter the primary result.
- evidence: `hy_memory/pipelines/reader_base.py:1057-1110` fill block; env default.
- links: `mechanism`↔EK-23; `constraint`→EK-30; `contrast`↔EK-22.

### EK-32 Multi-user episode expansion only considers the first user
- category: RETRIEVAL / LIMITATION
- layer: L2
- epistemic: Fact
- statement: `_episode_candidates` calls `list_by_episode` with `request.user_id` or `request.user_ids[0]`. A search spanning users A and B therefore expands episode candidates only for A. The code comments call this an omission, not a leak: it reduces recall for B, it does not return A’s episodes to B.
- evidence: `hy_memory/pipelines/reader_base.py:656` `_episode_candidates`.
- links: `constraint`→EK-24; `contrast`↔EK-25; `causal`→EK-46.

## D. Storage, audit and privacy

### EK-33 History is default-on and stores full old/new text
- category: AUDIT / PRIVACY
- layer: L1
- epistemic: Fact
- statement: `HistoryConfig.enable` defaults true and `db_path` defaults to `<data>/data/history.db`. The schema stores `event` (ADD/UPDATE/DELETE/SEARCH), `old_memory`, `new_memory`, `layer`, `isolation_key`, `actor_id`, `extra`. ADD stores the request content; UPDATE stores old and new; DELETE stores the old content again. `record_searches` defaults false, so search history is opt-in.
- evidence: `hy_memory/config.py:1140-1160`; `hy_memory/data/history_store.py:1-130`; `hy_memory/client.py:3253-3270`, `:5450-5465`.
- links: `dependency`→EK-07; `causal`→EK-34; `contrast`↔EK-26.

### EK-34 Deleting a memory writes its full text into the audit database
- category: DELETION / PRIVACY
- layer: L2
- epistemic: Fact
- statement: `_delete_chat` reads the node’s content before deletion, deletes VDB and graph data, then calls `history_store.record_delete(memory_id, content=old_content, isolation_key=...)` when the VDB deletion succeeded. The user-visible memory disappears from recall, but a full copy remains in `history.db`. `delete_all` does not write per-item history records.
- evidence: `hy_memory/client.py:5350-5465`; `hy_memory/data/history_store.py:record_delete`.
- links: `mechanism`↔EK-33; `contrast`↔EK-35; `constraint`→EK-36.

### EK-35 Single-record get/update/delete are ID-only, not owner-scoped
- category: PERMISSION / API
- layer: L2
- epistemic: Fact
- statement: `async_get(memory_id)`, `async_update(memory_id, content)`, and `async_delete(memory_id)` accept no `user_id`/`agent_id`. `get` resolves the ID across VDB, working/coding stores and graph; ambiguous namespace hits fail closed unless `kind`/`variant` is supplied. Delete reads the node, repairs the evolution chain, removes VDB/graph copies, and cleans tags. The security boundary is the unguessability of the UUID plus the caller’s authorization, not an owner check in the SDK.
- evidence: `hy_memory/client.py:2016-2100`, `:4794-4840`, `:4949-5000`, `:5256-5465`.
- links: `contrast`↔EK-25; `constraint`→EK-34; `dependency`→EK-06.

### EK-36 `delete_all` enumerates agent×session combinations and has no history cleanup
- category: DELETION / SCOPE
- layer: L1
- epistemic: Fact
- statement: `async_delete_all` uses three modes: no filters → one all-user round; agent IDs only → one round per agent; sessions → rounds for each agent×session (defaulting agent to `default_agent` when only sessions are given). Each round deletes VDB rows, retrieval clauses and graph data, then writes a `DELETE_ALL_VDB`/`DELETE_ALL_GRAPH` op log. It does not call `history_store.record_delete`, so prior `history.db` and `pipeline_logs` copies remain.
- evidence: `hy_memory/client.py:5581-5720`.
- links: `contrast`↔EK-34; `constraint`→EK-07; `causal`→EK-35.

### EK-37 Coding business rows and vector rows are explicitly not transactional
- category: STORAGE / CONSISTENCY
- layer: L2
- epistemic: Fact
- statement: `coding/store.py` file header states: “业务仓储保存经验、事实和轨迹，VDB 保存检索向量；写入先生成 embedding，再写业务仓储，最后写向量。两层不是同一个事务，数据库切换不会自动获得跨存储原子性。” Retrieval therefore trusts the business store, not the vector payload copy.
- evidence: `hy_memory/coding/store.py:1-25`.
- links: `subsystem`↔EK-38; `constraint`→EK-39; `contrast`↔EK-04.

### EK-38 Coding scope filtering over-fetches vectors then filters by business IDs
- category: RETRIEVAL / SCOPE
- layer: L2
- epistemic: Fact
- statement: `search_by_query_embedding` first recalls by `user_id`, then if an agent/session/workspace scope is supplied, obtains `scoped_experience_ids` from the business store, computes `_scope_overfetch` from selectivity, and filters vector hits by `custom.coding_experience_id`. An empty allowed set returns `[]`. The comment says this is deliberate because vector payload may hold a stale workspace after a move.
- evidence: `hy_memory/coding/store.py:577` `search_by_query_embedding`, `_scope_overfetch`.
- links: `mechanism`↔EK-37; `contrast`↔EK-24; `causal`→EK-39.

### EK-39 The coding diff chain is a decorator, not a retriever, and is zero-LLM
- category: MEMORY / CODING
- layer: L2
- epistemic: Fact
- statement: `HyCodeMem` wraps the coding writer. `observe()` stores each real patch as a thread/round; thread key degrades from hunk-header function name to line range to file; owner is `"<user_id>|<workspace_id>"`. `seq` is the observe call count, not a diff comparison. `chain_block_enabled` defaults false and `HCM_BLOCK_OFF=1` disables decoration. No LLM is called in this layer.
- evidence: `hy_memory/coding/chain_store.py:160` `HyCodeMem`, `:180` `observe`, `:504` `decorate`, `:910` `recall`.
- links: `subsystem`↔EK-37; `constraint`→EK-38; `contrast`↔EK-45.

### EK-40 L0 basic profile is user-level, not agent-level
- category: MEMORY / IDENTITY
- layer: L1
- epistemic: Fact
- statement: `USER_LEVEL_AGENT_ID=""` and `USER_LEVEL_SESSION_ID=""`; `basic_info_node_id` is `uuid5(namespace, "basic_info::<user_id>")`, so one profile node exists per user regardless of agent. Each update stores only diff fields, creates a new version, marks the old superseded, and links via `supersedes`/`superseded_by`. `get_user_profile`/`get_user_basic_info` query L0 by user only.
- evidence: `hy_memory/agent/tools/basic_profile.py:45-80`, `:337-440`.
- links: `contrast`↔EK-06; `causal`→EK-17; `dependency`→EK-26.

### EK-41 Vector filter priority is isolation_keys → isolation_key → user_ids → no user condition
- category: STORAGE / PERMISSION
- layer: L2
- epistemic: Fact
- statement: `_build_search_filter` adds `isolation_keys IN` when present; otherwise exact `isolation_key`; otherwise `user_id`/`user_ids` plus optional `agent_ids`; otherwise no user condition. `include_sources` adds must-match conditions; `exclude_sources` adds must-not. Defaults are `only_latest=True` and `status=ACTIVE`. `created_after` uses `memory_at` OR `gmt_created`.
- evidence: `hy_memory/data/vector_store_qdrant.py:1195-1250`; `hy_memory/data/vector_store_base.py`.
- links: `mechanism`↔EK-25; `constraint`→EK-24; `subsystem`↔EK-42.

### EK-42 Graph mutation tools do not share one authorization standard
- category: PERMISSION / GRAPH
- layer: L2
- epistemic: Fact
- statement: `add_evidence` checks that the target schema and each VDB evidence node match the current `user_id`/`agent_id`. `revise_schema` and `retire_schema` check the old schema’s user/agent scope. But `create_graph_node` fetches evidence by ID without checking its owner, and `add_edge` calls `graph_store.add_edge(source_id, target_id, ...)` without any scope check. `create_graph_node` can therefore create a DERIVED_FROM edge to a foreign evidence ID if one is supplied. Reachability through the public prompt is not demonstrated; this is a defense-in-depth gap, not a proven cross-tenant exploit.
- evidence: `hy_memory/pipelines/system2_tools.py:599-790` create; `:915-974` add_evidence; `:990-998` add_edge; `:790-915` revise/retire.
- links: `constraint`→EK-04; `contrast`↔EK-30; `causal`→EK-43.

### EK-43 Graph tag expansion interpolates `isolation_key` into Cypher text
- category: SECURITY / GRAPH
- layer: L1
- epistemic: Observation
- statement: `expand_with_tags` builds a Cypher query with `AND m.isolation_key = '{isolation_key}'` by string interpolation. `isolation_key` is normally constructed from user/agent/session identifiers, so a caller-controlled identifier containing a quote could alter the query. The reader does not call `expand_with_tags` anywhere in this clone; production reachability is unconfirmed. This stays an Observation/Candidate, not a confirmed vulnerability.
- evidence: `hy_memory/data/graph_store_kuzu.py:2307` `expand_with_tags`; grep for call sites.
- links: `contrast`↔EK-42; `constraint`→EK-06; `dependency`→EK-04.

## E. Embedding and migration

### EK-44 Write-path embedding is a per-endpoint batch queue; read-path embedding bypasses it
- category: PERFORMANCE / EMBEDDING
- layer: L2
- epistemic: Fact
- statement: `embed_queued_with_meta` checks cache first, then appends to a per-`(base_url, api_key)` bucket. It flushes when the bucket reaches `queue_batch_size` (default 64) or after `queue_batch_window_ms` (default 100 ms). The lock covers only enqueue/detach; the HTTP call runs outside the lock. One shared request serves all waiters. Search uses `embed()` directly, not the queue.
- evidence: `hy_memory/core/embed_service.py:473-610`; `:684` `embed`.
- links: `mechanism`↔EK-45; `contrast`↔EK-48; `dependency`→EK-49.

### EK-45 Batch finalization has an explicit no-hanging-future invariant
- category: FAILURE / CONCURRENCY
- layer: L2
- epistemic: Fact
- statement: `_send_batch` deduplicates identical texts, calls the provider once, and checks `len(unique_vectors) == len(unique_texts)` before fan-out. A mismatch raises rather than zipping/truncating, because truncation would leave extra futures unresolved forever. The `finally` block sets an exception on any future still unresolved. The result meta records batch size, unique count, queue depth, queue wait, lock wait and flush time.
- evidence: `hy_memory/core/embed_service.py:610-684`.
- links: `mechanism`↔EK-44; `causal`→EK-46; `contrast`↔EK-14.

### EK-46 The queue timer identity is cleared before HTTP to avoid a lost wakeup
- category: CONCURRENCY / BUG-HISTORY
- layer: L2
- epistemic: Fact
- statement: `_delayed_flush` detaches the batch under the lock and, before issuing HTTP, clears `_queue_flush_tasks[bucket]` if the current task owns it. The comment explains the historical bug: a new request arriving during the old batch’s HTTP would see an unfinished timer and not start another, so a sub-64 batch could wait indefinitely until traffic filled it. The fix is a code-level invariant, not a timeout.
- evidence: `hy_memory/core/embed_service.py:558-578` and comment at `:570-575`.
- links: `causal`→EK-44; `contrast`↔EK-45; `mechanism`↔EK-47.

### EK-47 Embedding cache is text-only MD5, but model and dimensions are fixed per service
- category: CACHE / EMBEDDING
- layer: L1
- epistemic: Fact
- statement: The cache key is `md5(text)` only; it does not include model, dimensions, endpoint or headers. This is safe only because (a) each `EmbedService` has one configured model/dimension, and (b) per-call `embed_override` is documented and implemented as endpoint/auth/header/timeout only; it explicitly does not override `model`, `embedding_dims` or `extra_body`. A future override that changes model/dims without changing cache key would return incompatible vectors.
- evidence: `hy_memory/core/embed_service.py:120-160`, `:372-390`; `embed_endpoint_override` docs.
- links: `constraint`→EK-48; `contrast`↔EK-44; `causal`→EK-49.

### EK-48 Embedding input truncation is deterministic and retries are transient-only
- category: FAILURE / EMBEDDING
- layer: L1
- epistemic: Fact
- statement: All embedding entry points pass through a token-length guard because `text-embedding-3-small` has an 8192-token hard limit and an over-limit input returns a deterministic 400 that retry cannot fix. Retry policy covers timeout/connection errors with `max_retries=2`, `retry_delay=0.3`, `timeout=15s`; a batch-size mismatch is not retried.
- evidence: `hy_memory/core/embed_service.py:1-120` truncation header; `_embed_openai` retry block.
- links: `mechanism`↔EK-45; `constraint`→EK-47; `contrast`↔EK-49.

### EK-49 Re-embedding changes vector space, not text; source stores are frozen by caller contract
- category: MIGRATION / EMBEDDING
- layer: L2
- epistemic: Fact
- statement: `reembed.py` defines an embedding space by model name + width. It rebuilds embeddable text from stored documents/metadata, not from original conversations. It never writes to the source: vector sources are read-only; the graph source is opened read-only. The caller must pause every writer and keep them paused until the switch commits. Vector re-embeds skip existing target IDs (resumable); graph re-embeds build a staging path, wipe it on start, and move it into place only after verification. Source and target must be the same provider; Neo4j/Memgraph cannot hold two embedding widths side-by-side.
- evidence: `hy_memory/reembed.py:1-40`, `space_identity`, `VectorSpace`, `ReembedPlan`.
- links: `subsystem`↔EK-37; `constraint`→EK-48; `contrast`↔EK-47.

## F. System 2 and graph authority

### EK-50 System 2 is an explicit, non-idempotent, unqueued cognitive pass
- category: CONTROL / SYSTEM2
- layer: L2
- epistemic: Fact
- statement: `evolve()` is the only trigger. `write` does not enqueue or auto-trigger it; the SDK keeps no in-process System 2 queue. One call runs the full current VDB scan, optional cluster preprocessing, one LLM operation pass, and (if the cycle is complete) the cross-domain sweeper. The docstring states restart/multi-pod queue problems are the reason for caller-controlled triggering.
- evidence: `hy_memory/pipelines/system2_writer.py:1-50`, `:1172-1265`.
- links: `contrast`↔EK-51; `causal`→EK-53; `constraint`→EK-54.

### EK-51 The “agent” is a single LLM JSON call, not a tool-calling loop
- category: CONTROL / SYSTEM2
- layer: L1
- epistemic: Fact
- statement: `run_system2_agent` builds one system+user message, calls `complete_messages` once, parses a JSON operation array, and executes operations through the tool executor. `max_iterations=10` is kept “for backward compat, unused in single-call mode.” A parse failure returns 502 `invalid_operations_json`; there is no retry inside the agent. There is no multi-turn tool call loop.
- evidence: `hy_memory/pipelines/system2_agent.py:1938-2025`.
- links: `contrast`↔EK-50; `causal`→EK-52; `constraint`→EK-53.

### EK-52 System 2 tool count is stated three different ways
- category: DOCUMENTATION / SYSTEM2
- layer: L1
- epistemic: Fact
- statement: `system2_tools.py` header says “10 个工具” and enumerates 4 read + 6 write (including `update_graph_node`, `delete_graph_node`). `system2_agent.py` header says “8 个 tools.” The actual `TOOLS` schema contains 9: `search_vdb`, `search_graph`, `get_node`, `expand_node`, `create_graph_node`, `add_evidence`, `revise_schema`, `retire_schema`, `add_edge`. There is no `update_graph_node` or `delete_graph_node` schema. Tool-count documentation cannot be trusted for capability assessment.
- evidence: `hy_memory/pipelines/system2_tools.py:1-20`, `:217-410`; `hy_memory/pipelines/system2_agent.py:1-15`.
- links: `contrast`↔EK-42; `constraint`→EK-51; `causal`→EK-54.

### EK-53 A partial System 2 cycle defers the cross-domain sweeper
- category: STATE / SYSTEM2
- layer: L2
- epistemic: Fact
- statement: `evolve` checks `results["should_continue"]`; if true, it sets `cross_domain_sweeper = {"skipped": True, "reason": "evoling_cycle_partial"}` instead of running the sweeper. The comment explains that running the sweeper after a partial batch would make its output depend on where the batch budget happened to end. `should_continue`, `cycle_status`, `remaining_clusters`, `remaining_fresh_facts` and `batch_failures` are returned to the caller.
- evidence: `hy_memory/pipelines/system2_writer.py:1205-1265`.
- links: `mechanism`↔EK-50; `constraint`→EK-51; `contrast`↔EK-20.

### EK-54 Graph evidence and edge authority is split between tool-level checks and store-level calls
- category: AUTHORITY / GRAPH
- layer: L2
- epistemic: Fact
- statement: `create_graph_node` writes the new node with `user_id`/`agent_id` from the executor, then for each evidence ID calls `ensure_vdbref` and `add_derived_from` without checking the evidence node’s owner. `add_evidence` does check both schema and evidence scope. `add_edge` does not check either endpoint. `revise_schema`/`retire_schema` check only the old schema. The result is an asymmetric authority model: some graph operations are scoped, some trust the ID.
- evidence: `hy_memory/pipelines/system2_tools.py:599-790`, `:915-998`, `:790-915`.
- links: `mechanism`↔EK-42; `constraint`→EK-06; `contrast`↔EK-30.

### EK-55 System 2 evidence IDs can be unresolved and still become edges
- category: DATA / GRAPH
- layer: L1
- epistemic: Fact
- statement: In `create_graph_node`, `evidence_nodes[eid] = await vector_store.get_by_id(eid)` stores `None` when the ID does not exist. The loop still calls `ensure_vdbref(eid, "unknown")` and `add_derived_from(node_id, eid)`. `add_evidence`, by contrast, skips `None` and foreign-scope nodes. Thus a hallucinated evidence ID can create a dangling VdbRef/DERIVED_FROM pair through create, but not through add.
- evidence: `hy_memory/pipelines/system2_tools.py:630-640`, `:710-735`; `:937-943`.
- links: `causal`→EK-54; `contrast`↔EK-42; `mechanism`↔EK-51.

## G. Factory, packaging and tests

### EK-56 The component factory shares one writer and one embed service across modes
- category: DEPENDENCY / FACTORY
- layer: L1
- epistemic: Fact
- statement: `ComponentFactory.get_writer()` always constructs `MemoryWriter`; mode-specific behavior is passed as `agent_mode` per request. `get_reader(reader_name)` caches one reader per effective name and shares the same `EmbedService`, `VectorStoreBase`, graph store and cache. `get_system2_writer()` constructs a `System2Writer` that internally reuses `MemoryWriter`. `close()` closes all cached components.
- evidence: `hy_memory/pipelines/registry.py:get_writer`, `get_reader`, `get_system2_writer`, `close`.
- links: `subsystem`↔EK-02; `constraint`→EK-57; `dependency`→EK-44.

### EK-57 Reader selection temporarily mutates a process-global environment variable
- category: CONCURRENCY / CONFIG
- layer: L2
- epistemic: Hypothesis
- statement: `get_reader` resolves the name, sets `os.environ["HY_MEMORY_READER"] = effective_name`, constructs `MemoryReader`, awaits `initialize()`, then restores the previous value in `finally`. `MemoryReader.__init__` reads the env synchronously, so the constructor is safe; but the `await initialize()` keeps the variable set across a suspension point. A concurrent `get_reader` with no explicit override can resolve to the temporary value and build the wrong reader. This is a plausible race, not yet reproduced; no test covers concurrent reader construction.
- evidence: `hy_memory/pipelines/registry.py:121-171`; `hy_memory/pipelines/reader.py:35-80`; `hy_memory/pipelines/_retrieval/config.py:resolve_reader_name`.
- links: `contrast`↔EK-56; `constraint`→EK-02; `causal`→EK-58.

### EK-58 Version metadata and README release row disagree
- category: PACKAGING / DOCUMENTATION
- layer: L1
- epistemic: Fact
- statement: `pyproject.toml` declares `version = "2.0.17"`. The README’s latest release row (2026-11) says `hy-memory==2.0.16` with MemHub `1.0.7`. No tag or CHANGELOG entry was checked to decide whether 2.0.17 is unreleased, a hotfix, or a packaging bump. The commit subject mentions MemHub 1.0.8 build 49, which is not the README’s 1.0.7 row. Version identity is therefore ambiguous at this commit.
- evidence: `pyproject.toml:7`; `README.md:36-37`; commit subject `d8138b5`.
- links: `contrast`↔EK-59; `constraint`→EK-60; `dependency`→EK-49.

### EK-59 The README links a CMB paper but does not publish the numeric scores used by the radar summary
- category: EVIDENCE / BENCHMARK
- layer: L1
- epistemic: Observation
- statement: A repository-wide grep for `LoCoMo`, `MemoryArena`, `CodeMemoryBench`, `72.6`, `55.31`, `64.2` found no README passage with those scores. The README does link a CMB paper PDF and a test comment cites a 77/216 → 45/216 coding-memory ablation, but the specific LoCoMo 72.6 / MemoryArena 55.31 / CodeMemoryBench 64.2/35.6 figures in KnowlegeMap’s secondary summary are not traceable in this clone. They must remain unverified.
- evidence: `README.md:42-58`; `tests/coding/test_scene_coding_variant.py:13,320`; grep result.
- links: `contrast`↔EK-58; `constraint`→EK-60; `causal`→EK-61.

### EK-60 The always-on test marker expression excludes the Neo4j isolation suite
- category: TEST / COVERAGE
- layer: L2
- epistemic: Fact
- statement: `pyproject.toml` sets default `-m 'not integration'`; `tests/test_vector_search_isolation.py` marks both test classes `@pytest.mark.integration` and requires a live Neo4j fixture (`neo4j://localhost:7687`, user `neo4j`, password `password`). The fixture is real, not missing, but the suite never runs in the default test command. The repository therefore has no always-on cross-user vector isolation test.
- evidence: `pyproject.toml` pytest config; `tests/test_vector_search_isolation.py:24-30`, `:361-388`.
- links: `constraint`→EK-25; `contrast`↔EK-61; `causal`→EK-62.

### EK-61 The tested subset proves reconcile/failure semantics, not graph authority or concurrency
- category: TEST / EVIDENCE
- layer: L1
- epistemic: Fact
- statement: The 40-test subset that passed covers reconcile failure, candidate user scope, chain merge, evolution-chain shadow filtering, and L1 raw blocking. It does not exercise `create_graph_node` evidence scope, `add_edge` scope, concurrent `get_reader`, history retention, or multi-process reconcile. Those remain code-read findings, not behaviorally validated ones.
- evidence: this run's pytest output `40 passed in 20.07s`; file names `tests/test_reconcile_*`, `test_evolution_chain_shadow_filter.py`, `test_block_l1_raw_in_read_filter.py`.
- links: `contrast`↔EK-60; `constraint`→EK-42; `causal`→EK-57.

### EK-62 The full non-integration test run was inconclusive at teardown
- category: TEST / PROCESS
- layer: L1
- epistemic: Fact
- statement: A background `pytest -q -m 'not integration' --timeout=120` run ended with `OSError: [Errno 24] Too many open files` in pytest’s `cleanup_dead_symlinks` on `/tmp/pytest-of-user/pytest-4`. The captured output did not include a usable pass/fail summary. No full-suite pass is claimed. The subset result is reported separately.
- evidence: background task `task-1` output.
- links: `contrast`↔EK-61; `constraint`→EK-60; `mechanism`↔EK-55.

### EK-63 Test execution polluted the target clone with ignored caches
- category: PROCESS / READ-ONLY
- layer: L1
- epistemic: Fact
- statement: Running pytest inside the target clone created `.pytest_cache/` and `__pycache__/` directories. `git status --porcelain` stayed empty because they are ignored. The caches were removed with `git clean -fdX`; no tracked file was modified. This is a formal violation of “target repository read-only” at the filesystem level, even though the Git tree remained clean.
- evidence: `git clean -ndX` output; `git status --porcelain` = 0 after cleanup.
- links: `constraint`→EK-62; `contrast`↔EK-61; `causal`→EK-60.

## H. Cross-cutting constraints

### EK-64 There is no global “memory committed” transaction
- category: CONSISTENCY / ARCHITECTURE
- layer: L3
- epistemic: Fact
- statement: The write path spans up to five stores: VDB, graph, business DB, history DB, pipeline-log DB. Each has independent success/failure and independent ack semantics. `coding/store.py` explicitly says business and vector are not one transaction; the writer’s graph projection is best-effort; history/trace writes are guarded and warning-only. “Write succeeded” means the primary store committed, not that all derived stores agree.
- evidence: `hy_memory/coding/store.py:1-25`; `hy_memory/pipelines/writer.py` graph/history paths; `hy_memory/client.py` history writes; `hy_memory/core/embed_service.py` separate HTTP.
- links: `subsystem`↔EK-37; `subsystem`↔EK-33; `causal`→EK-13; `contrast`↔EK-14.

### EK-65 The system’s scope model is layered, but its authorization model is not uniform
- category: PERMISSION / MODEL
- layer: L3
- epistemic: Pattern
- statement: The data model has a clean three-level isolation key, but enforcement is distributed: public search requires `user_ids`; vector store can search unscoped if called directly; get/update/delete are ID-only; graph tools differ by operation; history/trace have their own key. There is no single policy object or middleware that all entry points pass through.
- evidence: EK-06, EK-07, EK-24, EK-25, EK-35, EK-42, EK-54, EK-60.
- links: `constraint`→EK-24; `contrast`↔EK-42; `causal`→EK-25; `subsystem`↔EK-07.

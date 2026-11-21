# 01 Project Layer — Hy-Memory 2.0

- run_id: `ARCH-2026-11-20-001`
- source: `hy_memory/` at `d8138b501a755f662a024f7e8eb879613eabf1d3`
- scope: Python engine + SDK, with `memhub/` and `plugins/` only as integration boundaries

## 1. What the project is

Hy-Memory is a dual-system agent memory engine. The README frames it as a production-grade memory SDK with three scenes (`chat`, `working`, `coding`) and three processing depths (`lite`, `pro`, `ultra`). The code confirms a layered design rather than a single vector table:

- a **vector store** for L0–L4 (`l0_basic_info`, `l1_raw`, `l2_fact`, `l3_summary`, `l4_identity`);
- a **graph store** for L5–L7 (`l5_knowledge`, `l6_schema`, `l7_intention`) in `ultra` (and optionally `pro`);
- a **business relational store** for `working` and `coding` facts/experiences;
- a **diff chain** for coding revisions;
- **history** and **pipeline trace** stores for audit/debug.

The stated differentiator is not “store chats and retrieve top-k.” It is a write-time cognitive pipeline: extract atomic facts, reconcile them against existing memory with ADD/SUPERSEDE/UPDATE, maintain evolution chains, and in `ultra` run a separate System 2 pass that induces L6 schemas from clustered L2/L4 evidence.

## 2. Module responsibility map

| Module | Responsibility | Key entry |
|---|---|---|
| `hy_memory/client.py` | public SDK, routing, loop bridge, orchestration, public views | `HyMemoryClient` |
| `hy_memory/config.py` | env/default configuration for every backend and feature flag | `MemoryConfig` |
| `hy_memory/models/memory.py` | unified node schema, enums, isolation keys, evolution chain fields | `MemoryNode` |
| `hy_memory/pipelines/registry.py` | component singleton factory and shared dependencies | `ComponentFactory` |
| `hy_memory/pipelines/writer.py` | L1 write, extraction, reconcile, persistence, episode/graph side paths | `MemoryWriter` |
| `hy_memory/agent/reconciler.py` | candidate retrieval + LLM ADD/SUPERSEDE/UPDATE decisions | `MemoryReconciler` |
| `hy_memory/pipelines/reader*.py` | multi-signal recall and result shaping | `MemoryReader`, `BaseReadPipeline` |
| `hy_memory/pipelines/system2_*` | explicit cognitive pass: clusters → single LLM JSON ops → graph tools | `System2Writer`, `run_system2_agent` |
| `hy_memory/core/embed_service.py` | embedding batching, cache, retry, endpoint override | `EmbedService` |
| `hy_memory/data/*` | vector, graph, history, cache, business stores | backends |
| `hy_memory/coding/*` | working/coding experience/fact storage and code diff chain | `CodingMemoryStore`, `HyCodeMem` |
| `hy_memory/agent/episode_summary.py` | session/dynamic/series episode narratives | finalize functions |
| `hy_memory/reembed.py` | embedding-space migration | `ReembedPlan` |
| `hy_memory/inspector.py`, `snapshot.py`, `runtime.py` | inspection, backup, shared runtime | auxiliary surfaces |

## 3. Core abstractions

### 3.1 `MemoryNode`

One dataclass represents every layer. The fields encode:

- identity: `node_id`, `user_id`, `agent_id`, `session_id`, `source`;
- ownership: `owner` is `user` or `agent`, only for L2 and L7;
- content: `content` is the only field embedded;
- time: `memory_at`, `valid_from`, `valid_until`, `gmt_created`, `gmt_modified`;
- evolution: `supersedes`, `superseded_by`, `is_latest`;
- epistemic metadata: `status`, `confidence`, `source_type`, `evidence_count`, `evidence_chain`;
- access metadata: `access_count`, `last_accessed_at`;
- extension: `custom`, `tags`, `tag_ners`.

This is a “one schema, many layers” design. It avoids separate tables per memory type but forces every consumer to branch on `layer` and `status`.

### 3.2 Layer taxonomy

Eight concrete layers exist, despite the docstring calling the model “七层”:

`L0_BASIC_INFO`, `L1_RAW`, `L2_FACT`, `L3_SUMMARY`, `L4_IDENTITY`, `L5_KNOWLEDGE`, `L6_SCHEMA`, `L7_INTENTION`.

Storage boundary in the docstring: L0–L4 in VDB, L5–L7 in Graph. `L5` is described as “暂未实现” in the enum comment, while the graph code has node types and tool support for L6/L7.

The enum also contains aliases for three schema generations (`l6_identity → l4_identity`, `l4_knowledge → l5_knowledge`, `l5_schema → l6_schema`, plus V1 names `profile`, `dialogue`, `summary`, `knowledge`, `raw`). `from_string` checks the legacy map before the value loop, so stored payloads from older versions are readable.

### 3.3 Status lifecycle

`ACTIVE` → normal recall.
`SUPERSEDED` → old version on an evolution chain, retained but not latest.
`NEGATED` → explicitly negated.
`CONFLICTED` → conflicting versions coexist.
`SHADOW` → logical deletion or low-confidence inference; excluded from recall.
`ARCHIVED` → archived.

`SHADOW` is used for at least four distinct operations: failed/empty extraction downgrade of L1, reconcile `UPDATE` old-node retirement, dedup loser retirement, and low-confidence inference. One status name therefore covers several causal stories; callers cannot tell why a node was shadowed unless `shadow_reason`/`custom` is present.

### 3.4 Isolation keys

Canonical key: `"{user_id}::{agent_id}::{session_id}"`, with empty session replaced by `default_session` (`MemoryNode.build_isolation_key`).

Topic scope: `"{user_id}::{agent_id}"` (no session), so topics accumulate across sessions.

History keys use a different two-level form, `"{user_id}:{agent_id}"` (single colon), because history is an audit log rather than a recall store. This is an intentional difference, not a typo, but it means three key formats coexist: canonical node key, topic key, history key.

## 4. Write pipeline (chat scene)

The synchronous path is:

1. normalize request and validate `content`/`user_id`;
2. assign layer: explicit `extra.layer`, otherwise `L1_RAW`;
3. compute embedding through the write-path batch queue;
4. create a `MemoryNode` with `ACTIVE`, `is_latest=True`, `source_type=EXPLICIT`;
5. upsert L1 into the vector store;
6. in `lite`, stop here;
7. in `pro`/`ultra`, fetch recent L1 history for prompt context, run the extractor;
8. if extraction fails or returns no items, roll back L1 or shadow it (see below);
9. reconcile each extracted item against candidate memories;
10. persist ADD/SUPERSEDE/UPDATE results, update chains, optionally project to graph, optionally index tags/entities, optionally summarize episodes;
11. after agent success, downgrade the L1 raw node to `SHADOW`;
12. return a response with memory IDs, counts, timings and failure metadata.

Key semantics:

- **L1 is provisional.** It is written first so raw conversation survives a later extraction failure, but on success it is shadowed: “nothing to remember” is treated as a valid conclusion, so raw dialogue does not remain recallable. On extraction failure, the code deletes L1 if it can confirm deletion; if deletion is unconfirmed or higher-level nodes were partially written, it returns 409 and marks automatic retry unsafe.
- **Reconcile is not best-effort.** If the reconcile LLM fails, no new memory is stored; it does not fall back to ADD-all. This is a deliberate correctness choice: a retry can converge, whereas silently adding duplicates cannot.
- **Per-user serialization is process-local.** `_reconcile_user_lock` is keyed by user and event loop, not a cross-process lock. Multi-process deployments can still race.
- **Empty extraction shadows L1 but does not delete it.** The code comment explains the previous bug: an empty `stored_ids` condition left raw active and recallable.

## 5. Read pipeline

`search` has three routing modes:

- `scene` explicit → no LLM, direct route;
- `search_mode="default"` + single query → fixed normal/chat, no LLM;
- `search_mode="agentic"` or explicit multi-query rewrite enabled → LLM scene classification + query rewrite + time parsing.

`lite` ignores `scene`, `search_mode` and `layers` and forces raw-only normal search.

The reader fuses:

- dense vector similarity (VDB semantic);
- keyword/BM25 (Qdrant text index or rank-bm25; legacy/hybrid reader variants);
- tag cosine bridge;
- graph schema/intention side channels (when enabled);
- episode candidates;
- optional local reranker;
- optional underflow fill.

The default score is additive, not RRF: `sem + 1[bm25>0] * W_BM25 * bm25_norm + 1[tag>0] * W_TAG * tag_cos`, default `W_BM25=0.35`, `W_TAG=0.3`, tag cosine threshold `0.5`. The file header still says `w_bm25=0.2`, so documentation and code drift.

`scene="all"` runs chat/working/coding concurrently and fuses with RRF `1/(60+rank)`, tie order chat → working → coding.

Results are grouped into `schema`, `proactive` and `normal`, and L0 is excluded from search (returned by `get_user_profile` instead).

## 6. Working and coding scenes

`working` and `coding` share a two-tier schema:

- one `WorkingFact` per session: statement, subject, files, details;
- zero or more `Experience`: task, search_keys, solution, boundary_envs, confidence, with a value threshold;
- bidirectional ID links between fact and experience; superseded facts summarized by `fact_chain_summary`.

`coding` differs by:

- software-engineering extraction prompt;
- separate business SQLite DB (`coding_memory_cv.db`) and `*_cv` vector collections;
- a diff chain (`HyCodeMem`) that stores each real patch as a thread/round; thread key degrades from hunk header function name → line range → file; owner is `"<user_id>|<workspace_id>"`;
- zero LLM calls in the diff chain; `chain_block_enabled` defaults false (opt-in).

The coding store explicitly states that business rows and vector rows are **not one transaction**; vector payload may be stale after a workspace move. Retrieval therefore trusts the business store’s `scoped_experience_ids` and over-fetches the vector index by selectivity before filtering.

## 7. System 2 (ultra)

`evolve()` is explicit, synchronous and non-idempotent. It is not queued and not triggered by `write`; the docstring argues that an SDK-owned in-process queue would lose tasks on restart and duplicate work across pods.

The “agent” is a **single LLM call**, not a tool-calling loop:

1. deterministic preprocessing clusters L2/L4 facts (DBSCAN) and fetches graph forward/reverse context;
2. if clusters exist, one prompt asks for a JSON operation array;
3. the JSON is parsed and executed through `System2ToolExecutor`;
4. no clusters → skip the LLM;
5. if the cycle is partial, the cross-domain sweeper is skipped;
6. graph writes are allowed only through this path; `create_graph_node`, `add_evidence`, `revise_schema`, `retire_schema`, `add_edge` are the mutation tools.

The code calls this a “single-call mode”; `max_iterations=10` remains in the signature but is explicitly “unused”. Documentation is inconsistent: `system2_agent.py` says “8 tools”, `system2_tools.py` says “10 tools” and lists 4 read + 6 write, while the actual schema contains 9 (no `update_graph_node`, no `delete_graph_node`).

## 8. Persistence and side paths

- **Vector**: Chroma default, Qdrant optional, FAISS optional; collection `agent_memories`, separate coding collections.
- **Graph**: Kuzu default, Neo4j optional; L6/L7 nodes, `VdbRef` shadow nodes, `DERIVED_FROM` evidence edges, `RELATED_TO` edges, topic tags.
- **Business**: SQLite default, MySQL/Postgres optional; working/coding facts, experiences, chain threads.
- **History**: SQLite `history.db`, default enabled; ADD/UPDATE/DELETE/SEARCH events with full old/new text.
- **Pipeline trace**: SQLite `pipeline_logs_v2`, default enabled; LLM prompt/response raw text plus parsed output and token counts.
- **Episodes**: session/dynamic/series summaries, all opt-in; dynamic/series finalize failure returns 409 with `partial_agent_persist`.

There is no single transaction spanning vector + graph + business. Each subsystem has its own commit/acknowledgement and its own failure semantics. This is the project’s largest structural constraint: “memory written” is not a global invariant, only a per-store one.

## 9. Failure semantics observed

- L1 upsert failure: recorded in `_l1_error`; in `agent_mode=full`, write fails and extraction does not proceed.
- Extractor failure: `_rollback_failed_l1`; if rollback or partial L2 persist cannot be ruled out, 409 + automatic retry unsafe.
- Reconcile failure: zero new memories; no ADD-all fallback; caller may retry.
- Graph projection failure: best-effort side path; write can still return success.
- Coding chain observe failure: warning only; returns 0 rounds; “vectors are already written, failing the whole add is a net loss.”
- Dynamic/series episode finalize failure: 409 with partial persistence; retry may duplicate reconcile.
- Search path: a single scene lane failure is recorded in `errors`, the overall search still returns other lanes.
- Intention expiry: read path lazily mutates L7→L2; failure does not fail search.
- Dedup: search path fire-and-forget shadows losers; failure is silent at debug level.

## 10. Configuration surface

Environment variables dominate defaults. Key families:

- `MEMORY_MODE`, `MEMORY_DATA_DIR`, `MEMORY_ENABLE_GRAPH`;
- `MEMORY_EMBEDDING_*`, embedding provider/model/dims;
- `MEMORY_VECTOR_*`, `MEMORY_QDRANT_*`, `MEMORY_CHROMA_*`;
- `MEMORY_GRAPH_*`, Kuzu/Neo4j paths/URIs;
- `MEMORY_RECALL_*`, strength, episodes, local rerank, underflow;
- `MEMORY_HYBRID_V2_*`, BM25/tag weights;
- `MEMORY_EVOLING_*`, System 2 batch/token/dedup controls;
- `MEMORY_HISTORY_*`, `MEMORY_PIPELINE_LOG_ENABLED`, `MEMORY_PIPELINE_TRACE_ENABLED`;
- `HCM_*`, coding chain controls;
- `HY_MEMORY_READER`, legacy/hybrid reader selection.

Many features default off (strength, episodes, local rerank, underflow fill, schema dedup/compaction, coding chain block). Defaults matter: a stock `pro` deployment is materially narrower than the README’s full feature list.

## 11. Test surface

157 test files cover reconcile failure/chain merge/candidate scope, L1 raw filtering, shadow semantics, vector isolation (integration-only), coding variant, System 2 behavior, config, snapshots, re-embed, etc. The suite is behavior-heavy but backend-dependent: optional `chromadb`/`faiss` absence produces collection errors, and Neo4j isolation tests are excluded by default.

The 40-test subset that ran clean exercises reconcile failure, user-scope candidates, chain merge, evolution-chain shadow filtering, and L1 raw blocking. It does not prove graph write isolation, cross-process concurrency, history retention, or benchmark quality.

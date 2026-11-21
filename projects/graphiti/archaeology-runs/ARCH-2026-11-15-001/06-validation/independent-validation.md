# 06.3 · Independent Validation — Blind Reconstruction

> **Method**: Auditor re-read `ed23196` source (40+ files/excerpts across `graphiti_core/`, `server/`, `mcp_server/`) and built Independent Findings **before** opening `00`–`05`. Comparison was performed only afterwards. The package was not treated as a source of facts.
> **Constraint**: no runtime, no tests, no database. Every finding below is static code evidence.

## 1. Independent Findings (IF)

| ID | Independent finding | Evidence |
|---|---|---|
| IF-01 | EntityEdge carries four time fields: `created_at`, `valid_at`, `invalid_at`, `expired_at` | `edges.py`; EV-029–033 |
| IF-02 | `valid_at` is the event time, anchored to `reference_time`; `created_at` is ingest/knowledge time | `graphiti.py:1455-1475`; EV-009 |
| IF-03 | `invalid_at` is the event-time end; `expired_at` is the system-time end | `edges.py`; `_apply_temporal_invalidation` |
| IF-04 | Invalidation is soft: it writes end times, never deletes the old edge | `edge_operations.py:722-748`; EV-038 |
| IF-05 | A new edge older than a candidate is itself ended at the candidate's `valid_at` | `edge_operations.py:730-745`; EV-035 |
| IF-06 | Equal `valid_at` makes both overlap checks true, so neither edge is invalidated | `edge_operations.py:750-785`; EV-039 |
| IF-07 | Only LLM `contradicted_facts` enter invalidation; prompt forbids recency-based contradiction | `dedupe_edges.py`; EV-043–045 |
| IF-08 | `add_episode` resolves a request-scoped driver/clients bundle | `graphiti.py:1423-1431`; EV-006 |
| IF-09 | `add_episode_bulk` does the same | `graphiti.py:1669-1671` |
| IF-10 | `add_episode` previous window is `RELEVANT_SCHEMA_LIMIT = 10`, not `EPISODE_WINDOW_LEN = 3` | `graphiti.py:1438`; `search_utils.py:64` |
| IF-11 | `previous_episode_uuids` overrides retrieval and bypasses reference-time filtering | `graphiti.py:1443-1445`; `EpisodicNode.get_by_uuids` |
| IF-12 | `store_raw_episode_content=False` clears only `episode.content`; name/source_description/times/edges remain | `graphiti.py:1034`; EV-016 |
| IF-13 | `add_triplet` constructs a synthetic `EpisodicNode` when none is supplied, **with a fresh UUID**, and passes it to `resolve_extracted_edge` | `graphiti.py:2140-2156` |
| IF-14 | `append_episode_provenance` appends `episode.uuid` to the edge, so `add_triplet` persists a **dangling episode UUID** for an episode that is never saved | `edge_operations.py:884-890`; `graphiti.py:2156` |
| IF-15 | `add_triplet` bypasses LLM extraction but **does not** bypass dedupe/invalidation; it calls `resolve_extracted_edge` | `graphiti.py:2156` |
| IF-16 | `add_triplet` uses `self.driver` and `self.clients` throughout, not a request-scoped bundle | `graphiti.py:2048-2171` |
| IF-17 | `remove_episode` uses `self.driver` and `self.llm_client`, no group/driver parameter | `graphiti.py:2174-2251` |
| IF-18 | `get_nodes_and_edges_by_episode` uses `self.driver` | `graphiti.py:2020-2029` |
| IF-19 | `Graphiti.search` accepts an optional `driver` but does not auto-resolve `group_ids` to databases | `graphiti.py:1913-1962` |
| IF-20 | `build_communities` accepts an optional `driver`, but defaults to `self.clients.driver` | `graphiti.py:1867-1888` |
| IF-21 | Neo4j `execute_write` has real `begin/commit/rollback`; FalkorDB/Kuzu/Neptune directly await the function | `neo4j_driver.py:149-165`; others `execute_write` |
| IF-22 | `add_nodes_and_edges_bulk` opens one session and one `execute_write`; all nodes/edges/episodes go through it | `bulk_utils.py:128-150` |
| IF-23 | `add_episode_bulk` phase 1 writes episodes; phase 2 extracts and writes nodes/edges | `graphiti.py:1696-1776` |
| IF-24 | `build_communities` removes communities before building | `graphiti.py:1874-1888` |
| IF-25 | Default search passes an empty `SearchFilters()`; no validity filter | `graphiti.py:1943-1952`; `search_filters.py` |
| IF-26 | Context string facts contain only fact + dates; episodes contain only source_description + content; no UUIDs | `search_helpers.py` |
| IF-27 | REST server has no auth middleware; `SECURITY.md` states no authn/authz/caller isolation | `server/graph_service/main.py`; `SECURITY.md` |
| IF-28 | MCP queue is `dict[str, asyncio.Queue]`; failure is logged and `task_done()` called; no retry/dead-letter | `queue_service.py:49-78` |
| IF-29 | `max_coroutines` creates a Semaphore per `semaphore_gather` call, not a global limit | `helpers.py:132-142` |
| IF-30 | `remove_episode` deletes edges with no remaining episode support, disassociates supported edges, deletes nodes with `MENTIONS` count 1 | `graphiti.py:2178-2213` |
| IF-31 | `remove_episode` does not roll back `invalid_at`/`expired_at` on edges it once invalidated | `graphiti.py:2174-2251` |
| IF-32 | Node summary rebuild catches exceptions, restores old summary, logs, does not raise | `graphiti.py:2215-2244` |
| IF-33 | `remove_episode` summary rebuild uses `self.llm_client`, not the request-scoped `clients.llm_client` | `graphiti.py:2229` |
| IF-34 | `add_triplet` writes via `add_nodes_and_edges_bulk(self.driver, ...)`, so a non-default group edge lands in the default database | `graphiti.py:2171` |
| IF-35 | `get_nodes_and_edges_by_episode` and `remove_episode` cannot see episodes in a cloned database | IF-17/18 + `_resolve_request_scope` |
| IF-36 | Tests exist but were not executed; `S4/S5 = 0/0` | `tests/` |

## 2. Comparison Against Package

### 2.1 Factual claims

| # | Package claim | Independent verdict | Correction / note |
|---|---|---|---|
| 1 | Four temporal fields | CONFIRMED | — |
| 2 | `valid_at` anchored to `reference_time` | CONFIRMED | — |
| 3 | `created_at` is ingest time | CONFIRMED | — |
| 4 | `invalid_at` is event-time end | CONFIRMED | — |
| 5 | `expired_at` is system-time end | PARTIALLY_CONFIRMED | A historical edge with explicit `invalid_at` gets `expired_at=now` at ingest; semantics need a use-case definition |
| 6 | Soft invalidation, no delete | CONFIRMED | — |
| 7 | Equal `valid_at` leaves both current | CONFIRMED (logic) | Runtime unverified |
| 8 | Backdated new edge is ended at candidate's `valid_at` | CONFIRMED | — |
| 9 | Only LLM contradictions enter invalidation | CONFIRMED | — |
| 10 | Prompt forbids recency-based contradiction | CONFIRMED | — |
| 11 | `add_episode` is request-scoped | CONFIRMED | — |
| 12 | “Request scope covers operations” (broad reading) | PARTIALLY_CONFIRMED | Only `add_episode`/`add_episode_bulk`; `add_triplet`/`remove_episode`/`get_nodes_and_edges_by_episode`/default search are not |
| 13 | Non-default `group_id` clones driver/database | CONFIRMED | — |
| 14 | Previous context is 10 episodes | CONFIRMED | `RELEVANT_SCHEMA_LIMIT`; `EPISODE_WINDOW_LEN=3` is the `retrieve_episodes` default |
| 15 | `previous_episode_uuids` overrides retrieval | CONFIRMED | Also bypasses `valid_at <= reference_time` |
| 16 | Raw content stored by default | CONFIRMED | — |
| 17 | `store_raw=False` clears only content | CONFIRMED | — |
| 18 | `add_triplet` edge has **empty** `episodes` | **CONTRADICTED** | It has one **dangling synthetic UUID**, because `append_episode_provenance` appends the unsaved episode's UUID |
| 19 | `add_triplet` bypasses dedupe/invalidation | **CONTRADICTED** | It bypasses extraction only; it calls `resolve_extracted_edge` |
| 20 | `entity_edges` / `MENTIONS` are provenance | CONFIRMED | — |
| 21 | `remove_episode` deletes unsupported edges | CONFIRMED | — |
| 22 | `remove_episode` does not restore invalidation | CONFIRMED | — |
| 23 | `remove_episode` is request-scoped | **MISSING** | Uses `self.driver`; non-default DB will fail or target wrong DB |
| 24 | Bulk phase 1 writes episodes first | CONFIRMED | — |
| 25 | Bulk phase-2 failure can leave orphan episodes | PARTIALLY_CONFIRMED | Static path supports it; no fault injection |
| 26 | Neo4j has real transactions | CONFIRMED | — |
| 27 | FalkorDB/Kuzu/Neptune have no transaction wrapper | CONFIRMED | `execute_write` directly awaits |
| 28 | Community rebuild removes then builds | CONFIRMED | — |
| 29 | Community failure can leave an empty layer | PARTIALLY_CONFIRMED | Static path supports it; no failure injection |
| 30 | Default search does not filter invalid edges | CONFIRMED | `SearchFilters()` |
| 31 | Context has no edge/episode UUID | CONFIRMED | — |
| 32 | Reference server has no authn/authz | CONFIRMED | — |
| 33 | Same `group_id` = same trust | CONFIRMED | `SECURITY.md` |
| 34 | MCP queue is in-process memory | CONFIRMED | — |
| 35 | No retry/dead-letter | CONFIRMED | log + `task_done()` |
| 36 | `max_coroutines` is per-call, not global | CONFIRMED | — |
| 37 | `add_triplet` respects request scope | **MISSING** | Uses `self.driver`/`self.clients` |
| 38 | `get_nodes_and_edges_by_episode` respects request scope | **MISSING** | Uses `self.driver` |
| 39 | `search` auto-resolves group to database | **MISSING** | Caller must pass `driver`; otherwise default DB |
| 40 | Summary rebuild uses request-scoped LLM | **MISSING** | Uses `self.llm_client` |

### 2.2 Abstraction claims

| # | KO / model | Verdict | Reason |
|---|---|---|---|
| A1 | KO-01 “validity is an interval, not row existence” | CONFIRMED | Supported by four fields + soft invalidation |
| A2 | KO-02 “model proposes, code disposes” | **DOWNGRADED** | The separation exists, but code has an undefined equal-time branch and the LLM gate is a single point of failure; keep as L3, not L4 guarantee |
| A3 | KO-03 “retention and validity independent” | PARTIALLY_CONFIRMED | Storage does retain; but default read is audit-like, and the package must not imply current-only reads |
| A4 | KO-04 “isolation requires enforcement” | CONFIRMED | No auth in reference server; `group_id` is a partition key |
| A5 | KO-05 “ack durability ≤ queue durability” | PARTIALLY_CONFIRMED | Structure supports it; no restart experiment |
| A6 | KO-06 “transaction boundary is storage property” | CONFIRMED | Driver implementations |
| A7 | KO-07 “derived edge is not evidence unless anchored” | **CONTRADICTED** | Package overstates: `add_triplet` writes a dangling UUID, and context omits all UUIDs; the invariant is weaker than stated |
| A8 | KO-08 “concurrency safety starts with state ownership” | **OVER_GENERALIZED** | State ownership is applied to two ingestion APIs only; several public APIs still use `self.driver` |

## 3. Verdict Statistics

| Status | Count |
|---|---|
| CONFIRMED | 32 |
| PARTIALLY_CONFIRMED | 6 |
| DOWNGRADED | 1 |
| OVER_GENERALIZED | 1 |
| MISSING | 5 |
| CONTRADICTED | 3 |
| NEEDS_HUMAN_REVIEW | 5 (separate issues, below) |
| **Total checked** | **48** |

- Factual rows: 40.
- Abstraction rows: 8.
- No claim was accepted merely because it “looked plausible”; every confirmation has a file/symbol anchor.

## 4. Three Successful Validations

1. **Four-axis temporal model** — package correctly identified `created_at`/`valid_at`/`invalid_at`/`expired_at` and their distinct roles. Independent read of `edges.py` and `_apply_temporal_invalidation` agrees.
2. **Driver transaction asymmetry** — package correctly rejected “atomic write for all drivers”. Neo4j has real commit/rollback; FalkorDB/Kuzu/Neptune directly execute.
3. **No authorization boundary** — package correctly refused to treat `group_id` as security. Code has no auth middleware; `SECURITY.md` explicitly externalizes it.

## 5. Three Errors Found

1. **`add_triplet` does not produce an empty provenance list.** It produces a one-element list containing the UUID of a synthetic episode that is never persisted. This is worse than “no anchor”: it looks like provenance but cannot be resolved. Fix: either persist the synthetic episode, or mark the edge `derived_without_source` and do not append an unsaved UUID.
2. **“Request scope” is not system-wide.** Only `add_episode` and `add_episode_bulk` resolve a request-scoped driver/clients. `add_triplet`, `remove_episode`, `get_nodes_and_edges_by_episode`, and default `search` use `self.driver`/`self.clients`. In a multi-database `group_id` deployment, writes/reads/deletes can target the wrong database.
3. **`remove_episode` has no database routing.** It takes only `episode_uuid` and uses `self.driver`; a non-default group's episode cannot be found. This also means the “request-scoped state ownership” model is not a property of the public API surface.

## 6. Omissions (5)

| ID | Omitted fact | Why it matters |
|---|---|---|
| O-01 | `add_triplet` writes a dangling synthetic episode UUID | Breaks the claimed evidence invariant |
| O-02 | `add_triplet` uses default driver/clients | Cross-database write misrouting |
| O-03 | `remove_episode` uses default driver | Cross-database delete failure/misrouting |
| O-04 | `get_nodes_and_edges_by_episode` uses default driver | Cross-database read failure |
| O-05 | `search` does not map `group_ids` to databases | Retrieval from non-default group silently returns default DB results |

Additional minor omission: `remove_episode` summary rebuild uses `self.llm_client` rather than the request-scoped `clients.llm_client`, so per-request LLM routing does not apply on deletion.

## 7. Needs Human Review

| ID | Issue | Why code cannot decide |
|---|---|---|
| HR-01 | Equal `valid_at` tie-break | Product semantics: which fact wins at the same instant? |
| HR-02 | Default read mode (`current` vs `audit`) | Product semantics: accuracy vs audit |
| HR-03 | `add_triplet` provenance contract | API contract: persist synthetic episode vs mark source-less |
| HR-04 | Multi-database routing contract | Deployment contract: is `group_id` a database name? If yes, all APIs must route |
| HR-05 | `remove_episode` rollback semantics | Lifecycle contract: delete data or undo influence |

## 8. Benchmark Case (for skill CI)

**Case name**: `graphiti-bitemporal-memory-v1`
**Input**: repository at `ed23196506eba8806fae6da831d73fcd2ad714d8`.
**Expected findings (must be found)**:

| # | Expected finding | Weight |
|---|---|---|
| B1 | Four time fields and their semantics | 10 |
| B2 | Soft invalidation, no delete | 10 |
| B3 | Equal `valid_at` no invalidation | 8 |
| B4 | LLM only nominates; code applies time arithmetic | 8 |
| B5 | Default search does not filter invalid edges | 8 |
| B6 | Context exposes dates but not provenance UUIDs | 10 |
| B7 | `add_triplet` synthetic episode → dangling UUID | 12 |
| B8 | Request scope only on add_episode/bulk | 12 |
| B9 | Driver-dependent transaction semantics | 10 |
| B10 | No auth in reference server; `group_id` is partition only | 12 |

**Negative expectations (must NOT claim)**:
- “Graphiti guarantees current-only recall.”
- “`group_id` is an authorization boundary.”
- “All drivers are transactional.”
- “`add_triplet` edges have no provenance field.”
- “MCP ack is durable.”

**Scoring**: 100 points; pass at ≥80 and zero negative claims. A run that finds B1–B6 but misses B7–B8 should score ≤72 and be marked `PARTIAL`.

## 9. Auditor Verdict

`PARTIALLY_CONFIRMED` — the package's core architecture is correct, but three specific claims must be corrected before corpus commit:
1. `add_triplet` dangling synthetic UUID (not empty).
2. Request scope is partial, not system-wide.
3. `remove_episode`/`get_nodes_and_edges_by_episode`/default `search` do not route by group database.

No runtime evidence was produced; `S4/S5` remain 0.

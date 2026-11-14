# 06.2 · Coverage Gaps and Non-Claims

## 1. What Is Covered

| Area | Evidence type | Confidence |
|---|---|---|
| Facade/control flow | Static code | High for path existence, low for runtime behavior |
| Temporal field semantics | Field descriptions + arithmetic code | High for code semantics, no runtime |
| Edge dedupe/contradiction | Prompt + resolver | High for code path, no model behavior |
| Search recipes | Config recipes + search entry | High for configuration, no ranking test |
| Driver transaction semantics | Driver implementations | High for code, no backend test |
| MCP queue | Queue service | High for structure, no process test |
| Security boundary | Code absence + SECURITY.md | High for reference server; deployment unknown |

## 2. What Is Not Covered

### 2.1 Runtime
- No database started.
- No episode ingested.
- No LLM called.
- No search executed.
- No MCP/REST server started.
- No failure injection.

### 2.2 Historical
- Shallow clone: no commit history before `ed23196`.
- No ADR archaeology; design intent from README/SECURITY only.
- No issue/PR history (except issue #1676 referenced in code comment).

### 2.3 External
- Zep hosted service (proprietary) not examined.
- Deployment-specific auth gateways not examined.
- Client integrations not examined.

### 2.4 Data Volume
- No scale/performance test.
- No embedding dimension/recall test.
- No multi-tenant load test.

## 3. Claims Explicitly Not Made

1. “Graphiti always returns current facts.” — **Not claimed**; default filter is empty.
2. “Graphiti is secure.” — **Not claimed**; reference server has no auth.
3. “All drivers are transactional.” — **Not claimed**; only Neo4j has commit/rollback.
4. “MCP is reliable.” — **Not claimed**; in-memory queue, no retry.
5. “Graphiti is better than Mem0/MemMachine.” — **Not claimed**; no comparative benchmark.
6. “These patterns are universal.” — **Not claimed**; Cross-project validation pending.
7. “Tests pass.” — **Not claimed**; no tests executed.

## 4. Gaps That Independent Validation Should Attack

| Gap ID | Attack | Expected failure mode |
|---|---|---|
| G-01 | Equal `valid_at` contradiction | Both current; no tie-break |
| G-02 | Invalid facts in top-k | Old high-similarity fact dominates |
| G-03 | Provenance absent in LLM context | Model cannot cite source |
| G-04 | `add_triplet` no episode | Edge with empty `episodes` |
| G-05 | MCP restart | Accepted task lost |
| G-06 | Non-Neo4j partial write | Partial graph state |
| G-07 | `remove_episode` after invalidation | Old fact stays invalid |
| G-08 | Community rebuild failure | Community layer emptied |
| G-09 | Bulk phase-2 failure | Orphan episodes |
| G-10 | Concurrent same-group ingestion | Non-deterministic invalidation |
| G-11 | `previous_episode_uuids` future injection | Past context contaminated |
| G-12 | `group_id` database collision | Write to unintended DB |

## 5. Minimal Reproduction Suggestions (for later runtime work)

1. **Temporal unit test**: construct two edges with equal `valid_at`, call `resolve_edge_contradictions`, assert neither invalid.
2. **Search test**: 20 near-duplicate old facts + 1 new fact, run default search, inspect top-k.
3. **Context test**: call `search_results_to_context_string`, assert no UUID fields.
4. **Triplet test**: `add_triplet` without episode, read `edge.episodes`.
5. **Queue test**: submit slow job, kill worker, restart, query episode.
6. **Driver test**: in FalkorDB/Kuzu, force second statement failure, inspect partial state.
7. **Removal test**: A invalidates B, delete A, inspect B.
8. **Community test**: mock LLM failure during build.
9. **Bulk test**: mock edge resolver failure after phase-1.
10. **Concurrency test**: 100 parallel contradictory writes to same group.

These are suggestions, not executed work.

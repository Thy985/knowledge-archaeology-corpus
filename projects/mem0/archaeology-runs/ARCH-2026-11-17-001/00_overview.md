# 00 Overview — Mem0 2.2.1 refresh (ARCH-2026-11-17-001)

## One-line positioning

Mem0 is an agent long-term-memory layer in two forms: a hosted memory API (`MemoryClient` → `api.mem0.ai`) and an OSS stack (`mem0ai` Python SDK + `mem0-ts` TypeScript SDK + FastAPI self-hosted server). The OSS core compresses conversation into self-contained facts, stores them in a vector store with SQLite history, and retrieves them with semantic + BM25 + entity signals.

## Why this was a refresh, not a new project

The old run `ARCH-2026-09-21-001` covered 2.1.0 at commit `a39a802`. Between that commit and `f1791daa`, GitHub Compare reports:

- 40 commits, 269 files, +9390 / −3708
- `mem0/client/main.py` +422/−11, `mem0/client/types.py` +14/−25
- `mem0/memory/main.py` +65/−17
- 22 vector-store files changed
- new TS tests: add-filters isolation, entity guards, profiles, missing optional peers, turbopuffer score/unit, valkey
- `docs/core-concepts/memory-operations/delete.mdx` changed its delete contract

This delta lands directly on the active E-layer scope question and F-layer write/delete trust boundary, so a refresh is the correct unit of work for 2026-11-17.

## Repository facts

| Item | Value |
|---|---|
| repository | `https://github.com/mem0ai/mem0.git` |
| commit | `f1791daa86d469249c9b002f62644d58a1eb96a0` |
| branch | `main` |
| version | `2.2.1` (`pyproject.toml`) |
| commit time | `2026-11-06 22:23:06 +0530` |
| tag | `v2.2.1 = 94c3fe9f238f3dbf29c9ce98643bd71eb13077cd` |
| previous baseline | `a39a802bbc93e85b820078cd3c4dbaf53af25dbe` (2.1.0) |
| tracked files | 1844 |
| Python | 421 files / 126,148 LOC |
| TS/TSX | 457 + 136 files / 107,858 LOC |
| tests | 132 `test_*.py`; 174 `*.test.ts` |
| skill | knowledge-archaeology v3.2 |

## Top five findings

1. **2.2.1 is a scope-hardening release, not a retrieval release.** The core delta is the identity/entity guard: blank IDs rejected before HTTP, metadata identity keys stripped on add and constrained on update, entity IDs validated and trimmed, `search`/`get_all` forced to use `filters`. The retrieval algorithm (semantic + BM25 + entity boost, threshold before fusion) is essentially unchanged.

2. **The same "scope" has two grammars across three surfaces and no single authority.** Python OSS `add` accepts only top-level IDs and has no `filters`; Python Platform `add` accepts only top-level IDs and rejects `filters`; TS OSS `add` accepts both top-level IDs and `filters` (test-proven). `app_id` exists in the Platform client constants but not the OSS constants. This is KO-02, an L4 candidate rather than a principle.

3. **Destructive operations do not share an empty-scope rule.** Python OSS and TS OSS `delete_all` raise locally; both Platform `delete_all` clients send an empty request; `delete_users()` with no IDs enumerates and deletes every entity; `reset()` is separate; `"*"` is passed through literally. This is KO-03 / KO-08 and is the highest-risk product inconsistency in the delta.

4. **The self-hosted server authenticates but does not authorize per memory.** `GET/PUT/DELETE /memories/{id}`, history, and scoped list all depend on `verify_auth` only; they never compare the authenticated principal to the memory payload's scope. Only bulk delete and reset use `require_admin`. This is KO-04; it is pre-existing (server code unchanged in this window), but it is the boundary the F-layer question needs.

5. **The update receipt improved to tri-state but still collapses to success.** `base.py` defines `False` (missing), `True` (updated), `None` (unknown); `_update_memory` raises only on `False`. Pinecone always returns `None`; Qdrant keeps an `upsert` fallback on older clients. This is KO-06, with two explicit counterexamples and two runtime hypotheses.

## Three-layer shape

```
Project Layer  (01)   project map, lifecycle, stores, auth, tests
Engineering    (02)   55 EK, 157 links, avg 2.85, six edge types
Knowledge      (03)   10 KO = 8 L3 + 2 L4 candidates; 0 L5
Flow Atlas     (04)   83 edges across 7 classes
Candidates     (05)   15 C + 5 XP + 5 HR
Validation     (06)   self + independent + reconciliation
```

## Quality metrics (this run)

| Metric | Value |
|---|---|
| EK statements (fact/observation) | 55 |
| Engineering Knowledge | 55 |
| EK links | 157 |
| Average EK out-degree | 2.85 |
| Isolated EK | 0 |
| KO | 10 |
| KO aggregation coverage | 10/10 (100%) |
| L3 / L4 / L5 | 8 / 2 / 0 |
| Flow edges | 83 |
| Flow classes | 7/7 |
| Candidates | 15 |
| Cross-project candidates | 5 |
| Human-review items | 5 |
| Counterexamples recorded | 20 |
| EK→KO reverse traces | 10/10 |

## What is genuinely new vs carried

**New / revised in this refresh:**
- client entity guards and `app_id` mismatch
- three SDK scope grammars
- metadata identity protection on both add and update
- Platform `delete_all` empty behavior vs docs
- `delete_users()` all-entity behavior
- Phase 6 `persisted_records` and total-failure raise
- Qdrant `UPDATE_ONLY` + old-client fallback
- Pinecone `None` receipt
- server auth-vs-authorization boundary (re-read, not changed)
- server auth test drift

**Carried from 2.1.0 (still valid, not re-claimed as new):**
- V3 ADD-only pipeline
- hybrid retrieval three signals
- UUID mapping / hash dedup
- OSS telemetry funnel
- governance dual gate

## Validation status (summary; full report in 06)

- Targeted Python tests: 66 + 49 + 8 passed.
- Server auth suite: not green in this environment (56 failed / 18 passed with `JWT_SECRET`; setup errors without it), and the test file encodes 2.1.x assumptions.
- TypeScript tests: not executed (`mem0-ts/node_modules` absent).
- No live vector DB / LLM / embedder / Postgres; no end-to-end HTTP server run.
- Independent auditor: 24 CONFIRMED / 3 PARTIALLY_CONFIRMED / 1 DOWNGRADED / 1 CONTRADICTED / 3 MISSING (details in `06_validation/independent-validation.md`).

## Key disputes left open

1. Is the self-hosted server single-tenant by design (HR-01)?
2. Should `delete_users()` no-arg be an error (HR-02)?
3. Does the `delete.mdx` warning describe client or server behavior (HR-03)?
4. Is the server auth test expected to run with Postgres in CI (HR-04)?
5. Should `app_id` be added to self-hosted bulk delete (HR-05)?

## Reading order

1. `01_project_layer.md` — what the system is and how a memory moves.
2. `02_engineering_knowledge.md` — the evidence-bearing facts.
3. `03_knowledge_layer.md` — the KOs and their aggregation rules.
4. `04_flow_atlas.md` — the seven flows.
5. `05_candidates.md` — what is not settled.
6. `06_validation/` — self-validation, independent blind reconstruction, reconciliation.

## Refresh delta

`refresh-delta.md` maps the old 2.1.0 KO-01…KO-09 and EK-01…EK-36 to this run's KOs/EKs, with an explicit retained / revised / invalidated / new status per old item. This file exists to close the 11-16 benchmark gap (`BC-01`: refresh mode must carry a per-item delta table).

# Independent Validation — Mem0 2.2.1 refresh

- run_id: `ARCH-2026-11-17-001`
- auditor: independent re-read of the repository, not the package as the source of truth
- method: blind reconstruction first (repo map, signatures, tests, diffs), then comparison with `00`–`05`
- prohibition observed: the auditor did not modify the original package; the corrections are in `reconciliation.md`

## 1. Independent reconstruction (before reading the package)

Rebuilt from the repo only:

- two API families: **Platform** (`MemoryClient`, base `https://api.mem0.ai`, paths `/v1/...`, `/v2/...`) and **self-hosted OSS** (`server/main.py`, paths `/memories`, `/search`, `/reset`, no `/v1` prefix).
- Python OSS `Memory`: top-level entity args on `add`; `filters` on `search`/`get_all`; no `filters` on `add`.
- TS OSS `Memory`: top-level `userId/agentId/runId` **and** `filters` (both are accepted; `filters` is copied to avoid caller mutation).
- Python Platform `MemoryClient`: top-level entity args; `_reject_filters` rejects `filters` on `add` and `delete_all`; blank entity guard.
- destructive verbs: `delete(memory_id)`, `delete_all(...)`, `delete_users(...)`, `reset()`; empty-scope behavior differs by surface.
- server auth: `verify_auth` / `require_auth` / `require_admin`; per-memory CRUD depends on `verify_auth` only.
- write pipeline: Phase 6 `persisted_records`; all-fail raises; history/entity links from persisted.
- vector update: tri-state contract; Qdrant `UPDATE_ONLY` + older-client upsert fallback; Pinecone `None`.
- test reality: 66 + 49 + 8 targeted Python tests pass; server auth suite 56 failed / 18 passed; TS tests not run.

## 2. Findings

| ID | Independent finding | Package claim | Status |
|---|---|---|---|
| IF-01 | commit `f1791daa`, version 2.2.1, branch main, 1844 tracked files | same | CONFIRMED |
| IF-02 | Python OSS `add` has no `filters`; `search`/`get_all` require `filters` | EK-06, EK-05 | CONFIRMED |
| IF-03 | TS OSS `add` accepts **both** top-level `userId` and `filters`; `filters` is a second channel, not "the only" channel | EK-06 said "only scope source" | **CONTRADICTED** |
| IF-04 | Python Platform `add` rejects `filters` via `_reject_filters` | EK-06 | CONFIRMED |
| IF-05 | `app_id` in client `ENTITY_PARAMS`, absent from OSS `ENTITY_PARAMS` | EK-08 | CONFIRMED |
| IF-06 | Platform client paths are `/v1/memories`; self-hosted server paths are `/memories`; they are **two API families**, not one client calling one server | EK-17/C-10 framed as "client call to self-hosted route" | **PARTIALLY_CONFIRMED** |
| IF-07 | OSS `delete_all` raises with no filter | EK-11 | CONFIRMED |
| IF-08 | TS OSS `deleteAll` raises with no filter, points to `reset()` | EK-13 | CONFIRMED |
| IF-09 | Python Platform `delete_all()` sends `params={}` | EK-14 | CONFIRMED (mock) |
| IF-10 | `delete.mdx` claims the SDK raises; client code does not | EK-52 | CONFIRMED |
| IF-11 | `delete_users()` no IDs lists all entities and deletes each; test blesses it | EK-15 | CONFIRMED |
| IF-12 | `"*"` passes through literally; self-hosted route has no translation | EK-16 | PARTIALLY_CONFIRMED (no runtime) |
| IF-13 | Phase 6 batch success sets `persisted_records=records`; individual fallback only adds successes | EK-27 | CONFIRMED |
| IF-14 | all inserts fail -> `VectorStoreError` after `save_messages` | EK-29 | CONFIRMED |
| IF-15 | history built from `persisted_records` | EK-28 | CONFIRMED |
| IF-16 | `_delete_memory` does `vector_store.delete` then `db.add_history` | EK-42 | CONFIRMED |
| IF-17 | `base.py` tri-state: False/True/None | EK-35 | CONFIRMED |
| IF-18 | Pinecone `update()` returns None; caller treats as updated | EK-38 | CONFIRMED |
| IF-19 | Qdrant `UPDATE_ONLY` when available, else upsert fallback | EK-37 | CONFIRMED |
| IF-20 | per-memory CRUD uses `verify_auth`, no scope check | EK-20 | CONFIRMED |
| IF-21 | bulk delete/reset use `require_admin` | EK-21 | CONFIRMED |
| IF-22 | `ADMIN_API_KEY=""` is not auth-disabled; `AUTH_DISABLED=true` is | EK-19 | CONFIRMED |
| IF-23 | server auth test suite is stale; 56 failed / 18 passed with `JWT_SECRET`, setup errors without | EK-23 | CONFIRMED |
| IF-24 | `server/` application code unchanged between `a39a802` and `f1791daa` | EK-53 | CONFIRMED |
| IF-25 | `scoring.py` unchanged; retrieval carried | EK-44…EK-48, 00 overview | CONFIRMED |
| IF-26 | 55 EK / 156 links / 2.84 avg at audit time; 157 / 2.85 after reconciliation / 10 KO / 83 Flow | CONFIRMED (script) |
| IF-27 | 10/10 KO have aggregation rules; no L5 | same | CONFIRMED |
| IF-28 | "three SDK scope grammars" overstates: Python OSS and Python Platform are both top-level-only; the real split is **two grammars** (Python top-level vs TS top-level+filters) with three enforcement surfaces | KO-02 | **DOWNGRADED** |
| IF-29 | "no single scope authority" remains defensible, but only if stated as a candidate and not as "three grammars" | KO-02 L4 candidate | PARTIALLY_CONFIRMED |
| IF-30 | **Major omission:** the package never states that Platform `/v1/...` and self-hosted `/memories` are different API families; several EK/C statements become imprecise without this distinction | absent | **MISSING** |
| IF-31 | `Facts / observations retained = 47` is a hand-written metric with no enumeration in the package | 00 metric | **MISSING** |
| IF-32 | `delete_all` route's FastAPI behavior with `app_id` was not runtime-tested; static reasoning is "unknown query ignored" but the path itself differs | C-10 | **MISSING** (runtime evidence) |

## 3. Three strongest confirmations

1. **EK-01 / EK-14 blank-ID guard** — three tests plus code show `""`, spaces, tabs/newlines rejected before HTTP; `delete_all` no-arg is the exception.
2. **EK-27 / EK-28 Phase 6 gating** — the code and two tests agree that rejected records never enter history, entity links, or returned results.
3. **EK-20 per-memory auth-only** — route decorators show `verify_auth` on read/update/delete/history/scoped list, and `require_admin` only on bulk/reset.

## 4. Three strongest errors in the original package

1. **E-01 "TS filters is the only scope source"** — false. `config.userId` is also a source; `filters` is an additional accepted channel.
2. **E-02 "platform client call to self-hosted route"** — unsupported. `MemoryClient` is the Platform client (`/v1/...`); the self-hosted FastAPI serves `/memories` with no `/v1` prefix. The `app_id` parity gap is real, but the framing is wrong.
3. **E-03 "three grammars"** — over-counts. Python OSS and Python Platform both forbid `filters` on `add`; the meaningful split is two grammars plus three enforcement surfaces.

## 5. Omissions

1. **Platform vs self-hosted API family split** (IF-30). This is structural, not a detail.
2. **Facts=47 metric** has no enumeration (IF-31).
3. **Runtime evidence for `app_id`/`"*"`** (IF-32 / C-04). Static only.
4. **`delete_users()` returns "All users, agents, apps and runs deleted."** — the package states the behavior but does not flag the wording as a separate contract signal (minor).
5. **`_delete_memory` writes one history row per memory inside the batch loop**, so `delete_all` produces N rows, not one aggregate event (minor; relevant to KO-09).

## 6. Counterexample attack results

| Target | Attack | Result |
|---|---|---|
| KO-01 scope invariant | direct `Memory` object or direct vector-store write bypasses the invariant | KO-01 already limits to "no accidental scope injection"; **holds with scope caveat** |
| KO-02 no single authority | a shared OpenAPI schema could unify the surfaces | package already labels L4 candidate; **holds as candidate, wording downgraded** |
| KO-03 empty-scope | server may 400 empty request, making Platform client safe | package marks C-01 hypothesis; **holds as inconsistency, not proven exploit** |
| KO-04 authz | single-tenant deployment makes authn sufficient | package C-05/HR-01 already flags; **holds as boundary question, not universal defect** |
| KO-05 receipt | add path is persist-then-history, which is safe | package already conditions on write order; **holds** |
| KO-06 tri-state | pre-read catches missing before update, so `None` is mostly safe | race remains; **holds with conditional wording** |
| KO-09 history projection | add path is not evidence-losing | package says "conditioned on write order"; **holds** |

## 7. Benchmark case

### BC-01 — Refresh delta coverage
- input: old package at `a39a802`, new repo at `f1791daa`
- expected: a per-item old-KO table (9/9) and old-EK table (36/36), each with retained/revised/superseded/not-re-evidenced status and new mapping
- actual: `refresh-delta.md` covers 9/9 KOs and 36/36 EKs; 23 old EKs marked `NR`
- verdict: **PASS (delta coverage)**; **FAIL (full re-evidence)** if the benchmark requires all old EKs re-read. The package must not claim the latter.

### BC-02 — Destructive empty-scope
- input: six deletion surfaces
- expected: one table showing unit / empty-input rule / evidence
- actual: KO-03 + KO-08, EK-11…EK-18, Flow PF-05…PF-09, C-01/C-02/C-04/C-09
- verdict: **PASS** for static coverage; **FAIL** for runtime proof (no live server/vector store).

### BC-03 — Epistemic discipline
- input: all L3+ claims
- expected: no Hypothesis presented as Fact; L4 candidates labeled
- actual: 8 L3 + 2 L4 candidates; C-01/C-04/C-06/C-07/C-11 are hypotheses
- verdict: **PASS with one wording downgrade** (IF-28).

## 8. Verdict summary

| Status | Count |
|---|---|
| CONFIRMED | 24 |
| PARTIALLY_CONFIRMED | 3 |
| DOWNGRADED | 1 |
| OVER_GENERALIZED | 0 |
| CONTRADICTED | 1 |
| MISSING | 3 |
| NEEDS_HUMAN_REVIEW | 0 |

**Overall: PARTIALLY_CONFIRMED.** The evidence base is sound; one statement is contradicted, one is downgraded, one is mis-framed, and one structural fact (Platform vs self-hosted API families) is missing. None of these invalidates the core E/F findings, but they must be corrected in the corpus artifact.

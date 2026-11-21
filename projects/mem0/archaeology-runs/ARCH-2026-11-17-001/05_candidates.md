# 05 Candidates — Mem0 2.2.1

- run_id: `ARCH-2026-11-17-001`
- rule: candidate = something the evidence does not settle. It is not a KO and must not be used as a fact.
- status vocabulary: Observation / Hypothesis / Needs Runtime / Needs Owner / Cross-project Candidate

## C-01 Platform `delete_all()` empty call: does the server reject it?
- status: **Hypothesis / Needs Runtime**
- observation: Python `MemoryClient.delete_all()` with no args sends `DELETE /v1/memories/ params={}`. The client does not raise. The hosted API may return 400; the self-hosted route explicitly returns 400 when no identifier is present.
- evidence: `mem0/client/main.py:575-579`; dynamic mock call; `server/main.py:535-536`.
- missing: no live request to the hosted API; no `TestClient` call against a running self-hosted server.
- verification: boot `server.main` with `AUTH_DISABLED=true` and a fake `Memory`, call `DELETE /memories` with no params; also call `MemoryClient` against a local stub with `base_url` override.
- if true: docs "raises" is still false at the client layer, but the end-to-end effect may still be a refusal.

## C-02 `delete.mdx` says `delete_all` raises, both Platform clients do not
- status: **Fact + Needs Owner**
- observation: the doc contains a breaking-change warning claiming a local raise; Python and TS clients send the request. The same page also correctly says filters are rejected and blank IDs are rejected.
- evidence: `docs/core-concepts/memory-operations/delete.mdx:136-141`; `mem0/client/main.py:554-586`; `mem0-ts/src/client/mem0.ts:569-590`.
- missing: whether the doc refers to a server-side behavior or a planned client behavior.
- verification: git blame the warning line; check PR/issue that introduced it; ask maintainer.
- risk: users relying on the warning will believe an accidental `delete_all()` is locally blocked.

## C-03 Three SDK scope grammars: intentional compatibility or drift?
- status: **Hypothesis / Cross-project Candidate**
- observation: Python OSS `add` has no `filters`; Python Platform `add` rejects `filters`; TS OSS `add` accepts `filters`-only scope and has a test asserting it. `app_id` exists only in client constants.
- evidence: EK-06, EK-07, EK-08.
- missing: design notes, issue/PR discussion, OpenAPI schema.
- verification: search `docs/changelog/sdk.mdx` and PRs around #6796/#6797; compare TS `getAll`/`search` filter handling; check Python 2.2.0 release notes.
- if drift: a shared `Scope` schema would be the remediation. If intentional: docs need a per-SDK scope table.

## C-04 `"*"` on self-hosted server is a literal value, not "all"
- status: **Hypothesis / Needs Runtime**
- observation: the client sends `"*"` unchanged; the self-hosted route forwards it to `Memory.delete_all`, which builds literal filters. Most vector stores will match zero records and return success; some may error.
- evidence: `mem0/client/main.py:555-586`; `server/main.py:528-542`; `mem0/memory/main.py:1917-1928`.
- missing: any live vector store with a `"*"` entity value; provider-specific behavior.
- verification: run Qdrant local with two users, call `delete_all(user_id="*")`, inspect collection count and response.
- risk: "success" response with nothing deleted, or a destructive surprise if a provider treats `*` as a wildcard.

## C-05 Per-memory CRUD has no ownership check: deployment boundary or defect?
- status: **Hypothesis / Needs Owner**
- observation: any authenticated caller can read/update/delete a memory by ID; scoped list accepts any entity ID; only bulk delete/reset are admin-gated.
- evidence: `server/main.py:411-525`; `server/auth.py:144-233`.
- missing: server deployment docs; whether the product assumes a single tenant per instance; whether API keys are intended to be tenant-scoped.
- verification: read `docs/components/oss`/self-hosted docs; ask owner; run two API keys against one server.
- risk: in a multi-tenant self-hosted deployment, this is a cross-tenant read/write/delete path.
- note: this is not a new 2.2.1 regression; `server/` code did not change in the window. It is a pre-existing boundary that the refresh makes more visible.

## C-06 Qdrant older-client fallback can create a deleted point
- status: **Hypothesis / Needs Runtime**
- observation: when `models.UpdateMode` is absent, `update()` omits `update_mode` and calls `upsert`. On a non-local older deployment this can recreate a missing point, violating `base.py` "never create".
- evidence: `mem0/vector_stores/qdrant.py:540-548`; `base.py:33-40`.
- missing: a Qdrant client version without `UpdateMode`; a non-local server; a delete→update race.
- verification: pin `qdrant-client` below the `UpdateMode` introduction, run server mode, delete a point, call update, inspect point count.
- risk: a deleted memory silently returns.

## C-07 Pinecone `None` receipt creates a false UPDATE history row
- status: **Hypothesis / Needs Runtime**
- observation: `_update_memory` does a pre-read, then calls `update`. Pinecone returns `None`; the caller writes `UPDATE` history and returns success. A delete between the read and the update produces a no-op with a success receipt.
- evidence: `mem0/vector_stores/pinecone.py:315-331`; `mem0/memory/main.py:2052-2105`.
- missing: a Pinecone index and a delete/update race.
- verification: Pinecone local mock or real index; delete after `get`, then call `update`; inspect history and point existence.
- risk: stale memory counted as updated; audit trail claims a change that did not happen.

## C-08 Delete writes history after the vector delete, with no transaction
- status: **Fact + Hypothesis about impact**
- observation: `_delete_memory` calls `vector_store.delete` then `db.add_history(..., "DELETE")`. A history failure leaves a deleted memory with no DELETE row.
- evidence: `mem0/memory/main.py:2126-2144`.
- missing: failure injection; whether any caller retries history.
- verification: monkeypatch `db.add_history` to raise after a successful vector delete; inspect history.
- risk: audit gap, not data loss.

## C-09 `delete_users()` no-arg all-entity deletion is intentional but undocumented
- status: **Fact + Needs Owner**
- observation: `delete_users()` with no IDs lists every entity and deletes each. A test explicitly blesses it; `reset()` calls it. The public doc's warning about no-filter deletion is written for `delete_all`, not `delete_users`.
- evidence: `mem0/client/main.py:657-673`; `tests/test_client_entity_guards.py:149-157`.
- missing: product intent; whether `delete_users()` should require an explicit `"*"`.
- verification: check changelog and API reference for `delete_users`.
- risk: an accidental no-arg call is a full tenant wipe.

## C-10 `app_id` exists in the Platform entity vocabulary but not the self-hosted bulk-delete contract
- status: **Fact + Needs Owner**
- observation: the Platform client's `ENTITY_PARAMS` includes `app_id` and `delete_all(app_id=...)` sends it; the self-hosted `DELETE /memories` route has no `app_id` parameter. The two are separate API families (`/v1/...` vs `/memories`), so this is a cross-contract parity gap, not a client calling a mismatched route.
- evidence: `mem0/client/main.py:40-41, 555-586`; `server/main.py:527-542`.
- missing: whether the hosted API supports `app_id`; whether self-hosted intentionally omits apps.
- verification: read the Platform API reference; call the hosted endpoint with `app_id` only; call the self-hosted route with `app_id` only and observe 400.
- risk: API-family parity gap; a client-valid Platform concept has no self-hosted equivalent.

## C-11 `tests/test_server_auth.py` failure: CI red or environment-only?
- status: **Hypothesis / Needs Owner**
- observation: 56 failed / 18 passed with `JWT_SECRET` and no Postgres; failures include stale `ADMIN_API_KEY=""`, missing `JWT_SECRET`, DB connection, and `get_all` mock signature.
- evidence: actual run; `tests/test_server_auth.py`; `server/auth.py:15-16`; `server/main.py:432`.
- missing: CI workflow env (Postgres service, `AUTH_DISABLED`), current CI status for `main`.
- verification: read `.github/workflows/*` for a Postgres service and env; run the suite in the CI image; check GitHub Actions on `f1791daa`.
- risk: if CI is green via a different env, the test file is stale but not red; if CI runs this file as-is, `main` is red.

## C-12 TS OSS `deleteAll` support for `appId`
- status: **Hypothesis**
- observation: the TS OSS `deleteAll` code read shows `userId`/`agentId`/`runId` only; `DeleteAllMemoryOptions` in the platform client may include `appId`. TS OSS may be intentionally app-less.
- evidence: `mem0-ts/src/oss/src/memory/index.ts:1720-1745`; no `appId` in the read range.
- missing: full `DeleteAllMemoryOptions` definition and TS OSS parity tests.
- verification: grep `appId` in `mem0-ts/src/oss/src`, inspect `deleteAll` tests.

## C-13 Client blank guard does not reject internal whitespace
- status: **Fact**
- observation: client `_reject_blank_entity_ids` only checks `.strip()` emptiness; OSS `_validate_and_trim_entity_id` rejects internal whitespace. `"u 1"` passes the client and fails at OSS/server.
- evidence: `mem0/client/main.py:102-109`; `mem0/memory/main.py:205-208`.
- missing: whether the hosted API rejects internal whitespace.
- verification: call the platform API with `user_id="u 1"`; run the OSS path.
- impact: inconsistent validation, not a security bypass.

## C-14 Update "same identity value is silently accepted" can hide replay drift
- status: **Hypothesis**
- observation: Python update silently drops an identity value equal to the current payload and warns only on change. In a replay/migration scenario this can make a stale metadata payload look accepted without changing scope.
- evidence: `mem0/memory/main.py:143-162, 2072-2074`.
- missing: a migration/replay test.
- verification: replay an old metadata dict with the same ID after a scope rename; observe warning absence and final payload.
- impact: low; scope does not change.

## C-15 Profiles API is product surface, not E/F core
- status: **Observation**
- observation: `generate_profile`, profile jobs, and settings are new platform-only methods; they do not touch OSS scope or the write/delete path. They are recorded as EK-51, not promoted to a KO.
- evidence: `mem0/client/main.py:44-48`; `docs/api-reference/profiles/generate-profiles.mdx`, `get-profile-job.mdx`, `get-profile-settings.mdx`, `get-profile.mdx`, `update-profile-settings.mdx`; `tests/test_client_profiles.py`.
- missing: billing/idempotency semantics.
- verification: not needed for this run.

## Cross-project candidates

### XP-01 Surface-local scope authority
- statement: A memory product with multiple SDK entry surfaces tends to develop surface-local scope grammars; the memory object does not own one scope authority.
- evidence here: Mem0 2.2.1 (two grammars across three surfaces + server query grammar).
- comparators: Graphiti (single Python surface), AMG (single object + adapter), Letta (single runtime), Hermes (file memory).
- status: **Cross-project Candidate**, not a Principle.
- test: compare at least two multi-surface memory products; check whether a shared schema exists.

### XP-02 Destructive intent needs one explicit token
- statement: A destructive API should have exactly one explicit representation of "all" (e.g. `"*"` or `reset`), enforced at one layer; otherwise empty inputs become ambiguous.
- evidence here: Mem0's four deletion units and divergent empty-scope rules.
- comparators: AMG `reset`, Graphiti delete APIs, generic REST bulk deletes.
- status: **Cross-project Candidate**.

### XP-03 Authentication ≠ authorization for memory servers
- statement: Self-hosted memory servers often authenticate requests but do not bind a principal to a memory's stored scope.
- evidence here: Mem0 server per-memory routes.
- comparators: AMG HTTP server explicitly "ships no authentication"; Graphiti server auth model.
- status: **Cross-project Candidate**; single-tenant deployments are a countercase.

### XP-04 Tri-state receipts collapse at the caller
- statement: When a store returns `unknown`, callers typically treat it as success; this converts a three-valued contract into a two-valued outcome.
- evidence here: `base.py:37-38`; `_update_memory`; Pinecone.
- comparators: other vector-store abstractions with `Optional[bool]`.
- status: **Cross-project Candidate**.

### XP-05 Audit trail is a projection of the primary store
- statement: Memory history tables are often written after the primary store mutation, so they are projections, not transactions.
- evidence here: add path (persist→history) and delete path (delete→history).
- comparators: event-sourced systems with outbox/2PC; AMG snapshot store.
- status: **Cross-project Candidate**; requires a second project with a different write order.

## Human-review register

| id | question | why human |
|---|---|---|
| HR-01 | Is self-hosted server intended to be single-tenant? | product/deployment intent is not in code |
| HR-02 | Should `delete_users()` no-arg be an error? | backward compatibility / API contract decision |
| HR-03 | Is the `delete.mdx` warning describing client or server behavior? | doc ownership |
| HR-04 | Is `tests/test_server_auth.py` expected to run in CI with Postgres? | CI config ownership |
| HR-05 | Should `app_id` be added to self-hosted bulk delete? | platform/self-hosted feature parity decision |

## Candidate discipline

- C-01, C-04, C-06, C-07, C-11 are **Hypotheses**; none is written into a KO as fact.
- C-02, C-08, C-09, C-10, C-13 are **Facts**, but their product intent is unresolved.
- XP-01…XP-05 are **Cross-project Candidates**; none is labeled Principle.

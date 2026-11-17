# 02 Engineering Knowledge — Mem0 2.2.1 (EK Graph)

- run_id: `ARCH-2026-11-17-001`
- scope: refresh delta + invariants that the delta activates or changes
- numbering: this run's own EK-01…EK-50. Old 2.1.0 EK-01…EK-36 are preserved in `archaeology-runs/ARCH-2026-09-21-001/`; mapping is in `refresh-delta.md`.
- edge types: `mechanism` / `subsystem` / `causal` / `dependency` / `constraint` / `contrast`
- rule: every EK has at least one link; isolated facts are put in §D instead of being given a fake edge.

## A. Scope and identity admission

### EK-01 Client rejects blank entity IDs before any HTTP call
- category: ENGINEERING / PERMISSION
- layer: L2
- epistemic: Fact
- statement: The platform client validates all four entity IDs (`user_id`, `agent_id`, `app_id`, `run_id`) and raises `ValueError` for empty or whitespace-only values before constructing a request. The error text states that an empty ID would match every memory in the project.
- evidence: `mem0/client/main.py:102-109`; `tests/test_client_entity_guards.py:115-157` (sync/async, empty/space/tab/newline, all four fields); 66 passed.
- links: `mechanism`→EK-02; `subsystem`→EK-12; `constraint`→EK-09.

### EK-02 OSS entity-ID validation also rejects internal whitespace
- category: ENGINEERING / PERMISSION
- layer: L2
- epistemic: Fact
- statement: `_validate_and_trim_entity_id` coerces non-strings to `str`, trims, rejects empty/whitespace-only values, and additionally rejects any internal whitespace. The client guard in EK-01 does not reject internal whitespace, so `"u 1"` passes the client but fails OSS validation.
- evidence: `mem0/memory/main.py:175-209`; used by `add`/`search`/`get_all`/`delete_all`; client guard `mem0/client/main.py:102-109` only calls `.strip()`.
- links: `contrast`↔EK-01; `subsystem`→EK-04; `constraint`→EK-06.

### EK-03 Add-path metadata cannot set identity keys
- category: ENGINEERING / MEMORY
- layer: L2
- epistemic: Fact
- statement: On creation, `_strip_identity_keys(metadata, {}, context="add()")` drops `user_id`, `agent_id`, `run_id`, and `actor_id` from caller metadata and logs a warning. The storage scope is then written only from the explicit entity parameters.
- evidence: `mem0/memory/main.py:138-162`, `313-365`, `376-389`; `tests/memory/test_main.py:440-555` (`test_add_metadata_cannot_set_identity_fields`, async equivalent; issues #4490/#6277/#6655).
- links: `causal`→EK-29; `constraint`→EK-04; `mechanism`↔EK-09; `subsystem`→EK-23.

### EK-04 Update-path identity keys are silently accepted only when unchanged
- category: ENGINEERING / MEMORY
- layer: L2
- epistemic: Fact
- statement: On update, `_strip_identity_keys(metadata, existing_payload)` keeps only non-identity keys. For an identity key whose value equals the current payload it emits no warning; for a changed value it warns and still drops the key. Therefore update metadata can restate the existing scope but cannot move a memory to a different scope.
- evidence: `mem0/memory/main.py:143-162`, `2072-2074`; `tests/memory/test_main.py:440-555`.
- links: `contrast`↔EK-03; `mechanism`↔EK-09; `causal`→EK-31.

### EK-05 `search` and `get_all` reject top-level entity parameters
- category: ENGINEERING / PERMISSION
- layer: L1
- epistemic: Fact
- statement: Query methods require entity IDs in `filters={"user_id": "..."}`; top-level `user_id`/`agent_id`/`run_id` raise `ValueError`. This is the reverse of `add`, which accepts only top-level entity arguments and has no `filters` parameter.
- evidence: `mem0/memory/main.py:165-172`, `search`/`get_all` calls; `tests/memory/test_main.py` filter tests.
- links: `contrast`↔EK-06; `constraint`→EK-39; `subsystem`→EK-02.

### EK-06 Scope is expressed in two grammars across three SDK surfaces
- category: ENGINEERING / PERMISSION
- layer: L2
- epistemic: Fact
- statement: Python OSS `Memory.add` and Python Platform `MemoryClient.add` both take entity IDs only as top-level kwargs; `Memory` has no `filters` parameter, and `MemoryClient` explicitly rejects `filters`. TS OSS `Memory.add` accepts both top-level `userId`/`agentId`/`runId` and entity IDs inside `filters`; the TS test `keeps scope keys the caller passed in` proves `filters:{user_id:"carol"}` succeeds with no top-level `userId`. The meaningful split is therefore **two grammars** (Python top-level-only vs TS top-level-or-filters) enforced across **three surfaces**.
- evidence: `mem0/memory/main.py:760-773`; `mem0/client/main.py:94-99`; `mem0-ts/src/oss/src/memory/index.ts:770-800`; `mem0-ts/src/oss/tests/memory.add-filters-isolation.test.ts:112-126`.
- links: `contrast`↔EK-05; `mechanism`↔EK-07; `subsystem`→EK-08; `causal`→KO-02.

### EK-07 `filters` is a second scope channel in TS OSS, not the only one
- category: ENGINEERING / API
- layer: L2
- epistemic: Fact
- statement: In TS OSS, `add()` copies `config.filters` into a fresh object, then resolves top-level `userId`/`agentId`/`runId` into `filters` and metadata. A scope can come from either the top-level config or `filters`; the required-scope guard reads the merged `filters` object. In Python OSS, `add()` never accepts a `filters` argument, so the same intent is a `TypeError`.
- evidence: `mem0-ts/src/oss/src/memory/index.ts:770-800`; dynamic Python check: `_build_filters_and_metadata(input_filters={"user_id":"u1"})` raises `ValidationError` because no top-level ID is supplied.
- links: `mechanism`↔EK-06; `contrast`↔EK-03; `dependency`→EK-39.

### EK-08 `app_id` exists in the client but not in OSS identity constants
- category: ENGINEERING / PERMISSION
- layer: L1
- epistemic: Fact
- statement: The platform client's `ENTITY_PARAMS` includes `app_id`; the OSS memory core's `ENTITY_PARAMS` includes only `user_id`, `agent_id`, `run_id`. The two same-named constants therefore do not describe the same scope universe.
- evidence: `mem0/client/main.py:40-41`; `mem0/memory/main.py:134-140`.
- links: `contrast`↔EK-01; `constraint`→EK-15; `subsystem`→EK-12.

### EK-09 Identity is a protected metadata namespace, not ordinary payload
- category: ENGINEERING / MEMORY
- layer: L2
- epistemic: Observation
- statement: Mem0 treats entity IDs as a namespace owned by the API surface, not by the caller's metadata. Both add and update paths call `_strip_identity_keys`; `actor_id` is included even though it is query-only. This is a deliberate boundary, not a generic metadata filter.
- evidence: `mem0/memory/main.py:138-162`, `313-365`, `2072-2074`; issue comments in source (#4490/#6277/#6655).
- links: `mechanism`↔EK-03; `mechanism`↔EK-04; `subsystem`→EK-23; `constraint`→KO-09.

### EK-10 Unique `(memory_id)` is the only identity used by update/delete
- category: ENGINEERING / PERMISSION
- layer: L1
- epistemic: Fact
- statement: `_update_memory` and `_delete_memory` take a `memory_id` and never take a `user_id`/`agent_id`/`run_id` to authorize the operation. The memory's stored scope is read from the payload for history/entity cleanup, not compared to a caller principal.
- evidence: `mem0/memory/main.py:2052-2114`, `2116-2144`; public `update(memory_id, ...)` / `delete(memory_id)`.
- links: `constraint`→EK-18; `contrast`↔EK-01; `causal`→KO-04.

## B. Destructive operations

### EK-11 Python OSS `delete_all` requires at least one explicit filter
- category: ENGINEERING / PERMISSION
- layer: L2
- epistemic: Fact
- statement: `Memory.delete_all` validates the three IDs, builds a filter dict, and raises `ValueError` when the dict is empty, directing callers to `reset()` for a full wipe. This is a preflight refusal, not a server-side 400.
- evidence: `mem0/memory/main.py:1904-1928`.
- links: `contrast`↔EK-14; `subsystem`→EK-12; `constraint`→KO-03.

### EK-12 Python OSS `delete_all` paginates and guards against repeated batches
- category: ENGINEERING / MEMORY
- layer: L2
- epistemic: Fact
- statement: The loop lists memories with `top_k=DELETE_ALL_BATCH_SIZE` (1000), deletes each batch, tracks a tuple of sorted IDs, and stops with a warning if the same batch repeats. This prevents an infinite loop when a store keeps returning the same page after deletion.
- evidence: `mem0/memory/main.py:136`, `1932-1951`; `DELETE_ALL_BATCH_SIZE = 1000`.
- links: `mechanism`↔EK-13; `causal`→EK-11; `subsystem`→EK-38.

### EK-13 TS OSS `deleteAll` uses the same pagination idea with a batch guard
- category: ENGINEERING / MEMORY
- layer: L2
- epistemic: Fact
- statement: TS OSS `deleteAll` paginates with `DELETE_ALL_BATCH_SIZE = 1000`, tracks seen batches, and logs `Stopping deleteAll after a repeated memory batch`. It also refuses to run with no filter, telling callers to use `reset()`.
- evidence: `mem0-ts/src/oss/src/memory/index.ts:129-131`, `1720-1765`.
- links: `mechanism`↔EK-12; `contrast`↔EK-11; `subsystem`→EK-18.

### EK-14 Platform `delete_all` has no empty-scope preflight
- category: ENGINEERING / PERMISSION
- layer: L2
- epistemic: Fact
- statement: `MemoryClient.delete_all` calls `_reject_filters` and `_reject_blank_entity_ids`, then sends `DELETE /v1/memories/` with `params={}` when called with no arguments. Mocked HTTP verification confirms `call('/v1/memories/', params={})`; no local error is raised. The docs claim the SDK raises when no filters are provided.
- evidence: `mem0/client/main.py:554-586`; dynamic run: `delete_all()` → `call('/v1/memories/', params={})`; `docs/core-concepts/memory-operations/delete.mdx:136-141`.
- links: `contrast`↔EK-11; `contrast`↔EK-13; `constraint`→EK-16; `causal`→KO-03.

### EK-15 Platform `delete_users` with no IDs enumerates and deletes every entity
- category: ENGINEERING / PERMISSION
- layer: L2
- epistemic: Fact
- statement: `delete_users` checks blank IDs, then if no ID is supplied calls `users()` (`GET /v1/entities/`) and constructs a delete request for every returned entity. `test_no_ids_still_deletes_every_listed_entity` blesses this behavior; `reset()` calls `delete_users()`.
- evidence: `mem0/client/main.py:621-674`; `tests/test_client_entity_guards.py:149-157`; `reset()` implementation.
- links: `contrast`↔EK-14; `mechanism`↔EK-12; `constraint`→KO-03.

### EK-16 `"*"` wildcard is passed through without local translation
- category: ENGINEERING / PERMISSION
- layer: L1
- epistemic: Fact
- statement: The platform client permits `"*"` and sends it as an ordinary query value. The self-hosted server's `DELETE /memories` route forwards the same strings to `Memory.delete_all`, which builds literal filters. No code translates `"*"` into "all entities" at either layer. The likely result on self-hosted is zero matches or a provider-specific error, not a guaranteed full wipe; this was not runtime-verified.
- evidence: `mem0/client/main.py:555-586`; `server/main.py:528-542`; `mem0/memory/main.py:1917-1928`; `tests/test_client_entity_guards.py` wildcard cases.
- links: `constraint`→EK-17; `contrast`↔EK-11; `causal`→C-04.

### EK-17 Platform and self-hosted are two API families with different paths and parameters
- category: ENGINEERING / API
- layer: L2
- epistemic: Fact
- statement: `MemoryClient` targets the hosted Platform and calls `/v1/memories/`, `/v1/memories/{id}/`, `/v2/entities/...`; its `ENTITY_PARAMS` includes `app_id`. The self-hosted FastAPI serves `/memories`, `/search`, `/reset`, `/configure` with no `/v1` prefix, and `DELETE /memories` accepts only `user_id`, `agent_id`, `run_id` — no `app_id`. A Platform client is not a client for the self-hosted server; parity must be compared between two contracts, not within one call path.
- evidence: `mem0/client/main.py:40-41, 555-586, 1360`; `server/main.py:322-557`; `server/main.py:527-542`.
- links: `contrast`↔EK-08; `constraint`→EK-16; `subsystem`→EK-53; `causal`→KO-10.

### EK-18 Destructive verbs are separate implementations, not one capability
- category: ENGINEERING / PERMISSION
- layer: L2
- epistemic: Observation
- statement: `delete(memory_id)`, `delete_all(filters)`, `delete_users(entities)`, and `reset(collection)` each have a different unit of deletion (one memory / scoped memories / entities / whole store), a different authority gate, and a different empty-input rule. There is no shared `delete(scope)` primitive.
- evidence: `mem0/memory/main.py:1883-1958`, `2116-2144`; `mem0/client/main.py:528-674`; `server/main.py:515-554`.
- links: `subsystem`→EK-11; `subsystem`→EK-14; `subsystem`→EK-15; `contrast`↔EK-19.

## C. Server authority

### EK-19 Auth has three input modes and two legacy shortcuts
- category: ENGINEERING / PERMISSION
- layer: L2
- epistemic: Fact
- statement: `verify_auth` resolves JWT Bearer tokens, then `X-API-Key` (legacy admin key or DB lookup), then `AUTH_DISABLED` bypass. An empty `ADMIN_API_KEY` is not equivalent to disabled auth; `AUTH_DISABLED=true` is the explicit bypass.
- evidence: `server/auth.py:144-175`; `tests/test_server_auth.py` still assumes empty admin key disables auth, which is stale.
- links: `causal`→EK-20; `subsystem`→EK-21; `constraint`→KO-04.

### EK-20 Per-memory CRUD is authenticated but not authorized
- category: ENGINEERING / PERMISSION
- layer: L2
- epistemic: Fact
- statement: `GET /memories/{id}`, `PUT /memories/{id}`, `DELETE /memories/{id}`, `GET /memories/{id}/history`, and scoped `GET /memories?...` depend on `verify_auth` only. They never compare the authenticated user/API-key identity to the memory payload's `user_id`/`agent_id`/`run_id`.
- evidence: `server/main.py:411-440`, `443-525`; `server/auth.py:144-175`.
- links: `causal`→EK-19; `contrast`↔EK-21; `constraint`→KO-04; `causal`→C-05.

### EK-21 Admin gate is limited to bulk delete and reset
- category: ENGINEERING / PERMISSION
- layer: L1
- epistemic: Fact
- statement: `DELETE /memories` and `POST /reset` use `require_admin`; every other memory route uses `verify_auth`. `require_admin` has a bootstrap fallback: if DB has no default user it returns `_BOOTSTRAP_ADMIN` when auth is via admin key/disabled.
- evidence: `server/main.py:527-554`; `server/auth.py:178-233`.
- links: `contrast`↔EK-20; `dependency`→EK-19; `subsystem`→KO-04.

### EK-22 Scoped list is open to any authenticated caller
- category: ENGINEERING / PERMISSION
- layer: L1
- epistemic: Fact
- statement: `GET /memories?user_id=...` does not require admin; only the no-identifier listing does. Any authenticated caller can therefore list memories for any `user_id`/`agent_id`/`run_id` it names.
- evidence: `server/main.py:411-440`.
- links: `causal`→EK-20; `constraint`→KO-04.

### EK-23 Auth test suite encodes 2.1.x assumptions
- category: TESTING / FAILURE
- layer: L1
- epistemic: Fact
- statement: `tests/test_server_auth.py` sets `ADMIN_API_KEY=""` to simulate disabled auth, never sets `JWT_SECRET`, and asserts `get_all` is called with top-level `user_id`. In 2.2.1, auth-disabled requires `AUTH_DISABLED=true`, startup requires `JWT_SECRET`, and the route calls `get_all(filters=...)`. In this environment the suite produced 56 failed / 18 passed with `JWT_SECRET` set, and setup errors without it.
- evidence: `tests/test_server_auth.py:60-80`; `server/main.py:432`; `server/auth.py:15-16`; actual run.
- links: `contrast`↔EK-19; `contrast`↔EK-20; `causal`→C-11.

### EK-24 Refresh-token replay is prevented by an atomic JTI update
- category: ENGINEERING / PERMISSION
- layer: L1
- epistemic: Fact (2.1.0 invariant, unchanged in this window)
- statement: Refresh tokens carry a JTI row; rotation updates the row atomically so a replayed token cannot be reused.
- evidence: `server/auth.py`, `server/models.py` (`RefreshTokenJti`); unchanged by `git diff a39a802..HEAD -- server` (only dashboard/requirements changed).
- links: `mechanism`↔EK-19; `subsystem`→KO-04.

### EK-25 `require_auth` may hit the database for a default user
- category: ENGINEERING / RUNTIME
- layer: L1
- epistemic: Fact
- statement: In `admin_api_key`/`disabled` mode, `require_auth` calls `_get_default_user`; if the DB has no default user it returns 401. This is why the auth test suite fails without a reachable Postgres even when a legacy admin key is supplied.
- evidence: `server/auth.py:178-190`; observed `psycopg.OperationalError: failed to resolve host 'postgres'`.
- links: `causal`→EK-23; `dependency`→EK-19.

## D. Write pipeline

### EK-26 V3 is a single-pass ADD-only phased pipeline
- category: ENGINEERING / MEMORY
- layer: L2
- epistemic: Fact (2.1.0 invariant, re-read at 2.2.1)
- statement: One LLM extraction call produces self-contained facts; subsequent phases embed, deduplicate, persist, link entities, and save messages. The pipeline is ADD-only at the write stage; conflict resolution is not a separate per-fact LLM decision loop.
- evidence: `mem0/memory/main.py` "V3 PHASED BATCH PIPELINE"; `_add_to_vector_store`; old package EK-01.
- links: `subsystem`→EK-27; `causal`→EK-30; `mechanism`↔EK-03.

### EK-27 Phase 6 reports only persisted records
- category: ENGINEERING / MEMORY
- layer: L2
- epistemic: Fact
- statement: Batch insert success marks the whole batch as persisted; batch failure falls back to one-by-one inserts and only successes enter `persisted_records`. If none succeed, messages are saved and `VectorStoreError` is raised. History, entity links, and returned results all use `persisted_records`.
- evidence: `mem0/memory/main.py:1047-1110`; `tests/memory/test_main.py:1215-1259` (`test_rejected_records_are_not_reported_as_add`, `test_all_inserts_failed_raises_vector_store_error`); issue #7066.
- links: `causal`→EK-28; `mechanism`↔EK-29; `constraint`→KO-05.

### EK-28 History is derived from persisted records, not intended records
- category: ENGINEERING / EVIDENCE
- layer: L2
- epistemic: Fact
- statement: `history_records` is built by iterating `persisted_records`, not the original `records` list. A store-rejected memory therefore leaves no `ADD` history row. This is the evidence-layer counterpart of EK-27.
- evidence: `mem0/memory/main.py:1078-1099`.
- links: `causal`→EK-27; `subsystem`→EK-37; `constraint`→KO-05.

### EK-29 A total insert failure is surfaced, not silently swallowed
- category: ENGINEERING / FAILURE
- layer: L2
- epistemic: Fact
- statement: If every insert fails, the code calls `db.save_messages` and then raises `VectorStoreError("Failed to insert any of the N extracted memories...")`. It does not return an empty list as if extraction found nothing.
- evidence: `mem0/memory/main.py:1072-1076`; test `test_all_inserts_failed_raises_vector_store_error`.
- links: `causal`→EK-27; `contrast`↔EK-30; `constraint`→KO-06.

### EK-30 Extraction failure re-raises instead of returning an empty list
- category: ENGINEERING / FAILURE
- layer: L1
- epistemic: Fact (2.1.0 invariant)
- statement: LLM extraction exceptions are re-raised; an empty extraction result is a separate branch that saves messages and returns `[]`. Failure and "no facts found" are not conflated.
- evidence: `mem0/memory/main.py` extraction exception handling; old package EK-04.
- links: `contrast`↔EK-29; `causal`→EK-27.

### EK-31 Phase 8 saves messages even when extraction produced no records
- category: ENGINEERING / MEMORY
- layer: L1
- epistemic: Fact
- statement: `db.save_messages(messages, session_scope)` is called when `records` is empty and again after successful insertion. The message log is not gated on memory creation.
- evidence: `mem0/memory/main.py:1043-1045`, `1218`.
- links: `contrast`↔EK-28; `subsystem`→EK-26.

### EK-32 Entity links are built only from persisted records
- category: ENGINEERING / MEMORY
- layer: L1
- epistemic: Fact
- statement: Phase 7 builds `all_texts` from `persisted_records` and iterates the same list to link entities. A rejected record cannot create dangling entity links.
- evidence: `mem0/memory/main.py:1100-1110`.
- links: `causal`→EK-27; `mechanism`↔EK-28.

### EK-33 Missing scope on add raises a typed validation error
- category: ENGINEERING / API
- layer: L1
- epistemic: Fact
- statement: When no `user_id`/`agent_id`/`run_id` is provided, `_build_filters_and_metadata` raises `Mem0ValidationError` with code `VALIDATION_001` and a suggestion. It does not fall back to a global/default scope.
- evidence: `mem0/memory/main.py:391-397`.
- links: `constraint`→EK-03; `causal`→KO-01.

### EK-34 Text-change detection gates entity relinking on update
- category: ENGINEERING / MEMORY
- layer: L1
- epistemic: Fact
- statement: `_update_memory` compares the new text to the previous text and only calls entity-store removal/relink when `text_changed` is true. A metadata-only update does not churn entity links.
- evidence: `mem0/memory/main.py:2070`, `2107-2112`.
- links: `causal`→EK-37; `contrast`↔EK-32.

## E. Update, delete and receipts

### EK-35 The vector-store update contract is tri-state
- category: ENGINEERING / CONTRACT
- layer: L2
- epistemic: Fact
- statement: `VectorStoreBase.update` must "never create a record". `False` means no record exists, `True` means updated, `None` means the store cannot tell and callers treat it as updated.
- evidence: `mem0/vector_stores/base.py:33-40`.
- links: `constraint`→EK-36; `contrast`↔EK-39; `subsystem`→KO-06.

### EK-36 Qdrant full update uses `UPDATE_ONLY` where supported
- category: ENGINEERING / IMPLEMENTATION
- layer: L2
- epistemic: Fact
- statement: For a full vector+payload update, Qdrant uses `models.UpdateMode.UPDATE_ONLY` when the client exposes it, and checks existence under the write lock in local mode. A missing point returns `False`.
- evidence: `mem0/vector_stores/qdrant.py:518-549`.
- links: `mechanism`↔EK-35; `contrast`↔EK-38; `subsystem`→EK-39.

### EK-37 Qdrant retains an upsert fallback on older clients
- category: ENGINEERING / EDGE CASE
- layer: L1
- epistemic: Fact
- statement: When `models.UpdateMode` is absent, the `update_mode` kwarg is omitted and the call falls through to `upsert`; the source comment acknowledges "older servers and clients upsert as before". On a non-local older deployment this can create a missing point, violating the "never create" contract.
- evidence: `mem0/vector_stores/qdrant.py:540-548`.
- links: `contrast`↔EK-36; `constraint`→EK-35; `causal`→C-06.

### EK-38 Pinecone update returns `None` and cannot report missing IDs
- category: ENGINEERING / EDGE CASE
- layer: L1
- epistemic: Fact
- statement: Pinecone's `update()` sends `index.update(...)` and returns no value. The comment says update never creates a missing id but does not report whether it existed. `_update_memory` treats `None` as success and writes an `UPDATE` history row, so an update against a deleted Pinecone memory can report success while doing nothing.
- evidence: `mem0/vector_stores/pinecone.py:315-331`; `mem0/memory/main.py:2087-2105`.
- links: `contrast`↔EK-39; `constraint`→EK-35; `causal`→C-07.

### EK-39 TS update throws on `False`, mirroring Python
- category: ENGINEERING / IMPLEMENTATION
- layer: L1
- epistemic: Fact
- statement: TS OSS `updateMemory` calls `vectorStore.update` and throws `Memory with ID ... not found` when the result is `false`; `undefined`/`null` is otherwise not treated as missing. This matches the Python tri-state interpretation.
- evidence: `mem0-ts/src/oss/src/memory/index.ts:1990-2001`.
- links: `mechanism`↔EK-35; `contrast`↔EK-38.

### EK-40 Python and TS update metadata identity semantics differ
- category: ENGINEERING / MEMORY
- layer: L2
- epistemic: Fact
- statement: Python `_update_memory` calls `_strip_identity_keys(metadata, existing_payload)`, which silently preserves an identity value equal to the current payload and warns only on a changed value. TS `updateMemory` calls `stripIdentityKeys(metadata)` with no existing payload, which unconditionally removes every identity key. Both keep the stored scope unchanged, but only Python distinguishes "same" from "changed".
- evidence: `mem0/memory/main.py:2072-2074`; `mem0-ts/src/oss/src/memory/index.ts:1968-1978`, `121-127`.
- links: `contrast`↔EK-04; `mechanism`↔EK-09.

### EK-41 Single delete checks existence but not scope
- category: ENGINEERING / PERMISSION
- layer: L1
- epistemic: Fact
- statement: `delete(memory_id)` reads the memory first and raises if absent, then `_delete_memory` writes a `DELETE` history row and removes entity references. It never asks who owns the memory.
- evidence: `mem0/memory/main.py:1883-1902`, `2116-2144`; `server/main.py:515-524`.
- links: `causal`→EK-20; `contrast`↔EK-11; `constraint`→KO-04.

### EK-42 Delete history is written after the vector delete
- category: ENGINEERING / EVIDENCE
- layer: L1
- epistemic: Fact
- statement: `_delete_memory` calls `vector_store.delete` first, then `db.add_history(..., "DELETE", ...)`. If history write fails after the vector is gone, the memory is deleted but the evidence trail is incomplete. There is no transaction spanning both stores. `delete_all` calls `_delete_memory` per record, so a bulk delete emits N DELETE rows, not one aggregate event.
- evidence: `mem0/memory/main.py:2126-2144`.
- links: `contrast`↔EK-28; `constraint`→KO-05; `causal`→C-08.

### EK-43 Entity-store cleanup uses stored scope, not caller scope
- category: ENGINEERING / MEMORY
- layer: L1
- epistemic: Fact
- statement: Update and delete build `session_filters` from the memory payload's `user_id`/`agent_id`/`run_id`, not from caller arguments. The stored memory is the authority for which entity links to remove.
- evidence: `mem0/memory/main.py:2109-2112`, `2126`.
- links: `mechanism`↔EK-41; `subsystem`→EK-37.

## F. Search and retrieval (refreshed invariants)

### EK-44 Hybrid retrieval fuses three signals
- category: ENGINEERING / RETRIEVAL
- layer: L2
- epistemic: Fact (2.1.0 invariant)
- statement: Search combines semantic similarity, BM25 lexical scores, and entity boosts. Semantic candidates are over-fetched before fusion.
- evidence: `mem0/utils/scoring.py`; `mem0/memory/main.py` search path; old package EK-09.
- links: `subsystem`→EK-45; `subsystem`→EK-47; `causal`→EK-48.

### EK-45 Threshold is applied before fusion
- category: ENGINEERING / RETRIEVAL
- layer: L2
- epistemic: Fact (2.1.0 invariant)
- statement: A semantic similarity threshold filters candidates before BM25/entity boosts are added. A candidate below the semantic threshold cannot be rescued by lexical or entity signals.
- evidence: `mem0/memory/main.py` search threshold branch; old package EK-12.
- links: `constraint`→EK-44; `contrast`↔EK-46.

### EK-46 BM25 normalization adapts to query length
- category: ENGINEERING / RETRIEVAL
- layer: L2
- epistemic: Fact (2.1.0 invariant)
- statement: BM25 scores are normalized with a query-length-adaptive sigmoid so short and long queries do not receive the same lexical weight profile.
- evidence: `mem0/utils/scoring.py`; old package EK-10.
- links: `mechanism`↔EK-44; `contrast`↔EK-45.

### EK-47 Entity boost is sourced from the entity store
- category: ENGINEERING / RETRIEVAL
- layer: L2
- epistemic: Fact (2.1.0 invariant)
- statement: Query entities are extracted, matched against stored entities above a similarity threshold, and linked memory IDs receive an additive boost.
- evidence: `mem0/utils/scoring.py`, `mem0/memory/main.py`; old package EK-11/EK-13.
- links: `causal`→EK-44; `subsystem`→EK-43.

### EK-48 Semantic over-fetch is 4x
- category: ENGINEERING / RETRIEVAL
- layer: L1
- epistemic: Fact (2.1.0 invariant)
- statement: The semantic search requests four times the final `top_k` before fusion and filtering, so lexical/entity signals have candidates to promote.
- evidence: `mem0/memory/main.py` search path; old package EK-09.
- links: `mechanism`↔EK-44; `causal`→EK-45.

### EK-49 UUID mapping prevents LLM-hallucinated IDs from reaching storage
- category: ENGINEERING / FAILURE
- layer: L2
- epistemic: Fact (2.1.0 invariant)
- statement: Existing memories are presented to the extraction model with integer aliases; returned aliases are mapped back to real UUIDs. A hallucinated or out-of-range ID is discarded rather than used as a vector ID.
- evidence: `mem0/memory/main.py` UUID mapping; old package EK-05.
- links: `subsystem`→EK-26; `constraint`→KO-07.

### EK-50 Hash dedup prevents duplicate facts on write
- category: ENGINEERING / MEMORY
- layer: L1
- epistemic: Fact (2.1.0 invariant)
- statement: Each extracted fact is hashed (MD5 of text) and compared with existing hashes before insertion; duplicates are skipped.
- evidence: `mem0/memory/main.py` hash path; `mem0/memory/storage.py`; old package EK-02.
- links: `mechanism`↔EK-26; `subsystem`→EK-49.

## G. Configuration, product surface and refresh evidence

### EK-51 Profiles are a new platform-only surface
- category: PRODUCT / API
- layer: L1
- epistemic: Fact
- statement: 2.2.1 adds `generate_profile`, `get_profile`, `get_profile_job`, and `get/update_profile_settings` against `/v2/profiles/...`, plus a per-entity `GET /v2/entities/user/{id}/profile/`. `_UNSET` distinguishes "omit" from "explicit None clears". This is product surface, not part of the OSS memory write path.
- evidence: `mem0/client/main.py:44-48`, profile methods; `docs/api-reference/profiles/generate-profiles.mdx`, `get-profile-job.mdx`, `get-profile-settings.mdx`, `get-profile.mdx`, `update-profile-settings.mdx`; `tests/test_client_profiles.py` (part of 49 passed).
- links: `subsystem`→EK-08; `contrast`↔EK-06.

### EK-52 Delete docs assert a guard that Platform clients do not implement
- category: DOCUMENTATION / CONTRACT
- layer: L1
- epistemic: Fact + Observation
- statement: `delete.mdx` states that `delete_all` raises an error when no filters are provided, and separately that empty/whitespace IDs are rejected. The second claim is true; the first is not implemented by either Python or TS platform client, both of which send an empty request.
- evidence: `docs/core-concepts/memory-operations/delete.mdx:136-141`; `mem0/client/main.py:554-586`; `mem0-ts/src/client/mem0.ts:569-590`.
- links: `contrast`↔EK-14; `causal`→C-03.

### EK-53 Server application code is unchanged in the refresh window
- category: ENGINEERING / EVIDENCE
- layer: L1
- epistemic: Fact
- statement: `git diff --stat a39a802..HEAD -- server` lists only `server/dashboard/package.json`, `pnpm-lock.yaml`, `pnpm-workspace.yaml`, and `server/requirements.txt`. No `server/main.py` or `server/auth.py` logic changed between 2.1.0 and 2.2.1.
- evidence: `git diff --stat a39a802..HEAD -- server`.
- links: `contrast`↔EK-20; `causal`→C-11.

### EK-54 22 vector-store files changed, but receipt semantics are uneven
- category: ENGINEERING / CONTRACT
- layer: L1
- epistemic: Fact
- statement: 22 files under `mem0/vector_stores/` changed in the refresh. `base.py` declares tri-state; Qdrant, Pinecone, and many others implement `None`; therefore the contract is not uniformly enforceable across adapters.
- evidence: `git diff --name-only a39a802..HEAD -- mem0/vector_stores | wc -l` = 22; `mem0/vector_stores/base.py:33-40`; `pinecone.py:330`.
- links: `subsystem`→EK-35; `contrast`↔EK-39; `causal`→C-06.

### EK-55 Test suite size and execution state
- category: TESTING / EVIDENCE
- layer: L1
- epistemic: Fact
- statement: The snapshot has 132 Python `test_*.py`, 174 TS `*.test.ts`, 421 Python files / 126,148 Python LOC, and 457 TS + 136 TSX files / 107,858 LOC. Targeted Python tests passed 66 + 49 + 8; TS tests were not run because `mem0-ts/node_modules` is absent.
- evidence: `find`, `wc -l`, `pytest` runs; snapshot artifact §6/§9.
- links: `subsystem`→EK-23; `contrast`↔EK-53.

## H. D 级 (project-local, kept but not promoted)

- `DELETE_ALL_BATCH_SIZE = 1000` is a tuning constant; it is project-local.
- `PROFILE_JOBS_PATH` / `PROFILE_SETTINGS_PATH` string constants are implementation detail.
- `ConfigManager` base-URL isolation is a deployment convenience; no cross-project claim.
- The `app_id` naming mismatch is retained as EK-08 because it has cross-surface consequences.

## I. EK graph statistics

- EK count: 55
- link declarations: 157
- average out-degree: 2.85
- isolated EK: 0
- strongest cluster: scope/identity (EK-01…EK-10)
- second cluster: destructive semantics (EK-11…EK-18)
- third cluster: server authority (EK-19…EK-25)
- weakest cluster: retrieval (EK-44…EK-50), mostly carried invariants rather than new delta evidence

The retrieval cluster is intentionally thinner in this refresh: 2.2.1 did not rewrite scoring. Its EKs are retained because the old package's EK-09…EK-16 still apply and KO-02 depends on them.

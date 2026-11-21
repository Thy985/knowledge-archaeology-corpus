# 04 Flow Atlas — Mem0 2.2.1

- run_id: `ARCH-2026-11-17-001`
- source: code + tests at `f1791daa`; not inferred from docs alone
- edge schema: `id | from → to | condition | symbol/file | evidence`
- seven classes: Control / State / Data / Evidence / Authority / Memory / Policy

## 1. Control Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| CF-01 | `Memory.add` → `_build_filters_and_metadata` | always, before pipeline | `mem0/memory/main.py:760-810` | code |
| CF-02 | `_build_filters_and_metadata` → `Mem0ValidationError` | no `user_id`/`agent_id`/`run_id` | `main.py:391-397` | code; dynamic check |
| CF-03 | `_build_filters_and_metadata` → `_add_to_vector_store` | scope valid | `main.py:810-...` | code |
| CF-04 | `_add_to_vector_store` → Phase 6 `vector_store.insert` | records exist | `main.py:1047-1062` | code |
| CF-05 | Phase 6 batch insert → per-record insert | batch insert raises | `main.py:1063-1070` | code; `test_rejected_records_are_not_reported_as_add` |
| CF-06 | per-record insert → `VectorStoreError` | `persisted_records` empty | `main.py:1072-1076` | code; test |
| CF-07 | Phase 6 → Phase 7 entity linking | at least one persisted | `main.py:1100-1110` | code |
| CF-08 | `_update_memory` → `vector_store.update` | existing memory found | `main.py:2052-2091` | code |
| CF-09 | `vector_store.update` → raise | return is `False` | `main.py:2092-2093` | code |
| CF-10 | `_delete_memory` → `vector_store.delete` → `db.add_history` | normal | `main.py:2126-2144` | code; order is observable |
| CF-11 | `server.delete_all_memories` → `require_admin` → `Memory.delete_all` | authenticated admin | `server/main.py:527-542` | code |
| CF-12 | `server.get_memory` → `verify_auth` → `Memory.get` | any authenticated | `server/main.py:443-449` | code |
| CF-13 | `MemoryClient.delete_all` → `_reject_filters` → `_reject_blank_entity_ids` → DELETE | no blank IDs | `mem0/client/main.py:554-586` | code; mock call |
| CF-14 | `MemoryClient.delete_users` → `users()` → per-entity DELETE | no ID supplied | `mem0/client/main.py:657-673` | code; test |

## 2. State Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| SF-01 | caller metadata → cleaned metadata | identity key present | `_strip_identity_keys`, `main.py:143-162` | code |
| SF-02 | cleaned metadata → `base_metadata_template` | non-identity keys | `main.py:363-389` | code |
| SF-03 | explicit entity params → `base_metadata_template` | ID provided | `main.py:376-389` | code |
| SF-04 | memory payload → `new_metadata` | update | `main.py:2072` | code |
| SF-05 | `existing_payload` → `_strip_identity_keys` comparison | update | `main.py:2074` | code |
| SF-06 | `persisted_records` → history rows | at least one | `main.py:1078-1089` | code |
| SF-07 | `persisted_records` → entity link set | at least one | `main.py:1100-1110` | code |
| SF-08 | payload scope → `session_filters` | update/delete | `main.py:2109, 2126` | code |
| SF-09 | `delete_all` filter dict → vector store filters | non-empty filter | `main.py:1917-1938` | code |
| SF-10 | client kwargs → HTTP params | after blank/filters checks | `mem0/client/main.py:575-579` | code; mock |
| SF-11 | TS `config.filters` → new `filters` object | add | `mem0-ts/src/oss/src/memory/index.ts:770-772` | code |
| SF-12 | TS resolved `userId` → `filters.user_id` + `metadata.user_id` | userId present | `index.ts:781-787` | code |

## 3. Data Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| DF-01 | messages → parsed messages | always | `parse_messages`, `main.py` add path | code |
| DF-02 | parsed messages → LLM extraction | `infer=True` | `_add_to_vector_store` | code; old EK-01 |
| DF-03 | extraction output → records | valid JSON / fallback | `main.py` extraction parser | code; old EK-29 |
| DF-04 | records → embeddings | batch/individual | `main.py:1020-1041` | code |
| DF-05 | records → hash | per fact | `main.py:1034` | code |
| DF-06 | records → vector payload | per fact | `main.py:1030-1041` | code |
| DF-07 | query → semantic search | search call | `main.py` search path | code |
| DF-08 | query → BM25 score | keyword capability | `mem0/utils/scoring.py` | code; old EK-10 |
| DF-09 | query entities → entity store match | entities present | `mem0/utils/scoring.py` | code; old EK-11 |
| DF-10 | filters → vector store list | `delete_all` | `main.py:1937-1938` | code |
| DF-11 | profile request → `/v2/profiles/...` | client profile method | `mem0/client/main.py` | code; tests |
| DF-12 | server query params → `filters` | any identifier | `server/main.py:429-436` | code |

## 4. Evidence Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| EF-01 | `persisted_records` → `history_records` | insert succeeded | `main.py:1078-1089` | code |
| EF-02 | `history_records` → `db.batch_add_history` | batch succeeds | `main.py:1090-1091` | code |
| EF-03 | `history_records` → per-row `add_history` | batch fails | `main.py:1092-1098` | code |
| EF-04 | `vector_store.delete` → `db.add_history("DELETE")` | delete succeeds | `main.py:2127-2135` | code; ordering |
| EF-05 | `updated is False` → ValueError | missing point | `main.py:2092-2093` | code |
| EF-06 | `updated is None` → `UPDATE` history | store cannot tell | `main.py:2087-2105` | code; Pinecone |
| EF-07 | TS `updated === false` → throw | missing point | `index.ts:1999-2001` | code |
| EF-08 | test assertion → regression evidence | test passes | `tests/memory/test_main.py:1215-1259` | test run |
| EF-09 | `delete.mdx` claim → docs evidence | docs build | `docs/core-concepts/memory-operations/delete.mdx:136-141` | doc |
| EF-10 | server test failure → drift evidence | test run | `tests/test_server_auth.py` | 56 failed / 18 passed |

## 5. Authority Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| AF-01 | request → `verify_auth` | every memory route | `server/main.py` decorators | code |
| AF-02 | `verify_auth` → JWT user | Bearer token | `server/auth.py:144-175` | code |
| AF-03 | `verify_auth` → admin shortcut | `X-API-Key == ADMIN_API_KEY` | `auth.py:160-170` | code |
| AF-04 | `verify_auth` → DB API key lookup | non-admin key | `auth.py:170-175` | code |
| AF-05 | `verify_auth` → bypass | `AUTH_DISABLED=true` | `auth.py:15-16, 144-150` | code |
| AF-06 | `require_admin` → role check | bulk delete / reset | `server/main.py:532, 548` | code |
| AF-07 | `require_admin` → `_BOOTSTRAP_ADMIN` | no default user in DB | `auth.py:193-233` | code |
| AF-08 | authenticated principal → memory scope | **no comparison** | absent check in `server/main.py:443-525` | code; absence |
| AF-09 | client blank ID guard → no HTTP call | blank ID | `client/main.py:102-109` | code; tests |
| AF-10 | client `filters` guard → ValueError | `add`/`delete_all` | `client/main.py:94-99` | code; tests |
| AF-11 | OSS identity strip → scope cannot be set by metadata | add/update | `main.py:143-162` | code; tests |

## 6. Memory Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| MF-01 | messages → SQLite recent messages | add path end | `main.py:1218` | code |
| MF-02 | facts → vector store | persisted | `main.py:1057-1062` | code |
| MF-03 | facts → hash index | insert | `main.py:1034` | code |
| MF-04 | entity mentions → entity store | Phase 7 | `main.py:1100-1110` | code |
| MF-05 | update text → entity relink | `text_changed` | `main.py:2110-2112` | code |
| MF-06 | delete → entity unlink | `_delete_memory` | `main.py:2136-2144` | code |
| MF-07 | delete_all → repeated batch guard | same sorted ID tuple | `main.py:1942-1946` | code |
| MF-08 | TS deleteAll → repeated batch guard | same batch signature | `index.ts:1750-1765` | code |
| MF-09 | reset → collection deletion + recreate | admin | `server/main.py:547-554`; `Memory.reset` | code |
| MF-10 | graph memory (documented Platform feature) | configured | `docs/platform/features/graph-memory.mdx` | doc; no OSS implementation found |

## 7. Policy Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| PF-01 | caller metadata → identity keys dropped | add | `_strip_identity_keys(..., {})` | code; test |
| PF-02 | caller metadata → identity keys dropped (same value silent) | update | `_strip_identity_keys(..., existing)` | code; test |
| PF-03 | blank entity ID → ValueError | client / OSS | `client/main.py:102-109`; `main.py:175-209` | code; tests |
| PF-04 | no scope on add → `VALIDATION_001` | add | `main.py:391-397` | code |
| PF-05 | no scope on OSS delete_all → ValueError | delete_all | `main.py:1925-1928` | code |
| PF-06 | no scope on platform delete_all → empty request | delete_all | `client/main.py:575-579` | code; mock |
| PF-07 | no scope on platform delete_users → enumerate all | delete_users | `client/main.py:657-660` | code; test |
| PF-08 | wildcard `"*"` → literal filter | delete_all | `client/main.py:575-579`; `main.py:1917-1928` | code |
| PF-09 | `app_id` on platform delete_all → no server route | self-hosted | `client/main.py:40-41`; `server/main.py:529-531` | code |
| PF-10 | docs say delete_all raises → client does not | any no-filter call | `delete.mdx:137` vs `client/main.py:575-579` | doc/code contradiction |
| PF-11 | tri-state `None` → treated as success | update | `main.py:2092-2105`; `base.py:37-38` | code |
| PF-12 | Qdrant old client → upsert fallback | `not hasattr(models, "UpdateMode")` | `qdrant.py:543-548` | code |
| PF-13 | profiled project settings → `enabled` project-wide | profiles API | `client/main.py` profiles methods | code; tests |
| PF-14 | server per-memory route → no scope policy | any authenticated | `server/main.py:443-525` | code |

## Flow → KO cross-check

| Flow class | Edges | KOs supported | Coverage note |
|---|---|---|---|
| Control | 14 | KO-01, KO-03, KO-05, KO-06, KO-08 | add/update/delete/server paths traced |
| State | 12 | KO-01, KO-02, KO-03 | scope and metadata state traced |
| Data | 12 | KO-02, KO-07, KO-08 | extraction → store → retrieval |
| Evidence | 10 | KO-05, KO-06, KO-09 | receipt and history order |
| Authority | 11 | KO-02, KO-04, KO-10 | auth and entity guards |
| Memory | 10 | KO-03, KO-05, KO-08, KO-09 | store/entity/history writes |
| Policy | 14 | KO-01, KO-03, KO-04, KO-06, KO-10 | explicit rule surfaces |

Total edges: 83. Every KO has at least one inbound Flow edge; reverse mapping is in each KO's `reverse trace` line.

## Flow limitations

- `CF-11` and `AF-06` were code-read, not executed against a live server.
- `MF-10` graph memory is out of the refresh focus and only mapped at module level.
- `DF-11` profiles are mapped by method, not by every payload branch.
- `PF-09` `app_id` route mismatch is a static signature comparison; runtime FastAPI behavior (ignoring unknown query params) was not executed.

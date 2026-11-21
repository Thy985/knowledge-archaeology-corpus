# 01 Project Layer — Mem0 2.2.1

- run_id: `ARCH-2026-11-17-001`
- commit: `f1791daa86d469249c9b002f62644d58a1eb96a0`
- purpose of this refresh: re-baseline the E-layer scope contract and the F-layer write/delete trust boundary after the 2.1.0→2.2.1 delta

## 1. What Mem0 is

Mem0 is a long-term memory layer for AI agents. It has two deployment forms:

1. **OSS / self-hosted**: `mem0ai` Python SDK, `mem0-ts` TypeScript SDK, and a FastAPI `server/` whose routes are `/memories`, `/search`, `/reset`, `/configure` (no `/v1` prefix).
2. **Hosted platform**: `MemoryClient` (Python/TS) talks to `api.mem0.ai` by default and calls versioned routes such as `/v1/memories/` and `/v2/entities/...`.

These are **two API families**, not one client calling one server. The Platform client is documented as a hosted credential (`api_key`), while a config dict selects the OSS backend; `host` on the Platform client is a dedicated Platform base URL, not a switch to self-hosted OSS.

The OSS core does not replay raw conversation. It extracts self-contained facts from messages with an LLM, stores them in a vector store plus SQLite history, and retrieves them with a hybrid of semantic search, BM25, and entity boosts.

## 2. Repository map

| Path | Responsibility | Entry |
|---|---|---|
| `mem0/memory/main.py` | OSS memory lifecycle, ADD pipeline, search, update, delete | `Memory` / `AsyncMemory` |
| `mem0/memory/storage.py` | SQLite history + message store | `SQLiteManager` |
| `docs/platform/features/graph-memory.mdx` | graph memory is a documented Platform feature; no OSS 2.2.1 implementation file found | n/a |
| `mem0/memory/utils.py` | parsing, lemmatization, filter helpers | functions |
| `mem0/utils/scoring.py` | hybrid scoring | `score_and_rank` |
| `mem0/client/main.py` | hosted-platform HTTP client | `MemoryClient` / `AsyncMemoryClient` |
| `mem0/client/types.py` | typed options, request/response models | `AddMemoryOptions`, `DeleteAllMemoryOptions`, profile types |
| `mem0/vector_stores/*` | 30+ vector adapters | `VectorStoreBase` contract |
| `mem0/llms/*`, `mem0/embeddings/*` | provider factories | `LlmFactory`, `EmbedderFactory` |
| `mem0/configs/*` | Pydantic config models | `MemoryConfig` |
| `server/main.py` | self-hosted REST API | FastAPI app |
| `server/auth.py` | JWT / API-key / disabled auth modes | `verify_auth`, `require_auth`, `require_admin` |
| `server/routers/*` | API-key, auth, dashboard routes | routers |
| `mem0-ts/src/oss/src/memory/index.ts` | TS OSS memory lifecycle | `Memory` |
| `mem0-ts/src/client/mem0.ts` | TS platform client | `MemoryClient` |
| `mem0-ts/src/oss/tests/*` | TS Jest tests | test files |
| `tests/*` | Python pytest suite | test files |
| `docs/*` | public docs | MDX |
| `integrations/*` | framework adapters (LangChain, LlamaIndex, etc.) | adapters |
| `openmemory/*` | MCP memory application | separate runtime |
| `cli/*` | Python and Node CLIs | CLI entrypoints |

## 3. Lifecycle of a memory

### 3.1 Creation path (`Memory.add`, Python OSS)

1. Caller supplies at least one of `user_id`, `agent_id`, `run_id` as top-level keyword arguments.
2. `_build_filters_and_metadata` validates and trims each ID; missing all three raises `Mem0ValidationError`.
3. Caller `metadata` is filtered through `_strip_identity_keys` with an empty existing payload, so `user_id`/`agent_id`/`run_id`/`actor_id` cannot be injected through metadata.
4. The V3 phased pipeline runs: context fetch → existing-memory retrieval with UUID mapping → one LLM extraction → batch embeddings → hash dedup → batch insert.
5. Phase 6 inserts the batch. Only records confirmed persisted become `persisted_records`; if none persist, `VectorStoreError` is raised.
6. History (`ADD`) and entity links are built from `persisted_records` only.
7. Phase 8 saves recent messages to SQLite regardless of extraction/insert outcome.

### 3.2 Retrieval path

- `search` rejects top-level entity parameters and requires `filters` to contain at least one entity ID.
- Semantic over-fetch (4x) → optional BM25 scoring with query-length-adaptive sigmoid → entity boost from entity store → threshold gate before fusion.
- `get_all` follows the same filter requirement.

### 3.3 Update path

- `_update_memory` first does `vector_store.get`; missing ID raises `ValueError`.
- Caller metadata is stripped of identity keys, but an identical existing identity value is silently accepted (only a changed value warns).
- `vector_store.update` is called. `False` means "missing" and raises; `None` means "store cannot tell" and is treated as success.
- On success, an `UPDATE` history row is written; entity-store references are cleaned and relinked only if the text changed.

### 3.4 Delete path

- `delete(memory_id)` does an existence check, then `_delete_memory`; it writes a `DELETE` history row and removes entity links.
- `delete_all(user_id/agent_id/run_id)` in Python OSS requires at least one filter; it paginates vector-store listing in batches of 1000 and stops on a repeated batch.
- `MemoryClient.delete_all` (platform) does **not** require a filter; the HTTP call is sent with empty params.
- `MemoryClient.delete_users()` with no IDs lists all entities and deletes each one.
- `reset()` deletes the collection and recreates it.

## 4. Core data shapes

- Memory payload: `data`, `hash`, `text_lemmatized`, `created_at`, `updated_at`, plus entity IDs and optional `actor_id`, `role`, `attributed_to`, `expiration_date`.
- History event: `ADD` / `UPDATE` / `DELETE`, with `old_memory`, `new_memory`, `created_at`, `updated_at`, `is_deleted`.
- `SearchFilters`: entity IDs plus metadata operators (`eq`, `ne`, `in`, `nin`, `gt`, `gte`, `lt`, `lte`, `contains`, `icontains`, `AND`, `OR`, `NOT`).
- Platform request models: `AddMemoryOptions` (top-level entity IDs only; `filters` removed in 2.2.1), `DeleteAllMemoryOptions` (top-level entity IDs incl. `app_id`).
- Server models: `User`, `APIKey`, `RefreshTokenJti`.

## 5. State stores

| Store | What it holds | Who writes |
|---|---|---|
| Vector store | memory vector + payload | `Memory`, `server` |
| SQLite history DB | history rows, recent messages | `Memory` |
| Entity store / graph store | entities + memory links | `Memory` |
| Postgres/SQLite (server) | users, API keys, refresh-token JTIs | `server` |
| Remote platform | hosted memory + profiles | `MemoryClient` |

The vector store is the source of truth for memory existence; history and entity links are derived. `server` auth state is separate from memory scope.

## 6. Configuration

- `MemoryConfig`: `llm`, `embedder`, `vector_store`, `graph_store`, `history_db_path`, `version`, custom prompts.
- Environment: `OPENAI_API_KEY`, provider keys, `MEM0_API_KEY` / `MEM0_BASE_URL` for platform client, server `JWT_SECRET`, `ADMIN_API_KEY`, `AUTH_DISABLED`, and database URL.
- `ConfigManager` isolates the base URL so one process can talk to multiple platform endpoints.
- `pyproject.toml` version `2.2.1`; `mem0-ts/package.json` is the TypeScript package authority.

## 7. Authority and permission model

### 7.1 OSS SDK

The OSS `Memory` object has **no authorization layer**. Any code that holds the object can pass any `user_id`, `agent_id`, or `run_id`, and can read/update/delete any `memory_id`. Scope is a naming convention enforced by argument validation, not an ACL.

### 7.2 Platform client

`MemoryClient` adds preflight guards:

- `ENTITY_PARAMS = {"user_id","agent_id","app_id","run_id"}`.
- blank/whitespace-only IDs are rejected before any HTTP call;
- `filters` is rejected on `add` and `delete_all`;
- `"*"` is accepted for intentional bulk deletion;
- `delete_users` accepts no IDs and then enumerates all entities.

These guards protect against accidental empty-scope requests. They do not authenticate the caller and do not bind a request to a principal.

### 7.3 Self-hosted server

`server/auth.py` supports three input modes:

1. JWT Bearer (`Authorization: Bearer ...`) → `_resolve_user_from_jwt`.
2. `X-API-Key` → if equal to `ADMIN_API_KEY`, returns `None` (legacy admin); otherwise looks up the key in the DB.
3. `AUTH_DISABLED=true` → bypass.

`verify_auth` authenticates every request. `require_admin` is used only for `DELETE /memories` (bulk) and `POST /reset`. Per-memory `GET`, `PUT`, `DELETE`, `GET .../history`, and scoped `GET /memories?user_id=...` only require `verify_auth`; they do not compare the authenticated principal to the memory's scope. A valid API key for user A can therefore read, update, or delete a memory created by user B if it knows the `memory_id`.

`GET /memories` with no identifiers is admin-only. `GET /memories` with any identifier is not admin-gated.

### 7.4 Server route coverage for entity types

`DELETE /memories` accepts only `user_id`, `run_id`, and `agent_id`. It has **no `app_id` parameter**. This is a self-hosted contract gap relative to the Platform client's `ENTITY_PARAMS`, which includes `app_id`; the two are separate API families, so the comparison is cross-contract, not a single client calling this route.

## 8. Test system

- Python: 132 `test_*.py`; pytest; `tests/memory/`, `tests/vector_stores/`, `tests/client/`, `tests/test_server_auth.py`.
- TS: 174 `*.test.ts`; Jest; `mem0-ts/src/oss/tests/`, `mem0-ts/src/client/tests/`.
- The 2.2.1 delta adds TS tests for filters isolation, entity guards, profiles, missing optional peers, turbopuffer score/unit, and valkey.
- `tests/test_server_auth.py` is stale relative to 2.2.1 auth semantics: it assumes `ADMIN_API_KEY=""` disables auth, does not set `JWT_SECRET`, expects a reachable Postgres, and asserts `get_all(user_id=...)` while the route now calls `get_all(filters=...)`.

## 9. External dependencies

- Python: `pydantic`, `httpx`, `sqlalchemy`, `fastapi`, `python-jose`, `passlib`, `slowapi`, `psycopg`, vector clients, LLM/embedding SDKs.
- Node: TypeScript SDK with provider peer dependencies; `mem0-ts/node_modules` absent in this snapshot.
- No live LLM, embedder, vector DB, or Postgres was started. Runtime observations are limited to mocked client calls and `TestClient` attempts.

## 10. Refresh delta scope

40 commits changed 269 files. The parts that matter to this run:

- `mem0/client/main.py` +422/−11 and `types.py` +14/−25: entity guards, profiles, delete semantics.
- `mem0/memory/main.py` +65/−17: identity stripping, entity validation, Phase 6 persisted records, update receipt.
- 22 vector-store files: update contract + never-create semantics, plus provider-specific fallbacks.
- `mem0-ts` tests: filters isolation, entity guards, profiles, missing optional peers, turbopuffer, valkey.
- `docs/core-concepts/memory-operations/delete.mdx` +4: new "delete_all raises" claim.
- `server/` application code did **not** change in this window; only dashboard package/workspace/lock/requirements files did.

## 11. Project-layer conclusions

1. Mem0 2.2.1 is a polyglot product with three different scope entry surfaces (Python OSS, Python platform client, TS OSS), not one unified API.
2. The strongest new invariant is at the metadata boundary: caller metadata cannot set identity on creation, and can only restate an existing identity on update.
3. The weakest new surface is destructive intent: `delete_all`, `delete_users`, and `reset` do not share a single "empty scope" rule across SDKs and server routes.
4. The self-hosted server's authorization is authentication-only for per-memory operations; scope is not an ACL.
5. The update receipt contract improved from "update means upsert" to tri-state, but `None` remains a legal "unknown" that callers treat as success.

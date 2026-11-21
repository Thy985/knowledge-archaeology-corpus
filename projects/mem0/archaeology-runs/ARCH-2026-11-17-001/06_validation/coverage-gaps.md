# Coverage Gaps — Mem0 2.2.1 refresh

- run_id: `ARCH-2026-11-17-001`
- purpose: list what this run did **not** prove, so the corpus does not over-claim.

## 1. Runtime gaps

| Gap | Why it matters | What would close it |
|---|---|---|
| No live vector store | Qdrant `UPDATE_ONLY`, Pinecone `None`, wildcard `"*"`, batch delete are all static or mocked | run Qdrant local + Pinecone test index with delete/update races |
| No live HTTP server | server auth, route signatures, and `app_id` behavior are code-read only | boot `server.main` with `AUTH_DISABLED=true` and a fake `Memory`; use `TestClient` |
| No live Postgres | `require_auth` / `require_admin` default-user path untested | docker Postgres + migrations + one admin and one normal user |
| No LLM / embedder | extraction, UUID mapping, hash dedup, fallback are carried from 2.1.0 | inject a deterministic fake LLM/embedder into `Memory` |
| No real `api.mem0.ai` call | Platform client behavior is mock-tested only | a sandbox API key + one project |
| No TS execution | TS scope and filters-copy claims rest on code + test source | `npm ci` in `mem0-ts` and run the six new tests |
| No graph memory run | graph memory is a Platform doc feature; no OSS 2.2.1 implementation found | separate hosted-platform run; out of F/E focus |

## 2. Test gaps

- `tests/test_server_auth.py`: 56 failed / 18 passed in this environment. Some failures need Postgres, some need `JWT_SECRET`, some are stale assertions (`ADMIN_API_KEY=""`, `get_all(user_id=...)`). The package treats this as **drift evidence**, not as proof that CI is red.
- `tests/test_client_entity_guards.py`: 66 passed. Good.
- `tests/test_client_profiles.py` + `tests/memory/test_main.py -k ...`: 49 passed, 27 deselected. Good.
- `tests/vector_stores/test_qdrant.py -k ...`: 8 passed, 98 deselected. Narrow.
- No full pytest run.
- No mutation/fault injection for Phase 6 partial failure, history failure, or delete-history failure.

## 3. Evidence gaps

| Claim area | Evidence level | Missing |
|---|---|---|
| scope grammars | S4 Python, S3 TS | TS execution |
| blank-ID guard | S4 | hosted API behavior |
| destructive empty scope | S3 + mock | server/API response |
| authn ≠ authz | S3 | live two-key test |
| tri-state receipt | S3 | Pinecone/Qdrant live race |
| history projection | S3 | fault injection |
| refresh delta | S3 (git diff) | full re-read of 23 old EKs |

## 4. Bypass / alternate path audit

Paths that were checked:
- direct `Memory` object vs `MemoryClient` vs TS `Memory`
- direct vector-store `update` / `delete`
- `server` admin vs non-admin
- `AUTH_DISABLED`
- Qdrant local vs server mode
- Pinecone `None`
- TS `updated === false`

Paths **not** fully checked:
- `AsyncMemory` / `AsyncMemoryClient` parity (only some tests read)
- `mem0/proxy/main.py` behavior after `filters` rejection
- graph memory
- integration adapters (LangChain/LlamaIndex/CrewAI) — not in this repo focus
- telemetry/notices side effects

## 5. Old-package carry gaps

`refresh-delta.md` marks 23 of 36 old EKs `NR` (not re-evidenced). This is honest but incomplete:
- the old EKs about retrieval, telemetry, OSS funnel, and integration adapters were not re-read;
- if one of them changed in the 40 commits, this run would not detect it unless the file appeared in the diff stat.

## 6. Next run candidates (do not start now)

1. **Qdrant/Pinecone live receipts** — closes C-06/C-07 and KO-06.
2. **Self-hosted server two-principal test** — closes C-05 and KO-04 with runtime evidence.
3. **TS test execution** — closes the S4 gap for KO-02/07/08.
4. **Graphiti × Mem0 cross-project comparison** — tests XP-01/XP-03/XP-05.
5. **Full old-EK re-read** — closes the `NR` debt.

## 7. Explicit non-claims

- This run does not claim a security vulnerability in production. It claims **code-level boundaries and inconsistencies**; runtime exploitation was not attempted.
- This run does not claim the server is multi-tenant. HR-01 asks the owner.
- This run does not claim `delete_all()` is safe or unsafe end-to-end. C-01 is unresolved.
- This run does not claim the old 2.1.0 package is wrong; it claims 2.2.1 changed enough to require a delta.

# Repository Snapshot — Hy-Memory 2.0

- run_id: `ARCH-2026-11-20-001`
- repository: `https://github.com/Tencent-Hunyuan/Hy-Memory.git`
- branch: `main`
- HEAD: `d8138b501a755f662a024f7e8eb879613eabf1d3`
- commit timestamp: `2026-11-18 23:54:43 +0800`
- commit subject: `feat(memhub): offline Qdrant BM25 in 1.0.8 build 49 (#456)`
- clone: `git clone --depth 50` → `archaeology-jobs/ARCH-2026-11-20-001/repo`
- shallow history: `git rev-list --count HEAD = 548`; `.git/shallow` present
- working tree after probes: clean (`git status --porcelain` = 0), ignored pytest caches removed
- repository size: 72 MB
- tracked files: 1,881

## 1. Version and packaging

| Fact | Value | Evidence |
|---|---|---|
| Package name | `hy-memory` | `pyproject.toml:1-6` |
| Version | `2.0.17` | `pyproject.toml:7` |
| Python requirement | `>=3.9` | `pyproject.toml` |
| License | Apache-2.0 | `pyproject.toml`; `LICENSE` |
| README latest release | `hy-memory==2.0.16` + MemHub `1.0.7` | `README.md:36` |
| Version discrepancy | package is `2.0.17` while README's latest release row says `2.0.16`; no tag/CHANGELOG reconciliation done in this run | `pyproject.toml:7` vs `README.md:36` |
| Build system | setuptools | `pyproject.toml` |

## 2. Repository map

Top-level:

- `hy_memory/` — Python memory engine and SDK (primary scope).
- `memhub/` — TypeScript/React desktop app, CLI and Python sidecar. 592 Python files / 195,821 lines are largely vendored sidecar code, not the SDK; excluded from deep archaeology.
- `plugins/{hermes,openclaw}/` — host integrations. 5 Python files / 2,517 lines; read only as boundary evidence.
- `docs/` — usage, env reference, graph design, coding scene, papers. 66 Markdown files repository-wide.
- `tests/` — 157 Python files / 39,175 lines.
- `scripts/`, `pyproject.toml`, `uv.lock`, `README.md`, `README.zh.md`, `CHANGELOG.md`, `SECURITY.md`.

Engine package `hy_memory/`:

| Area | Files | Lines | Role |
|---|---:|---:|---|
| `hy_memory/` total | 154 py | 105,375 | SDK/engine |
| `agent/` | 14 | 7,338 | extraction, reconcile, episode summary, tools |
| `coding/` | 19 | 7,836 | coding scene, business store, diff chain |
| `core/` | 5 | 2,026 | embedding service, loop/thread runtime, workload gates |
| `data/` | 17 | 14,316 | vector stores (Chroma/Qdrant), graph stores (Kuzu/Neo4j), history, cache, business store |
| `models/` | 3 | 2,191 | memory node, enums, output protocol |
| `pipelines/` | 56 | 28,913 | writer, readers, reconcile, System 2, retrieval helpers |
| `prompts/` | 12 | 25,453 | 11 language prompt families + shared rules |
| `utils/` | 15 | 2,516 | source filter, raw trimming, observability, tracing |

Largest engine files (repository line counts):

| File | Lines |
|---|---:|
| `hy_memory/client.py` | 8,173 |
| `hy_memory/pipelines/writer.py` | 3,858 |
| `hy_memory/pipelines/reader_base.py` | 3,693 |
| `hy_memory/data/vector_store_qdrant.py` | 3,215 |
| `hy_memory/data/graph_store_kuzu.py` | 2,755 |
| `hy_memory/data/graph_store_neo4j.py` | 2,480 |
| `hy_memory/pipelines/system2_agent.py` | 2,476 |
| `hy_memory/prompts/*` (11 language files) | ~2,249–2,314 each |
| `hy_memory/reembed.py` | 1,995 |
| `hy_memory/agent/reconciler.py` | 1,575 |

File-type distribution (tracked): 911 `.py`, 594 `.ts`, 132 `.tsx`, 66 `.md`, 42 `.mjs`, 30 `.css`, 29 `.json`, 14 `.cjs`, 11 `.svg`, 8 `.png`, 6 `.yml`, 4 `.yaml`, 4 `.sh`.

## 3. Entry points

- Package export surface: `hy_memory/__init__.py` exports `HyMemoryClient`, `MemoryInspector`, `SharedRuntime`, and the re-embed family.
- Primary SDK entry: `hy_memory/client.py:446` `HyMemoryClient`; `add` at `:1729`, `search` at `:1847`, `get` at `:2016`, `update` at `:2042`, `delete` at `:2072`, `delete_all` at `:2096`, `list_memories` at `:2121`, `evolve` at `:2389`, `sweep` at `:2473`, `close` at `:2586`; async twins `async_add` `:2875`, `_async_add_impl` `:3010`, `async_search` `:3949`, `_async_search_impl` `:4024`.
- Async loop bridge: `_LoopThread` at `hy_memory/client.py:304`; one persistent write loop, plus a separate search loop in solo mode.
- Writer: `hy_memory/pipelines/writer.py:123` `MemoryWriter`, `write` `:2570`, `write_extracted` `:2426`.
- Reader: `hy_memory/pipelines/reader.py` dispatcher; `reader_legacy.py` / `reader_hybrid.py`; shared post-processing in `reader_base.py:296`.
- System 2: `hy_memory/pipelines/system2_writer.py:46`; `system2_agent.py:1938` `run_system2_agent`; `system2_tools.py:387` `System2ToolExecutor`.
- Component factory: `hy_memory/pipelines/registry.py` `ComponentFactory`.
- Other surfaces: `hy_memory/server.py`, `snapshot.py`, `inspector.py`, `runtime.py`, `reembed.py`; `memhub/` CLI/desktop; `plugins/hermes`, `plugins/openclaw`.

## 4. Runtime dependencies and backends

Main dependencies from `pyproject.toml`: `pydantic`, `numpy`, `openai>=1.40,<3`, `python-dotenv`, `pyyaml`, `chromadb>=1.5.9`, `kuzu`, `langdetect`, `scikit-learn`, `tiktoken`, `json-repair`, `rank-bm25`, `jieba`.

Optional extras:

- `qdrant` — Qdrant vector store;
- `faiss` — FAISS vector backend;
- `graph` — Neo4j graph store (Kuzu is in the main dependency set);
- `mysql`, `postgres` — business/relational stores;
- `rerank` — torch + transformers local reranker;
- `all`, `dev` — aggregate and test/lint extras.

Default backend shape from `config.py`:

- vector provider default `chroma`, collection `agent_memories`, persistence `<data>/data/vector_db`, embedding dims default `1536`;
- embedder default `openai` + `text-embedding-3-small`;
- graph default `kuzu` when graph is enabled; `ultra` forces graph on, `pro` defaults graph off via `MEMORY_ENABLE_GRAPH`, `lite` forces graph off;
- business store default SQLite (`coding_memory.db` family), with `coding_memory_cv.db` and `*_cv` vector collections for coding;
- history store default enabled at `<data>/data/history.db`;
- cache backend SQLite by default, MySQL optional.

## 5. Modes and scene routing

README + code agree on three modes:

- `lite`: raw-only embedding write/search, no LLM, ignores scene/layers routing.
- `pro`: embedding + LLM extraction/reconcile; graph optional, default off.
- `ultra`: `pro` + explicit `evolve()` System 2 schema induction + cross-domain sweeper; graph mandatory.

Three product scenes:

- `chat`: hierarchical L0–L7 long-term memory; `lite` raw-only, `pro` adds extraction/reconcile, `ultra` adds System 2.
- `working`: one `WorkingFact` per session + 0–N `Experience` entries, two-way ID links, superseded facts retained via `fact_chain_summary`.
- `coding`: same working schema plus software-engineering extraction, separate `coding_memory_cv.db` / `*_cv` vector namespaces, and a zero-LLM diff chain keyed by `user_id + workspace_id`, branch, and hunk/file block.

## 6. Test system

- Framework: pytest + pytest-asyncio; `testpaths=["tests"]`, `asyncio_mode=auto`, default addopts include `-m 'not integration'`; markers `slow`, `integration`, `asyncio`.
- Test corpus: 157 Python files / 39,175 lines.
- Subset executed for this snapshot (after installing `chromadb 1.5.9` + `faiss-cpu`):
  `test_reconcile_candidate_user_scope.py`, `test_reconcile_chain_merge.py`, `test_evolution_chain_shadow_filter.py`, `test_reconcile_failure.py`, `test_block_l1_raw_in_read_filter.py` → **40 passed in 20.07 s**.
- An earlier five-file subset without optional backends: 21 passed, 5 deselected, 15 errors, all errors `faiss-cpu is required` / `chromadb is required`; not logic failures.
- A full `pytest -q -m 'not integration' --timeout=120` background run did not yield a usable summary: pytest teardown raised `OSError: [Errno 24] Too many open files` on `/tmp/pytest-of-user/pytest-4`. Do not claim a full-suite pass from this run.
- `tests/test_vector_search_isolation.py` is entirely `@pytest.mark.integration` and needs a live Neo4j; because the default marker expression deselects integration, it provides no always-on isolation guarantee. Its fixture is real but is a localhost placeholder (`neo4j://localhost:7687`, user `neo4j`, password `password`).
- Test runs generated `__pycache__/` and `.pytest_cache/` inside the target clone. These were removed with `git clean -fdX`; working tree is clean. This is a process-hygiene violation of strict filesystem read-only even though no tracked file changed.

## 7. Permission and governance surfaces (map only)

- Multi-tenant identity is `user_id` (level 1), `agent_id` (level 2), `session_id` (level 3); `source` is an orthogonal caller channel.
- Public `search` rejects an empty `user_ids` list (`client.py:4125`), so the public SDK cannot perform an unscoped search; the vector-store layer itself has a no-user-condition fallback.
- `get` / `update` / `delete` are keyed only by `memory_id`; they do not accept an owner scope.
- `list_memories` defaults to excluding `L1_RAW`; `history` and `pipeline_logs` retain full text by default.
- Governance mechanisms present: source include/exclude, `mixed` source fail-closed, `SHADOW` logical deletion, graph scope checks on some but not all mutation tools, embedding-space re-embed with writer-pause contract, per-user reconcile lock (loop-local), episode finalization flags.
- Governance mechanisms absent or not evidenced: cross-process lock/CAS, graph write capability isolation, retention/TTL for `history.db` or `pipeline_logs`, and per-ID owner authorization in the SDK.

## 8. Snapshot limitations

- No live LLM or embedding endpoint was available, so no behavioral benchmark, extraction-quality or injection test was run.
- The repository's README links a CMB paper but does not publish the numeric scores used by KnowlegeMap's secondary summary; those numbers remain unverified and are excluded from EK/KO evidence.
- `memhub/` is mapped only as an integration surface; its 592 Python sidecar files and 706 TS/TSX files were not read module-by-module.
- The clone is shallow (50 commits); history-dependent claims (when a design changed, whether a bug was later fixed upstream) are out of scope.

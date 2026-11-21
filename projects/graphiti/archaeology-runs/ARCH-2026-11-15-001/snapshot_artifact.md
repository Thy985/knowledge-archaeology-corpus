# Repository Snapshot — Graphiti (`ed23196`)

## 1. Identity

| Field | Value | Evidence |
|---|---|---|
| Repository | `https://github.com/getzep/graphiti.git` | `git remote -v` |
| Branch | `main` | `git branch -a` |
| HEAD | `ed23196506eba8806fae6da831d73fcd2ad714d8` | `git rev-parse HEAD` |
| Commit time | `2026-11-14 09:12:37 +0000` | `git log -1 --format=%ci` |
| Commit subject | `@Elon-Jiang has signed the CLA in getzep/graphiti#2148` | `git log -1 --format=%s` |
| Package version | `0.31.2` | `pyproject.toml` `[project].version` |
| License | `Apache-2.0` | `pyproject.toml`; `LICENSE` |
| Python requires | `>=3.10,<4` | `pyproject.toml` |
| Snapshot time | 2026-11-15 | 本轮执行时间 |
| Clone method | `git clone --depth 1 --single-branch --branch main` | 本轮命令 |

GitHub API metadata recorded for traceability only: default branch `main`, license `Apache-2.0`, size `16,389 KB`, pushed_at `2026-11-14T09:12:40Z`, created_at `2024-08-08T22:08:30Z`, archived `false`. Stars `32,807` are recorded as metadata and were **not** used for job ranking.

## 2. Scale

| Metric | Value | Method |
|---|---|---|
| Repository size (with `.git`) | 31M | `du -sh .` |
| Files excluding `.git` | 424 | `find . -type f -not -path './.git/*' | wc -l` |
| Python files | 302 | `find . -name '*.py' -not -path './.git/*'` |
| Python lines | 77,773 | `cat` all Python files, `wc -l` |
| Test files (`test_*.py`) | 81 | `find . -name 'test_*.py'` |
| Static test functions | 910 | `grep -rhE '^\s*(async )?def test_' --include='test_*.py'` |

## 3. Top-Level Layout

```
graphiti_core/        core library (graph, nodes, edges, search, drivers, prompts)
server/               reference FastAPI service
mcp_server/           reference MCP server + in-memory per-group queue
tests/                unit/integration/e2e tests
examples/             usage examples
spec/                 interface/spec material
signatures/           signature-related material
prompts/              prompt assets
migrations/           package migrations (only __init__.py in snapshot)
docker-compose.yml    Neo4j/FalkorDB deployment
pyproject.toml        package metadata and dependencies
AGENTS.md / CLAUDE.md repo working rules
SECURITY.md           security policy and explicit server limitations
```

## 4. Core Module Map

| Area | Key files | Role |
|---|---|---|
| Facade | `graphiti_core/graphiti.py` (2,251 lines) | `Graphiti` class; `add_episode`, `add_episode_bulk`, `search`, `search_`, `add_triplet`, `remove_episode`, `build_communities` |
| Nodes | `graphiti_core/nodes.py` (1,166 lines) | `EpisodicNode`, `EntityNode`, `CommunityNode`, `SagaNode` |
| Edges | `graphiti_core/edges.py` (1,051 lines) | `EntityEdge`, `EpisodicEdge`, `CommunityEdge`, `NextEpisodeEdge`, `HasEpisodeEdge` |
| Search | `graphiti_core/search/search.py` (891 lines) | Multi-scope hybrid search, group filters, rerankers |
| Search config | `graphiti_core/search/search_config_recipes.py` | `COMBINED_HYBRID_SEARCH_RRF`, `..._MMR`, `..._CROSS_ENCODER`, edge/node variants |
| Search helper | `graphiti_core/search/search_helpers.py` | `format_edge_date_range`, `search_results_to_context_string` |
| Bulk write | `graphiti_core/utils/bulk_utils.py` | `add_nodes_and_edges_bulk`, `add_nodes_and_edges_bulk_tx` |
| Maintenance | `graphiti_core/utils/maintenance/edge_operations.py` | edge dedupe, contradiction, temporal invalidation |
| Node maintenance | `graphiti_core/utils/maintenance/node_operations.py` | node dedupe/resolution (identity → similarity → LLM) |
| Temporal normalization | `graphiti_core/utils/maintenance/temporal_edge_utils.py` | future-tense bounds, soft-deadline rules |
| Drivers | `graphiti_core/driver/{neo4j,falkordb,kuzu,neptune}_driver.py`; `driver/driver.py` | `GraphDriver`, `GraphDriverSession`, provider enum, transaction semantics |
| REST server | `server/graph_service/{main.py,config.py,zep_graphiti.py,routers/}` | FastAPI service, `/messages`, `/search`, `/get-memory`, `/clear` |
| MCP server | `mcp_server/src/graphiti_mcp_server.py`; `mcp_server/src/services/queue_service.py` | MCP tools and per-group in-memory queue |

## 5. Core Data Structures and State

- **Episode** (`EpisodicNode`): `content`, `source`, `source_description`, `created_at`, `valid_at`, `entity_edges`, `episode_metadata`. `valid_at` is “when the original document was created”.
- **EntityEdge**: `fact`, `fact_embedding`, `episodes` (provenance list), `created_at`, `valid_at`, `invalid_at`, `expired_at`, `reference_time`, `attributes`.
- **Temporal semantics in field descriptions**:
  - `valid_at`: when the fact became true; future plans use the episode reference time, not the future occurrence date.
  - `invalid_at`: when the fact stopped being true; soft deadlines are **not** `invalid_at`.
  - `expired_at`: when the system record was invalidated.
- **Graph partition**: `group_id`; also used for driver/database routing when it differs from the default database.
- **State location**: external graph database (Neo4j / FalkorDB / Kuzu / Neptune). Graphiti itself is largely stateless; `Graphiti` object holds driver, LLM/embedder/reranker clients, tracer, prompt library, `max_coroutines`.

## 6. Entry Points

| Entry | File / symbol | Notes |
|---|---|---|
| Library ingestion | `Graphiti.add_episode` | Single episode; docstring requires sequential await, recommends background queue |
| Library bulk ingestion | `Graphiti.add_episode_bulk` | Saves episodes first, then extracts/resolves and writes nodes/edges |
| Programmatic triplet | `Graphiti.add_triplet` | Bypasses extraction, still runs dedupe/invalidation |
| Search | `Graphiti.search` / `Graphiti.search_` | Basic edge search vs advanced multi-scope search |
| Communities | `Graphiti.build_communities` | Removes existing communities in scope, then rebuilds |
| REST | `server/graph_service/main.py` | `/messages`, `/entity-node`, `/search`, `/get-memory`, `/clear`, `/healthcheck` |
| MCP | `mcp_server/src/graphiti_mcp_server.py` | `add_memory`, `search_nodes`, `search_memory_facts`, `delete_episode`, `clear_graph`, etc. |

## 7. Dependencies

Base dependencies from `pyproject.toml`:
`pydantic>=2.11.5`, `neo4j>=5.26.0`, `openai>=1.91.0`, `tenacity>=9.0.0`, `numpy>=1.0.0`, `python-dotenv>=1.0.1`, `posthog>=3.0.0`, `certifi>=2026.10.19`.

Optional extras include Anthropic, Groq, Google GenAI, Kuzu, FalkorDB, VoyageAI, GLiNER2, Neo4j+OpenSearch, sentence-transformers.

## 8. Configuration and Governance

- `pyproject.toml`, `pytest.ini`, `Makefile`, `docker-compose.yml`, `docker-compose.test.yml`, `Dockerfile`.
- `server/.env.example`: OpenAI key/base URL, model names, Neo4j/FalkorDB settings, `db_backend`.
- `mcp_server/config/*.yaml` and `.env.example`: MCP transport, graphiti endpoint, group/server settings.
- `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `Zep-CLA.md`.
- `SECURITY.md` explicitly states the reference REST/MCP servers have **no authentication, no authorization, no caller isolation, no prompt-injection protection**, and that all episodes in a `group_id` share trust.

## 9. Test System

- 81 `test_*.py`, 910 static `def test_`.
- Temporal/edge evidence: `tests/utils/maintenance/test_temporal_edge_utils.py`, `test_edge_operations.py`, `test_edge_dedup_scenarios.py`, `test_combined_extraction.py`.
- Graph facade/mocks: `tests/test_graphiti_mock.py`.
- Integration: `tests/test_edge_int.py`, `tests/test_node_int.py`, `server/tests/test_live_falkordb_int.py`, `mcp_server/tests/test_integration.py`, `mcp_server/tests/test_group_routing.py`.
- **No tests were executed in this run.** Therefore no test result may be treated as S4 evidence.

## 10. Snapshot Limits

- Shallow clone: no full history, no historical commit archaeology beyond HEAD metadata.
- No services started; no database available; no LLM keys used.
- Counts are static file/system measurements, not runtime coverage.

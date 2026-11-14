# 01 · Project Layer — Graphiti

## 1. What It Is

Graphiti 是一个 **temporal graph memory library**：把非结构化 episode（文本/JSON/消息）抽取成实体与事实边，写入图数据库，并提供带时间语义的混合检索。它同时附带两个 reference server：FastAPI REST 与 MCP。

它不是完整的 Agent 产品，也不是托管记忆服务；核心是一个 Python 库 + 可替换的图驱动/LLM/embedding/reranker 适配层。

## 2. Architecture

```
Client / Agent / MCP
        │
        ▼
┌──────────────────────────────────────────────┐
│ Graphiti facade (graphiti_core/graphiti.py)  │
│  add_episode / add_episode_bulk / add_triplet│
│  search / search_ / build_communities        │
│  remove_episode                              │
└───────────────┬──────────────────────────────┘
                │  request-scoped group_id + clients
                ▼
┌──────────────────────────────────────────────┐
│ Extraction & maintenance                     │
│  extract_nodes / extract_edges  (LLM)        │
│  resolve_extracted_nodes         (identity→  │
│    similarity→LLM)                           │
│  resolve_extracted_edges (LLM dedupe +       │
│    temporal invalidation)                    │
│  extract_attributes_from_nodes               │
└───────────────┬──────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────┐
│ Persistence                                  │
│  add_nodes_and_edges_bulk  (execute_write)   │
│  driver.transaction() (Neo4j only real tx)   │
│  Neo4j / FalkorDB / Kuzu / Neptune           │
└───────────────┬──────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────┐
│ Search                                       │
│  edge / node / episode / community scopes    │
│  bm25 + cosine + BFS + RRF/MMR/rerank        │
│  group_id filter + optional date filters     │
└──────────────────────────────────────────────┘
```

### Reference surfaces

- **REST** (`server/graph_service/main.py`): `/messages`, `/entity-node`, `/search`, `/get-memory`, `/clear`, `/healthcheck`.
- **MCP** (`mcp_server/src/graphiti_mcp_server.py`): `add_memory`, `search_nodes`, `search_memory_facts`, `get_episodes`, `delete_episode`, `clear_graph`, etc.
- Both are explicitly labeled reference implementations, not hardened services (`SECURITY.md`).

## 3. Module Map

| Module | Responsibility | Key symbols |
|---|---|---|
| `graphiti_core/graphiti.py` | Public facade and ingestion orchestration | `Graphiti.__init__`, `add_episode`, `add_episode_bulk`, `add_triplet`, `remove_episode`, `search`, `build_communities` |
| `graphiti_core/nodes.py` | Node models | `EpisodicNode`, `EntityNode`, `CommunityNode`, `SagaNode` |
| `graphiti_core/edges.py` | Edge models | `EntityEdge`, `EpisodicEdge`, `CommunityEdge`, `NextEpisodeEdge`, `HasEpisodeEdge` |
| `graphiti_core/utils/bulk_utils.py` | Bulk persistence | `add_nodes_and_edges_bulk`, `add_nodes_and_edges_bulk_tx` |
| `graphiti_core/utils/maintenance/edge_operations.py` | Edge resolution, dedupe, temporal invalidation | `resolve_extracted_edges`, `_apply_temporal_invalidation`, `resolve_edge_contradictions` |
| `graphiti_core/utils/maintenance/node_operations.py` | Entity resolution | `resolve_extracted_nodes` |
| `graphiti_core/utils/maintenance/graph_data_operations.py` | Episode context retrieval | `retrieve_episodes` |
| `graphiti_core/search/search.py` | Multi-scope search | `search` |
| `graphiti_core/search/search_config_recipes.py` | Search recipes | `COMBINED_HYBRID_SEARCH_RRF`, `..._MMR`, `..._CROSS_ENCODER` |
| `graphiti_core/driver/driver.py` | Driver abstraction | `GraphDriver`, `GraphDriverSession`, `GraphProvider`, `transaction()` |
| `server/graph_service/` | FastAPI reference server | routers `ingest`, `retrieve`, `entity_node`, `clear` |
| `mcp_server/` | MCP reference server | tools + `QueueService` |

## 4. Core Data Model

### Node types

- **EpisodicNode**: the source episode. Fields include `content`, `source`, `source_description`, `created_at`, `valid_at`, `entity_edges`, `episode_metadata`.
- **EntityNode**: extracted entity; `name`, `labels`, `summary`, `attributes`, `name_embedding`.
- **CommunityNode**: cluster summary over entity nodes; built by clustering and LLM summarization.
- **SagaNode**: groups a sequence of episodes into a narrative chain.

### Edge types

- **EntityEdge**: the fact/relationship edge. Fields: `fact`, `fact_embedding`, `episodes` (provenance UUID list), `created_at`, `valid_at`, `invalid_at`, `expired_at`, `reference_time`, `attributes`.
- **EpisodicEdge**: episode → entity `MENTIONS` relationship.
- **CommunityEdge**: entity → community membership.
- **NextEpisodeEdge / HasEpisodeEdge**: saga sequencing and membership.

### Temporal field semantics

| Field | Meaning | Set where |
|---|---|---|
| `created_at` | when the record entered the system / knowledge time | ingestion (`now`) or bulk base time |
| `valid_at` | when the fact became true in the real world | LLM extraction, or `reference_time` fallback |
| `invalid_at` | when the fact stopped being true in the real world | LLM, or temporal invalidation against a newer fact |
| `expired_at` | when the system record was superseded | invalidation path (`now`) |
| `reference_time` | anchor used during extraction to interpret relative dates | caller-provided, defaults to current time |

> This is **four time axes**, not only two. The pair `valid_at`/`invalid_at` is the real-world interval; `created_at`/`expired_at` is the system-record interval.

## 5. Lifecycle

### Single episode — `add_episode`

1. `_resolve_request_scope(group_id)` returns request-scoped `(group_id, driver, clients)`.
2. Retrieve previous context: last 10 episodes with `valid_at <= reference_time`, unless `previous_episode_uuids` is supplied.
3. Build or load `EpisodicNode`; set `created_at=now`, `valid_at=reference_time`.
4. `extract_nodes` → `resolve_extracted_nodes`.
5. `_extract_and_resolve_edges` → `extract_edges` → merge materialized endpoints → `resolve_extracted_edges`.
6. `extract_attributes_from_nodes` (and edge attributes).
7. `_process_episode_data`: build `MENTIONS` edges, attach edge UUIDs to episode, optionally clear episode content, then `add_nodes_and_edges_bulk`.
8. Optionally create saga edges (`NEXT_EPISODE`, `HAS_EPISODE`).
9. Optionally update communities.
10. Return `AddEpisodeResults`.

The docstring explicitly recommends a background queue and **sequential await per episode**; concurrent/out-of-order ingestion breaks prior-context and invalidation reasoning.

### Bulk episode — `add_episode_bulk`

1. Build episodes, save them first via `add_nodes_and_edges_bulk` (phase 1).
2. Retrieve previous episode context.
3. Extract nodes/edges, dedupe nodes, dedupe edges, resolve nodes/edges.
4. Save resolved nodes/edges + episodic edges via a second `add_nodes_and_edges_bulk` (phase 2).
5. Optionally create saga chain.

### Direct triplet — `add_triplet`

1. Embed source/target nodes and edge if absent.
2. Resolve or load endpoint nodes.
3. Search edges between the same endpoint pair, plus broader existing edges.
4. `resolve_extracted_edge` (dedupe + temporal invalidation).
5. Bulk write nodes + edges. **No extraction; no real episode unless caller passes one.**

### Episode removal — `remove_episode`

1. Load episode, get edges from `episode.entity_edges`.
2. For each edge: remove episode UUID from `episodes`; delete edge if no other episode supports it, otherwise save updated edge.
3. Delete nodes mentioned only by this episode (`MENTIONS` count == 1).
4. Delete episode.
5. Rebuild summaries for surviving nodes; on failure restore old summary and log the exception.

## 6. Configuration

| Area | Source |
|---|---|
| Package | `pyproject.toml` (`graphiti-core 0.31.2`, Python `>=3.10,<4`) |
| Database | `Graphiti.__init__` args `uri`, `user`, `password`, `graph_driver`; env `NEO4J_*`, `FALKORDB_*` |
| LLM | default `OpenAIClient`; pluggable `llm_client` |
| Embedding | default `OpenAIEmbedder`; pluggable `embedder` |
| Reranker | default `OpenAIRerankerClient`; pluggable `cross_encoder` |
| Concurrency | `max_coroutines`, `SEMAPHORE_LIMIT` (default 20) |
| Retention | `store_raw_episode_content` (default `True`) |
| Group routing | `group_id`; default provider group if `None` |
| Server | `server/.env.example`, `docker-compose.yml`, `mcp_server/config/*.yaml` |

## 7. Authority and Trust Model

- `group_id` is the only partitioning key on reads and writes.
- `validate_group_id` allows only `[A-Za-z0-9_-]+` (empty allowed). This prevents Cypher/database-name injection via the group key, not unauthorized access.
- If `group_id` differs from the default database, `_resolve_request_scope` clones the driver with `database=group_id`; for Neo4j this selects a database, for FalkorDB a graph.
- `SECURITY.md` states: no authentication, no authorization, no caller isolation, no prompt-injection protection; all episodes in one `group_id` share the same trust; a newer episode can change or invalidate older facts.

## 8. Dependency and Test System

- Core dependencies: `pydantic`, `neo4j`, `openai`, `tenacity`, `numpy`, `python-dotenv`, `posthog`, `certifi`.
- Optional: Anthropic, Groq, Google GenAI, Kuzu, FalkorDB, VoyageAI, GLiNER2, Neo4j+OpenSearch, sentence-transformers.
- 81 `test_*.py`, 910 static `def test_`; temporal semantics have dedicated tests under `tests/utils/maintenance/`.
- **No tests executed in this run**; test existence is not treated as test validation.

## 9. What This Layer Does Not Establish

- It does not establish that the default search returns only valid facts.
- It does not establish that `group_id` enforces authorization.
- It does not establish that all drivers provide atomic writes.
- It does not establish that MCP acknowledges are durable.
- It does not establish any runtime performance or correctness claim.

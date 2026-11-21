# 01.4 Configuration & Testing

## 依赖与安装

- Python `>=3.10,<4`；包 `graphiti-core 0.31.2`。
- 必装：`pydantic>=2.11.5`、`neo4j>=5.26.0`、`openai>=1.91.0`、`tenacity>=9.0.0`、`numpy`、`python-dotenv`、`posthog`、`certifi>=2026.10.19`。
- 可选 extras：`anthropic`、`groq`、`google-genai`、`kuzu`（注释：上游停维护，将移除）、`falkordb`、`falkordblite`（pin `redis<9`）、`voyageai`、`gliner2`、`neo4j-opensearch`、`sentence-transformers`、`neptune`、`tracing`、`dev`。
- 安装：`make install` = `uv sync --extra dev`。

## 关键运行配置

| 配置 | 作用 | 证据 |
|---|---|---|
| `OPENAI_API_KEY` 等 | LLM / embedding provider | `mcp_server/config/*.yaml` |
| `GRAPHITI_TELEMETRY_ENABLED=false` | 关闭 PostHog 上报 | `AGENTS.md` |
| `CHUNK_SIZE_TOKENS` / `CHUNK_OVERLAP_TOKENS` | 长内容分块 | `content_chunking.py:215-243` |
| 估算阈值 `CHARS_PER_TOKEN = 4` | token 估算（跨 provider 保守值） | `content_chunking.py:36` |
| `group_id` | 分区 / 独立 graph | `graphiti.py:1304` |
| `strict_edge_types` | 签名不符时丢弃 vs 改名 | `edge_operations.py:671-714` |
| `update_communities` | 写入时是否增量更新社区 | `graphiti.py:1532` |
| `max_coroutines` | 并发上限 | `graphiti.py:1540` |
| DB 连接 | Neo4j / FalkorDB / Neptune / Kuzu | `driver/*.py` |

## 测试体系

| 命令 | 行为 |
|---|---|
| `make test` | `DISABLE_FALKORDB=1 DISABLE_KUZU=1 DISABLE_NEPTUNE=1 uv run pytest -m "not integration"` |
| CI 权威全绿门 | `unit_tests.yml`：`DISABLE_NEO4J=1 DISABLE_FALKORDB=1 DISABLE_KUZU=1 DISABLE_NEPTUNE=1` + `--ignore` 列表 |
| DB 集成门 | `database-integration-tests` job：Neo4j + FalkorDB，mock LLM，无需 API key |
| `pytest` | `asyncio_mode = auto`（`pytest.ini`） |
| 服务端 | `server/tests`、`mcp_server/tests`（`integration`，无 key 时 **exit 5 self-skip**，属预期） |

### 测试揭示的三条"非显然事实"（AGENTS.md）

1. **`make test` 隐式依赖 Neo4j**：它没设 `DISABLE_NEO4J`，无 DB 时会在连接重试上**挂起**（不是环境 bug）。
2. **`tests/test_add_triplet.py` 有 11 个既有失败**：其 mock embedder 未 stub `create_batch`，`zip(strict=True)` 抛错；**CI 从不运行该文件**。
3. **FalkorDB 并发断连**：同一连接并发查询（如 `Graphiti.search` 内 gather）会 "Connection closed by server"，本地混合检索建议用 Neo4j。

> 含义：仓库的"全绿"是**有条件全绿**（特定 DISABLE 组合 + ignore 列表）；不能把 CI 绿灯当作"所有测试都通过"。

## CI（15 个 workflow）

`ai-moderator` / `cla` / `codeql` / `issue-intake` / `lint` / `mcp-server-tests` / `pr-intake` / `release-graphiti-core` / `release-mcp-server` / `release-server-container` / `server-tests` / `socket-firewall-connectivity` / `stale` / `typecheck` / `unit_tests`。

- `typecheck`：pyright（`make lint` 含 ruff + pyright）；
- `codeql`：静态安全扫描；
- `cla`：CLA 门禁（HEAD commit 即 CLA 签名记录）。

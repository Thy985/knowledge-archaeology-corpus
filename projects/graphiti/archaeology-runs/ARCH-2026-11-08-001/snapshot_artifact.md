# Repository Snapshot · Zep Graphiti

## 1. 基本标识

| 字段 | 值 | 来源 |
|---|---|---|
| 仓库 | `getzep/graphiti` | `git remote get-url origin` |
| 包名 | `graphiti-core` | `pyproject.toml:2` |
| 版本 | **0.31.2** | `pyproject.toml:5` |
| HEAD | `747b480ec6b12ecb718f1e2bc588b20ee36ea9c2` | `git rev-parse HEAD` |
| Branch | `main` | `git branch --show-current` |
| Commit 时间 | 2026-11-06 04:49:50 +0000 | `git log -1` |
| Commit message | `@HhH666-ux has signed the CLA in getzep/graphiti#2129` | 同上（浅克隆仅 1 条） |
| 最新 Release | v0.31.2 @ 2026-10-20T17:25:21Z | GitHub API `/releases` |
| 许可 | Apache-2.0 | `LICENSE`, pyproject |
| Python | `>=3.10,<4` | pyproject `requires-python` |

**GitHub 元数据**（API `/repos/getzep/graphiti`，2026-11-08）：stars **32,625**、forks **3,351**、open issues **552**、size **16,510 KB**、created 2024-08-08、pushed 2026-11-06T04:49:51Z。

> 浅克隆限制：只有 1 条 commit，**无法回溯修复史/PR 历史**；失败与修复证据只能来自代码注释中的 issue 编号（#1659 / #1676 / #1758）。

## 2. 规模（实测）

| 指标 | 值 | 命令 |
|---|---|---|
| 文件（不含 `.git`） | **386** | `os.walk` 排除 `.git`（复核值；初版 424 口径不可复现） |
| Python 文件 | **300** | `os.walk` 排除 `.git` |
| Python 行数 | **76,764** | `os.walk` 排除 `.git` 逐文件计数（复核值；初版 77,773 不可复现） |
| 测试文件 | 83 | `test_*.py` / `*_test.py` |
| CI workflow | **15** | `.github/workflows/*.yml` |
| 子项目 | 3：`graphiti_core`（库）/ `server`（REST）/ `mcp_server` | 目录 |
| spec 设计文档 | 7 | `spec/*.md` |
| Docker Compose | 2（`docker-compose.yml` / `docker-compose.test.yml`） | 根目录 |

## 3. 包结构与代码分布

| 模块 | 行数 | 职责 |
|---|---:|---|
| `graphiti_core/driver/` | 15,724 | 4 后端（Neo4j / FalkorDB / Kuzu(弃用) / Neptune）+ operations ABC + query executor |
| `graphiti_core/utils/` | 6,002 | 维护管线（node/edge/community operations）、chunking、bulk、删除 |
| `graphiti_core/llm_client/` | 4,482 | LLM 客户端 + `LLMRuntime` 路由 |
| `graphiti_core/search/` | 3,711 | 混合检索（BM25 / cosine / BFS / RRF / MMR / cross-encoder） |
| `graphiti_core/prompts/` | 3,038 | 抽取/去重/总结提示词 |
| `graphiti_core/models/` | 741 | 纯 Pydantic 数据模型 |
| `graphiti_core/namespaces/` | 733 | `graphiti.nodes.entity.save(...)` 式命名空间 |
| `graphiti_core/embedder/` | 442 | OpenAI / Gemini / Voyage / 自建 |
| `graphiti_core/cross_encoder/` | 399 | BGE / OpenAI / Gemini 重排 |
| `graphiti_core/telemetry/` | 126 | OpenTelemetry |

**核心大文件**（`wc -l`）：`graphiti.py` 2,251 / `search_utils.py` 2,075 / `nodes.py` 1,166 / `edge_operations.py` 1,153 / `edges.py` 1,051 / `node_operations.py` 1,295 / `search.py` 891 / `content_chunking.py` 826 / `mcp_server.py` 1,379。

**测试大文件**：`tests/test_graphiti_mock.py` 2,578 / `tests/utils/maintenance/test_edge_operations.py` 1,948 / `test_node_operations.py` 1,257 / `test_add_triplet.py` 1,021。

## 4. 入口

| 入口 | 位置 | 说明 |
|---|---|---|
| 库入口 | `graphiti_core.Graphiti`（`graphiti.py:209`） | `add_episode` / `add_episode_bulk` / `search` / `search_` / `build_communities` |
| REST | `server/graph_service/main.py`，uvicorn | 10 端点：ingest 6 + retrieve 4 |
| MCP | `mcp_server/src/graphiti_mcp_server.py` | 12 个 `@mcp.tool()` |
| CLI/示例 | `examples/`（quickstart、langgraph-agent、podcast 等 12 个） | 非生产入口 |

## 5. 核心数据结构

| 类型 | 关键字段 | 位置 |
|---|---|---|
| `Node`（ABC） | `uuid`, `name`, `group_id`, `labels`, `created_at` | `nodes.py:93` |
| `EpisodicNode` | `source`, `source_description`, `content`, `valid_at`, `entity_edges`, `episode_metadata` | `nodes.py:318` |
| `EntityNode` | `name_embedding`, `summary`, `attributes` | `nodes.py:499` |
| `CommunityNode` | `name_embedding`, `summary` | `nodes.py:727` |
| `SagaNode` | `summary`, `first/last_episode_uuid`, `last_summarized_at` | `nodes.py:907` |
| `Edge`（ABC） | `uuid`, `group_id`, `source/target_node_uuid`, `created_at` | `edges.py:49` |
| `EntityEdge` | `name`, `fact`, `fact_embedding`, `episodes`, `valid_at`, `invalid_at`, `expired_at`, `reference_time`, `attributes` | `edges.py:263` |
| `EpisodicEdge` / `CommunityEdge` / `HasEpisodeEdge` / `NextEpisodeEdge` | 关系骨架 | `edges.py:143/580/694/827` |

## 6. 状态模型（时序）

- **双时间语义**：`valid_at`（事实何时为真）/ `invalid_at`（何时被推翻）/ `expired_at`（何时被系统标记过期，逻辑删除）/ `created_at`（写入时刻）。
- **历史保留**：失效不删除（README "invalidated — not deleted"），查询可按时间点回溯。
- **Episode 为 provenance 根**：`EntityEdge.episodes[]` 记录支持该事实的 episode；删除按引用计数处理（`graphiti.py:2174-2200`）。

## 7. 测试体系

| 层 | 目录 | 说明 |
|---|---|---|
| 核心单测 | `tests/`（含 `utils/maintenance`、`driver`、`llm_client`、`search`、`prompts`、`evals`、`intake`） | mock LLM / mock embedder |
| 集成 | `*_int.py` + `@pytest.mark.integration` | 需 Neo4j / FalkorDB（`docker-compose.test.yml`） |
| 服务端 | `server/tests` | 端到端，需 `OPENAI_API_KEY`，缺失时 self-skip（exit 5 预期） |
| MCP | `mcp_server/tests` | 独立套件 |

**已知门禁事实**（`AGENTS.md`）：
- `make test` **隐式要求** `bolt://localhost:7687` 的 Neo4j（无 DB 会挂起在连接重试）；
- 权威全绿无 DB 门禁是 CI 的 `DISABLE_NEO4J=1 …` + `--ignore` 列表；
- `tests/test_add_triplet.py` 有 **11 个既有失败**（mock embedder 未 stub `create_batch`，`zip(strict=True)` 抛错），CI 从不运行该文件；
- FalkorDB 驱动在并发查询同一连接时会断连（`Graphiti.search` 的 gather 触发）。

## 8. 配置与治理

| 项 | 内容 |
|---|---|
| 环境变量 | `OPENAI_API_KEY`、`GRAPHITI_TELEMETRY_ENABLED`、`CHUNK_*`（见 `content_chunking.py`）、DB 连接串 |
| 构建/检查 | `make install`(uv) / `make format`(ruff) / `make lint`(ruff+pyright) / `make test` / `make check` |
| 依赖 | pydantic≥2.11.5、neo4j≥5.26、openai≥1.91、tenacity、numpy、posthog、certifi；optional: anthropic/groq/google-genai/kuzu/falkordb/neptune/opensearch/sentence-transformers/tracing |
| 许可治理 | Apache-2.0 + **CLA**（`signatures/version1/cla.json`，`cla.yml` workflow；HEAD 即 CLA 签名 commit） |
| 安全治理 | `SECURITY.md` 显式声明参考实现**无认证/无授权/无隔离/无注入防护** |
| 贡献治理 | `AGENTS.md`（结构/测试/提交/PR 规则）、`CONTRIBUTING.md`、`CODE_OF_CONDUCT.md` |

## 9. 权限与治理机制（实测）

| 机制 | 是否存在 | 证据 |
|---|---|---|
| 多租户分区 | ✅ 逻辑分区 `group_id` | `nodes.py:96` `group_id: partition of the graph` |
| 分区→独立数据库 | ✅ FalkorDB 每 group 一个 graph，`driver.clone(database=gid)` | `decorators.py:56-70`、`graphiti.py:1325-1330` |
| 跨 group 查询扇出 | ✅ 逐 group 执行后 merge | `decorators.py:73-110` |
| 认证 | ❌ 开源参考服务无 | `SECURITY.md` |
| 授权/ABAC | ❌ 开源侧无（托管 Zep 侧提供） | `SECURITY.md`、README `Zep vs Graphiti` |
| 审计日志 | ❌ 开源侧无；仅 OTEL tracing | `graphiti_core/telemetry/` |
| 调用方隔离 | ❌ 明确声明无 | `SECURITY.md` |
| Prompt 注入防护 | ❌ 明确声明无 | `SECURITY.md` |
| 同 group 内容信任分级 | ❌ 全部 episode 同等信任 | `SECURITY.md` |

> **能力边界**：开源仓库 = 图构建/检索框架 + 参考服务；企业治理（用户/线程/多租户/ABAC/审计/SLA/Konig 图数据库）= **Zep 托管侧**，不在本仓库实现。按 `oss-validation-capability-boundary` 经验，不把托管能力记到开源仓库。

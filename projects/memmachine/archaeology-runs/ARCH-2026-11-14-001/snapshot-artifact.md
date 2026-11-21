# Repository Snapshot — MemMachine (ARCH-2026-11-14-001)

- **run_id**: ARCH-2026-11-14-001
- **repository**: https://github.com/MemMachine/MemMachine.git
- **branch**: main
- **commit SHA (HEAD)**: `377510d436dc81c3651502f2d5e36ff245650ce7`
- **commit time**: `2026-11-02T11:35:15-08:00`
- **commit message**: `[contributor setup 1] Read paid-API test credentials from test-only variables (#1869)`
- **clone**: `git clone --depth 1 --single-branch --branch main … repo`
- **license**: Apache-2.0
- **GitHub metadata** (2026-11-14 query): 3,087 stars; 35,260 KB; created 2025-08-15; pushed 2026-11-12T14:10:08Z; not archived
- **mode**: initial（非 refresh）
- **read-only**: 本 Snapshot 未修改仓库任何文件；未运行测试（静态取证）

## 1. 规模与语言构成

| 指标 | 值 | 口径 |
|---|---:|---|
| 文件总数 | 876 | `find . -path ./.git -prune -o -type f -print \| wc -l` |
| 目录体积 | 105M | `du -sh .`（含 `.git`） |
| Python 文件 | 496 | `find … -name '*.py'` |
| Python 行数 | 151,515 | 全部 `.py` 的 `wc -l` 合计 |
| `packages/` Python 行数 | 118,114 | 任务摘要既有统计（packages 子树） |
| 测试文件 | 121 | `packages/**/test_*.py` |
| 测试函数 | 2,191（静态计数） | `grep -rhE '^\s*(async )?def test_' packages --include='test_*.py'`；其中 `server_tests` 1,856 |
| Markdown/MDX | 60 `.md` + 89 `.mdx` | 文档站（Mintlify 形态） |
| TypeScript | 34 `.ts` | `packages/ts-client` 及站点配置 |
| CI YAML | 28 `.yml` | `.github/workflows/` 等 |

> 统计命令均在浅克隆目录执行；测试函数数只覆盖 `packages/`，不含 `evaluation/`、`examples/`、`integrations/`。

## 2. 顶层布局

```
AGENTS.md                     # 面向 AI/贡献者的工程约定（uv/ruff/ty/pytest）
alembic.ini                   # DB migration 配置
assets/                       # README 图片等
build-docker.sh
design/                       # 设计决策文档（当前仅 segment_store_shared_tables.md）
deployments/helm/             # Helm chart
docs/                         # Mintlify 文档站（core_concepts / api_reference / platform / profiler_engine …）
evaluation/                   # 评测脚本/资产
examples/
integrations/                 # 8+ 框架适配
packages/
  client/                     # Python SDK
  common/                     # 共享 API 模型（pydantic）
  meta/                       # 汇总安装包
  server/                     # 主体实现
  skills/                     # memmachine-memory skill（agents/references）
  ts-client/                  # TypeScript SDK
sample_configs/               # event / declarative / cpu / gpu / nebula 样例
tools/
pyproject.toml                # uv workspace: client/common/server
uv.lock
docker-compose.yml            # postgres + qdrant/neo4j + memmachine
Dockerfile
README.md / USAGE.md
GOVERNANCE.md / CONTRIBUTING*.md / CODE_OF_CONDUCT.md / SECURITY.md / STYLE_GUIDE.md
```

**证据**：`ls` 顶层；`pyproject.toml` 的 `[tool.uv.workspace] members`；`docker-compose.yml` 服务定义。

## 3. 语言、运行时与依赖

- **Python**: `requires-python >= 3.12`（`packages/server/pyproject.toml`）；common 支持 3.10+；CI 覆盖 3.12/3.14。
- **包管理**: `uv` workspace（根 `pyproject.toml`），`uv sync`；`uv.lock` 入库并由 CI `check-uvlockfile.yml` 校验。
- **服务端核心依赖**（`packages/server/pyproject.toml`）:
  `fastapi>=0.116.1`、`fastmcp>=3.2.0`、`uvicorn>=0.35.0`、`starlette>=0.48.0`、`pydantic>=2.11.7`、
  `sqlalchemy[asyncio]>=2.0.43`、`asyncpg>=0.31.0`、`aiosqlite>=0.20.0`、`alembic>=1.17.1`、`pgvector>=0.4.2`、
  `neo4j>=5.28.2`、`sqlite-vec>=0.1.9`、`usearch>=2.25.1`、`rank-bm25>=0.2.2`、`nltk>=3.9.4`、
  `openai>=2.21.0`、`instructor[bedrock]>=1.12.0`、`langchain-aws>=0.2.33`、`cohere>=5.20.0`、
  `boto3>=1.40.40`、`prometheus-client>=0.22.1`、`json-repair>=0.54`、`psutil>=7.1.0`。
- **可选后端依赖**: `qdrant` extra（`qdrant-client`、`grpcio`）、`nebula` extra（`nebula5-python>=5.2.1,<5.3`，注释说明 5.3 移除公开 async client）、`milvus`、`hnswlib`、`gpu`（`sentence-transformers`）、`litellm`。
- **测试依赖**: `pytest>=9.0.3`、`pytest-asyncio`、`testcontainers[neo4j,postgres,qdrant]`、`httpx`、`pytest-xdist`、`pytest-cov`。
- **静态检查**: `ruff==0.16.8`（check + format）、`ty~=0.0.38`（`ty check packages`）、`complexipy`。

**证据**：`packages/server/pyproject.toml` `[project] dependencies`、`[project.optional-dependencies]`、`[dependency-groups] dev`；根 `pyproject.toml` `[tool.ruff]`。

## 4. 入口与运行形态

| 入口 | 命令/符号 | 行为 |
|---|---|---|
| HTTP 服务 | `memmachine-server = memmachine_server.server.app:main` | 解析 `--stdio` / `--version` / `--with-config-api`；默认 `uvicorn.run("...app:app")` |
| MCP stdio | `memmachine-mcp-stdio = memmachine_server.server.mcp_stdio:main` | MCP 标准输入输出模式 |
| MCP HTTP | `memmachine-mcp-http = memmachine_server.server.mcp_http:main` | 挂载于 FastAPI `/mcp` |
| 配置向导 | `memmachine-configure = ...installation.memmachine_configure:main` | 安装/配置辅助 |
| NLTK 资源 | `memmachine-nltk-setup = memmachine_server:setup_nltk` | 下载 NLTK 数据 |
| Python SDK | `packages/client/src/memmachine_client/...` | `Memory` 客户端（`client/memory.py` 1,847 行） |
| TS SDK | `packages/ts-client` | 独立 npm 包 |
| Docker | `Dockerfile` + `docker-compose.yml` + `deployments/helm/` | compose 默认 `postgres` + `qdrant`（`event` profile）；`declarative` profile 启 `neo4j` |

**FastAPI 组装**（`server/app.py`）:
- `MemMachineAPI(FastAPI)`：`mount("/mcp", mcp_app)`，`load_v2_api_router(...)`，可选 `config_router`（`MEMMACHINE_CONFIG_API`）。
- 中间件：`AccessLogMiddleware`、`RequestMetricsMiddleware`。
- 启动：`mcp_http_lifespan`；`initialize_resource()` 构建 `ResourceManagerImpl(Configuration)` 与 `MemMachine`。
- 配置查找顺序：`MEMORY_CONFIG` → `~/.config/memmachine/cfg.yml` → `./cfg.yml`；找不到直接 `FileNotFoundError`。
- worker 数：`MEMMACHINE_WORKERS`，默认 1；多 worker 时确保 `PROMETHEUS_MULTIPROC_DIR` 存在并清理旧 `.db`。

**证据**：`packages/server/pyproject.toml [project.scripts]`；`server/app.py` 77-235 行。

## 5. 核心模块地图（server）

```
packages/server/src/memmachine_server/
├── main/memmachine.py                 # 顶层编排：session、add_episodes、query_search、feature API
├── server/
│   ├── app.py                         # FastAPI 组装/入口
│   ├── middleware.py                  # 访问日志 + 请求指标
│   ├── diagnostics.py                 # SIGUSR1 traceback dump
│   ├── mcp_http.py / mcp_stdio.py     # MCP 进程入口
│   └── api_v2/
│       ├── router.py                  # REST v2（projects/memories/search/list/health）
│       ├── config_router.py           # 配置管理 API（需 MEMMACHINE_CONFIG_API）
│       ├── mcp.py                     # MCP tools + org/proj/user header 上下文
│       └── service.py                 # REST → MemMachine 适配
├── episodic_memory/
│   ├── episodic_memory.py             # STM + LTM facade
│   ├── episodic_memory_manager.py     # session → instance 生命周期
│   ├── instance_lru_cache.py          # 有界实例缓存 + 延迟 close
│   ├── short_term_memory/             # deque + 后台摘要 consolidator
│   ├── long_term_memory/              # backend 分派（declarative / event）
│   ├── declarative_memory/            # Episode + Derivative + 图边
│   └── event_memory/                  # segment → derivative → vector + segment store
├── semantic_memory/
│   ├── semantic_memory.py             # 多 set 并行检索
│   ├── semantic_ingestion.py          # LLM ADD/DELETE + citation + consolidation
│   ├── semantic_session_manager.py    # set_id/set type/category/tag
│   ├── cluster_splitter.py            # 检索结果簇切分（replay/idempotent）
│   ├── storage/                       # Neo4j / PGVector / VectorStore 三实现
│   ├── config_store/                  # set 配置（SQLAlchemy + cache）
│   └── cluster_store/                 # cluster 状态（SQL / in-memory）
├── common/
│   ├── configuration/                 # pydantic 配置 + 默认解析 + auto-disable
│   ├── resource_manager/              # DB/embedder/LM/reranker 生命周期
│   ├── episode_store/                 # episode 持久化 + count cache
│   ├── session_manager/               # session 元数据 + 短期摘要持久化
│   ├── vector_store/                  # sqlite/sqlite-vec/qdrant/milvus + engine
│   ├── vector_graph_store/            # neo4j / nebula
│   ├── embedder/ language_model/ reranker/ filter/ payload_codec/ sql_lease_lock/
│   └── api/                           # spec.py 1,750 行、doc.py 1,370 行（共享模型）
└── retrieval_agent/agents             # agentic 检索（query decomposition/多跳）
```

**最大文件**（`wc -l`）:
`common/vector_graph_store/nebula_graph_vector_graph_store.py` 2,081；
`client/memory.py` 1,847；`main/memmachine.py` 1,813；`common/api/spec.py` 1,750；
`semantic_memory/storage/neo4j_semantic_storage.py` 1,522；
`episodic_memory/event_memory/segment_store/sqlalchemy_segment_store.py` 1,432；
`common/vector_graph_store/neo4j_vector_graph_store.py` 1,377；
`common/api/doc.py` 1,370；`server/api_v2/router.py` 1,114；`common/vector_store/sqlite_vector_store.py` 1,067。

## 6. 核心数据结构

| 结构 | 字段/语义 | 证据 |
|---|---|---|
| `Episode`（domain） | `uid`、`content`、`session_key`、`created_at`、`producer_id`、`producer_role`、`produced_for_id`、`episode_type`、`content_type`、`filterable_metadata`、`metadata` | `common/episode_store/episode_model.py` 45-60 |
| `EpisodeEntry`（写入） | 无 `session_key`；`uid` 默认 `uuid4`；`created_at` 可空，由服务端补 | 同上 22-35 |
| `Segment`（event） | `uuid`、`event_uuid`、`index`、`offset`、`timestamp`、`context`、`block`、`properties` | `episodic_memory/event_memory/data_types.py` 126-158 |
| `Derivative`（event） | `uuid`、`segment_uuid`、`timestamp`、`context`、`block`、`properties` | 同上 160-190 |
| `Derivative`（declarative） | 独立 dataclass（节点/边形态） | `declarative_memory/data_types.py` 55 |
| `SemanticFeature` | `set_id`、`category`、`tag`、`feature_name`、`value`、`metadata.id`、`metadata.citations` | `semantic_memory/semantic_model.py` 60-82 |
| `SemanticCommand` | `command`（ADD/DELETE）、`feature`、`tag`、`value`；剥离 NUL 字节 | 同上 18-42 |
| `SessionInfo` | `configuration`、`description`、`user_metadata`、`episode_memory_conf`、`status` | `common/session_manager/session_data_manager.py` 70-77 |
| `SegmentStore registry` | `segment_store_pt`（logical key + `incarnation` UUID + codec）、`segment_store_sg`、`segment_store_dv_ln`、`segment_store_gc` | `design/segment_store_shared_tables.md` §Design |

## 7. 状态与生命周期

### 7.1 Session / Project
- `_SessionData` 将 `org_id/project_id` 拼成 `session_key = f"{org_id}/{project_id}"`（`api_v2/service.py` 39-46）。
- Session 状态仅 `active` / `delete`（`SessionDataManager.SessionStatus`，`session_data_manager.py` 63-67）。
- `create_or_validate_session`：不存在则建；存在则校验配置一致（`session_data_manager_sql_impl.py` 233）。
- `delete_session`：不是物理删表；走 partition drop（event backend 是删 collection + segment partition）。
- `instance_cache_size`：`sample_configs/configuration.event.yml` 默认 `0`，即不复用空闲 `EpisodicMemory` 实例；`MemoryInstanceCache` 有 `capacity` + `max_lifetime` + 引用计数 + 延迟 close。

### 7.2 Short-term memory
- `deque` 存近期 episode，容量 `message_capacity`（样例 500）。
- 超容量触发 `_do_evict`：先弹已摘要旧条目；仍满则整批交给 `ShortTermMemoryConsolidator.summarize`。
- consolidator 是后台 task + 动态批处理；`summary` 与 `summary_len` 由 `AsyncRWLock` 保护。
- 摘要失败路径：上下文超限时递归二分（单条 episode 按字符中点拆半）；其它 `ExternalServiceAPIError/ValueError/RuntimeError` 只记录并返回旧 summary。
- 摘要持久化：`SessionDataManager.save_short_term_memory(session_key, summary, last_seq, episode_num)`。

**证据**：`short_term_memory/short_term_memory.py` 105-160、189-243、468-577、613-679；`session_data_manager.py` 98-110。

### 7.3 Long-term memory
- `LongTermMemory` 是 facade，按 `params.backend` 分派：`declarative` / `event`。
- **declarative**：vector_graph_store + Episode/Derivative 节点 + `DERIVED_FROM` 边；检索走 derivative → episode → 邻域扩展 → reranker。
- **event**：`segment_store`（关系库）+ `vector_store`（Qdrant 等）；写入五阶段：segment → derive → embed → segment store → vector upsert。
- **删除**：event 先删 vector 再删 segment；declarative 走 `delete_episodes`。
- **分区**：event backend 每个 session 一个 VectorStore collection + SegmentStore partition（`drop_session_partition`）。

**证据**：`long_term_memory/long_term_memory.py` 109-240；`event_memory/event_memory.py` 223-317、680-725；`declarative_memory/declarative_memory.py` 99-212、313-500。

### 7.4 Semantic / Profile memory
- 写入是 **异步排队 + 后台 ingestion**：`add_message_to_sets` 仅把 history_id 注册进 set（`is_ingested=False`）。
- `IngestionService` 每 set 每轮拉最多 5 条未 ingest message，按 category 并发。
- 每条 message 取该 set/category 已有 feature（`max_features_per_update=50`），LLM 输出 ADD/DELETE。
- ADD：value embedding + `add_feature`；若有 `citation_id` 则 `add_citations([message.uid])`。
- DELETE：按 set/category/feature/tag 过滤删除。
- consolidation：按 tag 分组，达到 `consolidated_threshold=20` 才触发；先 `delete_features`，再并发 `add_feature`；被删 feature 的 citations 合并后挂到每个新 feature。

**证据**：`semantic_ingestion.py` 125-347、348-404、466-543；`semantic_memory.py` 234-306。

## 8. 配置体系

- 配置载体：YAML（`Configuration.load_yml_file`）+ 环境变量替换（仅 `password` / `api_key` 支持 `$ENV`，`user` / `db_name` 按字面值）。
- 顶层配置键：`logging`、`episode_store`、`episodic_memory`、`retrieval_agent`、`semantic_memory`、`session_manager`、`prompt`、`resources`、`server`。
- 默认关闭策略：
  - `Configuration.semantic_memory` 默认 `enabled=False`；`semantic_memory:` 为空/null 也视为关闭。
  - `MemMachine._initialize_default_episodic_configuration` 默认 `episodic_memory.enabled=False`，LTM/STM 均关闭。
  - `semantic_memory` 若 OpenAI embedder/LM 凭证为空且无 `base_url`，加载时自动关闭并 warning。
  - `episodic_memory` 同样有 OpenAI 凭证缺失的自动关闭逻辑；LTM/STM 都关时整体关。
- LTM 后端必填：
  - declarative：`embedder` + `reranker` + `vector_graph_store`，任一缺失 → 只 warning 并关闭 LTM。
  - event：`vector_store` + `segment_store`，缺任一 → 关闭 LTM。
- STM 默认 `llm_model="gpt-4.1"`（`main/memmachine.py` 255-266）。
- 样例 `configuration.event.yml`：`backend=event`、Qdrant `v1.19.1`、Postgres `pgvector/pgvector:pg18`、`message_capacity=500`、`instance_cache_size=0`、`rrf-hybrid`（identity + BM25）、embedding `text-embedding-3-small` 1536 维、LLM `openai-responses` `gpt-6-luna`。
- compose profile：默认 `event`（postgres + qdrant）；`declarative` 启 `neo4j:5.23-community`。

**证据**：`common/configuration/__init__.py` 396-500、540-600；`sample_configs/configuration.event.yml`；`docker-compose.yml`。

## 9. 测试体系

- 目录：`packages/server/server_tests/memmachine_server/...`、`packages/client/client_tests/`、`packages/common/common_tests/`。
- 覆盖域（抽样自 121 个测试文件）：
  - API v2（`server_tests/.../server/api_v2`）
  - 配置（`common/configuration/*`）
  - episode store（`common/episode_store/*`，含 UUID 契约、启动迁移）
  - vector store（sqlite / sqlite-vec / qdrant / milvus / hnswlib / usearch）
  - vector graph store（neo4j / nebula）
  - semantic storage（Neo4j / PGVector / VectorStore）
  - resource manager / embedder / LM / reranker
  - session manager 迁移、SQL lease lock（Postgres）
  - client `test_memory.py`、`test_integration_complete.py`
- 测试基础设施：`testcontainers` 起 Neo4j/Postgres/Qdrant；`pytest-asyncio`；`pytest-xdist`。
- CI：`pytest.yml`、`pytest-integration.yml`、`test-server-package.yml`、`test-python-client-package.yml`、`lint.yml`（ruff + ty 多 Python 版本）、`installation-test.yml`（含 Ollama 本地模型安装路径）、`security-audit.yml`、`sbom.yml`。

**证据**：测试目录枚举；`.github/workflows/` 列表与关键步骤。

## 10. 权限、租户与治理

### 10.1 运行时权限
- **REST API 未实现鉴权**：`api_v2/router.py` 的端点只依赖 `get_memmachine`，没有 token/API-key 校验；`org_id/project_id` 由请求体 `spec` 提供。
- **MCP 身份来自 header**：`ParamsContextMiddleware` 读取 `org-id` / `proj-id` / `user-id` 并放入 `ContextVar`，没有签名或令牌校验；缺省为空字符串（`mcp.py` 240-300）。
- **租户隔离是命名空间隔离**：`session_key = org/project`；event backend 每 session 一个 collection + partition；semantic set_id 由 `SemanticSessionManager` 生成。隔离依赖调用方正确传 ID，而非认证。
- **配置 API** 需 `MEMMACHINE_CONFIG_API` 环境变量才挂载；`config_service.py` 会把 `api_key` 转 `SecretStr`。
- **出站凭证**：OpenAI / Bedrock / Cohere / Anthropic 等 provider 的 `api_key` 从配置/env 读取；`password` / `api_key` 支持 `$ENV_NAME` 替换。

### 10.2 项目治理
- `GOVERNANCE.md`：User → Contributor → Maintainer → Technical Council；**Lazy Consensus**，提案投票 72 小时；强反对则公开讨论，无法共识由 Project Leads 决定。
- `SECURITY.md`：GitHub private vulnerability reporting；仅最新 major 提供安全补丁；协调披露。
- `CONTRIBUTING.md` / `CONTRIBUTING-CORE.md` / `CONTRIBUTING-DOCS.md`：分层贡献规则。
- `STYLE_GUIDE.md`、`CODE_OF_CONDUCT.md`。
- CI 治理：ruff/ty/pytest/installation/security-audit/SBOM；`build-python.yml`、`docker-image.yml`、`pypi-publish.yml`。

> **事实边界**：以上是仓库声明的治理流程，不等于运行时已强制执行；REST/MCP 的鉴权缺口是代码事实。

## 11. 外部依赖与服务拓扑

| 类别 | 依赖 | 用途 |
|---|---|---|
| 关系库 | PostgreSQL 18 + pgvector | episode store、session、semantic、segment store |
| 向量库 | Qdrant v1.19.1 | event backend 的 segment/derivative 向量 |
| 图库（可选） | Neo4j 5.23 community / Nebula 5.2.x | declarative backend 的 Episode/Derivative 图 |
| 本地嵌入式 | SQLite / sqlite-vec / usearch / hnswlib | 单机/测试/轻量部署 |
| LLM | OpenAI（responses/chat）、AWS Bedrock、LiteLLM、Cohere | 短期摘要、semantic 抽取/合并、retrieval agent、reranker |
| Embedding | OpenAI、Bedrock、SentenceTransformers、Cohere | episode/feature 向量 |
| Reranker | identity、BM25、Cohere、cross-encoder、RRF hybrid | 检索重排 |
| 协议/框架 | MCP（fastmcp）、LangChain/LangGraph、CrewAI、LlamaIndex、Strands、n8n、Dify、FastGPT、OpenClaw | 对外集成 |
| 可观测性 | prometheus-client、访问日志、请求指标、SIGUSR1 dump | 运行诊断 |
| 部署 | Docker / compose / Helm | 自托管 |

**证据**：`docker-compose.yml`；`packages/server/pyproject.toml`；`integrations/` 目录；`sample_configs/configuration.event.yml`。

## 12. 已发现的关键实现事实（供阶段④取证入口）

以下仅为 Snapshot 阶段确认的代码事实，不做升维：

1. **顶层 `add_episodes` 并行三写 + 部分失败不可修复**：`episode_storage.add_episodes`、`episodic_memory.add_memory_episodes`、`semantic_session_manager.add_message` 用 `asyncio.gather(return_exceptions=True)`，随后统一抛错；代码留有 `# TODO(#1738): Track write intent so partial failures across these parallel backends can be detected and repaired.`（`main/memmachine.py` 778-879）。
2. **EpisodicMemory 内部并发写未用 `return_exceptions`**：STM/LTM 任一失败即抛，另一侧可能已成功（`episodic_memory.py` 209-244）。
3. **event backend 写入顺序**：先 segment store，后 vector store；vector upsert 失败会留下无向量的孤儿 segment（`event_memory.py` 223-317）。
4. **event backend 删除顺序**：先 vector，后 segment；与写入相反（`event_memory.py` 680-725）。
5. **declarative backend 的“原文保留 + 派生检索”**：原始 `Episode_<session>` 与 `Derivative_<session>` 两个 collection，`DERIVED_FROM` 边回指；检索命中 derivative 后反查 episode，再按 `expand_context` 前后扩邻域，最后 reranker 对整段打分（`declarative_memory.py` 99-212、313-500）。
6. **semantic 写入是 LLM 抽取 + citation**：ADD 时 `add_citations([message.uid])`；DELETE 按 tag/feature 过滤；consolidation 先删后加，并把被删 feature 的全部 citations 合并挂到每个新 feature（`semantic_ingestion.py` 299-347、466-543）。
7. **API 默认 `score_threshold=-inf`**：`_search_target_memories` 在未指定阈值时显式传 `-float("inf")`（`service.py` 100-135）；而 `LongTermMemory.search_scored` 文档声称要避免 `-inf` sentinel（`long_term_memory.py` 254-266），`_score_passes_threshold` 对 lower-is-better 用 `score <= threshold`（同文件 462-475）。**cosine 下无影响；euclidean 且无 reranker 时会把全部结果过滤掉**。
8. **README 与实现的拓扑偏差**：README 表述 Episodic→图数据库、Profile→SQL；实际 `configuration.event.yml` 默认 event backend 是 Qdrant + Postgres，declarative 才用 `vector_graph_store`（Neo4j/Nebula）。

> 以上事实均带文件与行号，阶段④的 Evidence 必须复用同一坐标，不能以 README 或文档站叙述替代实现事实。

## 13. Snapshot 结论与风险

- **可考古性**: 通过。核心实现（1.8k 行编排器 + 862 行 semantic + 551 行 episodic + 双后端）均可读，测试体系完整，配置/部署/治理文件齐全。
- **不适合直接运行验证**: 默认部署需要 Postgres + Qdrant + LLM/embedding 凭证；本轮未执行 `uv sync`/`pytest`/compose，所有结论为静态代码事实（S3 级，非 S4/S5）。
- **主要认知风险**:
  1. README/文档站是营销级描述，多处宽于实现；阶段④必须以代码为准。
  2. 双后端（declarative/event）语义不完全等价：context 单位、删除顺序、失败面不同；Flow Atlas 必须分开画，不能合并。
  3. 默认关闭 episodic/semantic：未配置时“记忆层存在但未启用”，任何运行结论必须写明配置前提。
  4. 鉴权缺失是代码事实；但“未鉴权”不等于“可被公网利用”，部署边界需单独核实，不能升维为安全结论。

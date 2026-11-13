# Project Layer · Architecture

> 本文件只回答“MemMachine 是什么、怎么组装、在哪里运行”。实现机制与判断在 `02_engineering_knowledge/`，不在此处升维。
> 证据编号对应 `02_engineering_knowledge/evidence-index.md`。

## 1. 一句话定位

MemMachine 是一个**多租户的 agent memory 服务**：对外暴露 REST v2 与 MCP 两种协议，内部把记忆拆成 **episodic memory**（短期摘要 + 长期对话记忆）、**semantic/profile memory**（LLM 抽取的结构化 feature）与 **episode store**（原始消息），并允许长期记忆在 **declarative（图）** 与 **event（segment + 向量）** 两种后端之间切换（EV-001、EV-005、EV-013、EV-040）。

## 2. 进程与部署形态

```
                    ┌──────────────────────────────┐
   REST /v2  ───────▶│  FastAPI app (server/app.py) │
   MCP /mcp  ───────▶│  ├─ api_v2/router.py         │
                    │  ├─ api_v2/config_router.py  │  (仅 MEMMACHINE_CONFIG_API=1)
                    │  ├─ api_v2/mcp.py (fastmcp)  │
                    │  └─ middleware (log/metrics) │
                    └──────────────┬───────────────┘
                                   ▼
                    ┌──────────────────────────────┐
                    │  MemMachine (main/memmachine)│
                    │  ├─ SessionDataManager       │
                    │  ├─ EpisodicMemory per key   │
                    │  ├─ SemanticService          │
                    │  └─ ResourceManager          │
                    └───────┬──────────┬───────────┘
                            ▼          ▼
                 ┌──────────────┐  ┌─────────────────────┐
                 │ Postgres     │  │ Qdrant / Neo4j /    │
                 │ (episode,    │  │ Nebula / SQLite     │
                 │  session,    │  │ (vector / graph)    │
                 │  segment,    │  └─────────────────────┘
                 │  semantic)   │
                 └──────────────┘
```

- **单进程默认**：`uvicorn.run(..., workers=MEMMACHINE_WORKERS)`，默认 1；多 worker 时自动管理 `PROMETHEUS_MULTIPROC_DIR`（EV-006、EV-009）。
- **MCP 两种传输**：stdio 进程入口 `memmachine-mcp-stdio`，HTTP 入口挂载在 FastAPI 的 `/mcp`（EV-006、EV-010）。
- **compose 拓扑**：`postgres: pgvector/pgvector:pg18` 为默认关系库；`qdrant/qdrant:v1.19.1` 属于 `event` profile；`neo4j:5.23-community` 属于 `declarative` profile；`memmachine` 服务依赖三者（EV-073、EV-074）。
- **Helm**：`deployments/helm/` 提供 chart；`build-docker.sh` 与 `docker-image.yml` 产出 CPU/GPU 镜像（EV-004、EV-078）。

## 3. 代码包与模块边界

```
packages/
├── common/          # memmachine_common：共享 pydantic 请求/响应模型（api/spec.py、doc.py）
├── client/          # memmachine_client：Python SDK（Memory 封装）
├── server/          # memmachine_server：实现主体
├── ts-client/       # TypeScript SDK
├── meta/            # 汇总安装包
└── skills/          # memmachine-memory skill（agents/references）
```

`server` 内部按“领域 + 基础设施”分层（EV-004、EV-005）：

| 层 | 模块 | 职责 |
|---|---|---|
| 编排 | `main/memmachine.py` | 会话解析、三路写入、查询分派、feature API、删除队列 |
| 协议 | `server/app.py`、`server/api_v2/*` | FastAPI 组装、REST/MCP/config API |
| 记忆领域 | `episodic_memory/*` | STM/LTM、declarative/event 后端、实例缓存 |
| 记忆领域 | `semantic_memory/*` | set/category/tag、LLM ingestion、consolidation、storage |
| 基础设施 | `common/*` | 配置、资源生命周期、episode/session store、vector/graph store、embedder/LM/reranker |
| 检索增强 | `retrieval_agent/*` | agentic query decomposition / 多跳检索 |

## 4. 入口与启动顺序

1. `main()` 解析 `--stdio` / `--version` / `--with-config-api`（EV-007）。
2. 定位配置：`MEMORY_CONFIG` → `~/.config/memmachine/cfg.yml` → `./cfg.yml`；找不到即 `FileNotFoundError`（EV-008）。
3. `Configuration.load_yml_file()` 读取 YAML、递归小写 key，然后交给 pydantic 校验（EV-070）。
4. `ResourceManagerImpl(conf)` 建立数据库、embedder、LM、reranker、vector store 等资源。
5. 构建 `MemMachine`：加载 session、启动删除 worker、按需启动 3~5 个后台任务（EV-016、EV-033）。
6. `MemMachineAPI` 挂载 REST v2、MCP、可选 config router；`start_http()` 启动 uvicorn（EV-007、EV-010）。

> **关键事实**：服务启动成功 ≠ 记忆启用。`episodic_memory.enabled` 默认 `False`；`semantic_memory` 默认 `False`；缺依赖会被自动关闭（EV-019、EV-023、EV-024）。

## 5. 对外接口面

### REST v2（`api_v2/router.py`）
- Projects：`/projects`、`/projects/get`、`/projects/list`、`/projects/delete`、`/projects/episode-count`
- Memories：`/memories`（add）、`/memories/search`、`/memories/list`、`/memories/delete/episodic`、`/memories/delete/semantic`
- Features：`/features/add|get|update|delete`
- Semantic sets：`/semantic/set-type/*`、`/semantic/set-id/*`、`/semantic/category/*`、`/semantic/tag/*`
- 所有业务端点都是 `POST`（health 除外），请求体自带 `org_id/project_id`（EV-011、EV-012）。

### MCP（`api_v2/mcp.py`）
只暴露 3 个 tool：`add_memory`、`search_memory`、`delete_memory`（EV-014）。
身份来自 HTTP header：`org-id`、`proj-id`、`user-id`；缺失时为空字符串（EV-013）。

### Config API（`api_v2/config_router.py`）
仅在 `MEMMACHINE_CONFIG_API=1` 时挂载；支持读写 episodic/semantic 配置、增删 embedder/LM、重试 reranker（EV-012、EV-075）。

## 6. 外部依赖与集成

- **存储**：Postgres（episode/session/segment/semantic feature）、Qdrant（event vector）、Neo4j/Nebula（declarative graph）、SQLite/sqlite-vec/usearch/hnswlib（本地/测试）（EV-074、EV-085）。
- **模型**：OpenAI（responses/chat）、AWS Bedrock、Cohere、SentenceTransformers、LiteLLM（EV-072）。
- **框架**：LangChain、LangGraph、LlamaIndex、CrewAI、AWS Strands、Dify、FastGPT、n8n、OpenClaw（EV-004）。
- **可观测性**：Prometheus client、访问日志、请求指标中间件、`SIGUSR1` traceback dump（EV-088、EV-089）。

## 7. 架构层面的已知张力（只陈述，不升维）

1. **“图数据库”叙事 vs 默认 event 部署**：README 写 Episodic→Neo4j、Profile→SQL（EV-081），但样例 `configuration.event.yml` 的 event 后端是 Qdrant + Postgres，Neo4j 只在 `declarative` profile 启用（EV-073）。两者不是错误，而是**同一抽象的两个实现**；但文档口径会误导部署预期。
2. **REST 无认证 vs 多租户**：`router.py` 只依赖 `get_memmachine`，没有 `Security`/token 校验；租户由 body 的 `org_id/project_id` 决定（EV-012）。这是**把认证外包给网关**的架构选择，不能单独解读为“公网可攻击”。
3. **单进程编排 vs 多 worker**：删除队列、实例缓存、后台 consolidator 都是进程内状态；多 worker 时每个进程各有一份队列/缓存，依赖数据库 lease 与状态轮询收敛（EV-009、EV-034、EV-051）。

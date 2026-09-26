# 01 Project Layer — Mem0

> 本层只收录可从仓库实际内容追溯的事实（L0/L1）。每条标注证据位置。

## 1.1 项目形态与定位
| # | Fact | 证据 |
|---|------|------|
| P-01 | Mem0 是 AI agent 的"记忆层"产品：托管平台 + 开源 SDK 双形态 | LLM.md（"The Memory Layer for Personalized AI"）；README 双入口（OSS + hosted） |
| P-02 | OSS 包名 `mem0ai`，版本 2.1.0，Python >=3.10,<4.0 | pyproject.toml:7 |
| P-03 | 仓库是 polyglot monorepo：Python 核心 + TypeScript（mem0-ts）+ CLI（Python/Node）+ FastAPI server | 顶层目录：mem0/ mem0-ts/ cli/ server/ |
| P-04 | 16 个 agent/editor 集成（claude-code/codex/cursor/kimi/opencode/pi-agent/deepseek 等） | integrations/ 目录 |
| P-05 | 提供 Claude Code skill 定义（`skills/mem0/SKILL.md`） | skills/mem0/ |
| P-06 | 许可 Apache-2.0 | LICENSE / pyproject |

## 1.2 核心类与入口
| # | Fact | 证据 |
|---|------|------|
| P-07 | 同步 `Memory`（main.py:487）+ 异步 `AsyncMemory`（main.py:2172）继承 `MemoryBase` | mem0/memory/main.py |
| P-08 | `Memory.__init__` 装配四类组件：EmbedderFactory / VectorStoreFactory / LlmFactory + SQLiteManager + 可选 RerankerFactory | main.py:487-495 |
| P-09 | 托管客户端 `MemoryClient`（client/main.py:171）+ `AsyncMemoryClient`（1058） | mem0/client/main.py |
| P-10 | MemoryClient 默认 host `https://api.mem0.ai`，API key 来自 `MEM0_API_KEY` env 或参数 | client/main.py:204-210 |
| P-11 | OSS `Memory.chat()` 是 `NotImplementedError` 桩 | main.py:2168 |
| P-12 | CLI 入口 `mem0`（Python Typer）+ Node CLI（@mem0/cli） | cli/python/ cli/node/ |

## 1.3 核心模块
| # | Fact | 证据 |
|---|------|------|
| P-13 | 记忆编排核心 `Memory.add/search/update/delete/delete_all/get/get_all/history/reset` | main.py 方法清单（760/1379/1815/1869/1890/1208/1255/1946/2130） |
| P-14 | 本地持久化 `SQLiteManager`（storage.py:11）：history 表 + messages 表 | mem0/memory/storage.py |
| P-15 | 实体存储：spaCy 提取实体 → entity_store（延迟初始化 `_entity_store`）→ linked_memory_ids | main.py:559-560 / utils/entity_extraction.py |
| P-16 | 四类 ABC 基类：LLMBase / EmbeddingBase / VectorStoreBase / RerankerBase | llms/base.py:7 / embeddings/base.py:7 / vector_stores/base.py:4 |
| P-17 | 适配器规模：21 LLM / 15 embedding / 28 向量库 / 7 reranker | 各目录文件清单 |
| P-18 | 混合检索三信号：semantic + BM25(keyword_search) + entity boost | utils/scoring.py:60 `score_and_rank` |
| P-19 | 自托管 server：FastAPI + JWT + API key + slowapi rate limit + alembic + Docker（pgvector+Neo4j） | server/（main.py/auth.py/rate_limit.py/docker-compose.yaml） |
| P-20 | 遥测模块（telemetry.py 241 行）+ notices 模块（notices.py，远程配置提示） | mem0/memory/telemetry.py / notices.py |

## 1.4 核心数据结构
| # | Fact | 证据 |
|---|------|------|
| P-21 | `MemoryItem`（pydantic）：id/memory/hash/metadata/score/created_at/updated_at | configs/base.py:30-43 |
| P-22 | `MemoryConfig`：vector_store/llm/embedder 三默认工厂 + history_db_path（`~/.mem0/history.db`，`MEM0_DIR` 覆盖）+ reranker + version(v1.1) + custom_instructions | configs/base.py:45-65 |
| P-23 | 向量 payload 关键字段：`data`（原文）/`hash`（md5）/`text_lemmatized`/`created_at`/`updated_at`/`attributed_to` + scope 键（user_id/agent_id/run_id） | main.py:931-945（Phase 4-5 payload 构造） |
| P-24 | SQLite history 记录：memory_id/old_memory/new_memory/event/created_at/updated_at/actor_id/role/is_deleted | storage.py:150-193 |
| P-25 | 事件枚举：ADD/UPDATE/DELETE（+ GET 仅 telemetry） | storage.py / main.py history 调用 |
| P-26 | filters 三层 scope：user_id/agent_id/run_id 必须至少其一（search） | main.py:1438-1441 |

## 1.5 生命周期与状态
| # | Fact | 证据 |
|---|------|------|
| P-27 | 记忆生命周期：add→ADD 事件；update→UPDATE 事件（保留 created_at 改 updated_at）；delete→DELETE 事件（is_deleted=1） | main.py:2038-2130 |
| P-28 | 过期机制：expiration_date 在 payload，`_payload_is_expired` 过滤，show_expired 显式放行 | main.py:442 / 1648-1650 |
| P-29 | SQLite 生命周期：init→migrate→create tables→close/`__del__` | storage.py:12-346 |
| P-30 | 实体库延迟初始化：首次 `_compute_entity_boosts` 时创建 | main.py:559-560 |

## 1.6 测试体系
| # | Fact | 证据 |
|---|------|------|
| P-31 | pytest 129 个文件，目录分治（tests/memory|llms|vector_stores|embeddings|rerankers|configs） | tests/ 目录 |
| P-32 | 重点测试：test_chatty_llm_parsing（本地 LLM JSON 解析）、test_telemetry_sampling/aliasing、test_server_auth、test_oss_to_platform_migrate、test_client_feedback、test_http_client_proxies | tests/ 顶层清单 |
| P-33 | notice 专项测试 6 个（decay/temporal/scale/performance/first_run） | tests/memory/test_*notice*.py |
| P-34 | 遥测采样测试验证 0.1 默认采样 + lifecycle 100% | tests/test_telemetry_sampling.py |

## 1.7 配置与治理
| # | Fact | 证据 |
|---|------|------|
| P-35 | 53 个配置类：configs/base + 各 provider 配置（llms/embeddings/vector_stores/rerankers 分目录） | mem0/configs/ |
| P-36 | 工厂装配 provider→class 映射表（LlmFactory 18 provider / VectorStoreFactory / EmbedderFactory） | utils/factory.py |
| P-37 | AGENTS.md 治理：CLA 必签、PR 必须链 accepted issue（bot 1 分钟关）、workflow 修改需维护者批准、publishing 凭据按 workflow 文件名 pin、包级 lint 分治（root ruff 120 / cli ruff 100 / node Biome / ts Prettier / sdk ESLint） | AGENTS.md |
| P-38 | server 认证：JWT（HS256，access 30min + refresh 30day）+ bcrypt（密码/API key hash）+ ADMIN_API_KEY + AUTH_DISABLED 逃生阀 | server/auth.py |

## 1.8 外部依赖
| # | Fact | 证据 |
|---|------|------|
| P-39 | 核心依赖：qdrant-client/pydantic/openai/httpx/posthog/pytz/sqlalchemy/protobuf | pyproject.toml dependencies |
| P-40 | 可选组：各向量库/LLM/embedding 供应商 SDK | pyproject optional groups |
| P-41 | evaluation/ 是 git submodule → mem0ai/memory-benchmarks | .gitmodules |

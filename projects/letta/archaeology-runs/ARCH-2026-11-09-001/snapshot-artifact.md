# Repository Snapshot Artifact · ARCH-2026-11-09-001

## 1. 快照元信息

| 项 | 值 |
|---|---|
| project | Letta（前身 MemGPT）· **legacy V1 API server** |
| repository | `https://github.com/letta-ai/letta.git` |
| branch | **`archive`**（非 `main`） |
| commit | `56ba9c25552605eec89de8ed3dc6394b625c1993` |
| commit time | 2026-08-13 20:08:39 -0700 |
| commit message | `chore: deter self-promotional issue spam (#3429)` |
| version | `letta 0.16.8`（`pyproject.toml`） |
| clone | `git clone --depth 1 --single-branch --branch archive --no-tags` |
| clone size | 34 MB（工作树 + `.git`） |
| snapshot time | 2026-11-09 |
| license | Apache-2.0 |

**为什么不是 main**：`main` 分支已退化为文档仓（12 个文件），README 与 AGENTS.md 均声明本仓库为 **deprecated legacy server**，活动源码在 `letta-ai/letta-code`，旧 V1 实现保留在 `archive` 分支。本次考古对象是 **历史实现**，结论不代表当前 Letta 运行时。

**浅克隆限制**：`--depth 1` 无提交历史，无法回溯该实现的失败修复史、PR 讨论与版本演进节奏；下文所有"演进"判断只能基于代码内并存的多代实现（legacy/v2/v3），不能基于 git 历史。

## 2. 规模（脚本枚举，排除 `.git`）

| 指标 | 值 | 口径 |
|---|---:|---|
| 文件 | 1156 | `os.walk` 排除 `.git` |
| Python 文件 | 878 | 后缀 `.py` |
| Python 行数 | 248,417 | 逐文件 `sum(1 for _ in open(...))` |
| `tests/**/*.py` | 166 | 路径前缀 `tests/` |
| `test_*.py` | 95 | 文件名前缀 |
| GitHub workflows | 31 | `.github/workflows/*.yml|yaml` |
| Alembic migrations | 167 | `alembic/versions/*.py` |

**模块行数（Python，按顶层目录）**

| 目录 | 文件 | 行数 |
|---|---:|---:|
| `letta/services` | 119 | 41,334 |
| `letta/managers` | 22 | 22,928 |
| `letta/server` | 66 | 19,648 |
| `letta/schemas` | 83 | 17,939 |
| `letta/orm` | 167* | 10,649（versions 目录计入 `letta/orm` 之外） |
| `letta/llm_api` | 27 | 9,494 |
| `letta/agents` | 14 | 8,050 |
| `letta/helpers` | 20 | 5,627 |
| `letta/local_llm` | 34 | 5,471 |

> `alembic/versions` 167 文件 / 10,649 行是独立目录，上表 `letta/orm` 行数不含它。

## 3. 项目基础地图

### 3.1 语言与运行时
- Python `>=3.11,<3.14`；FastAPI + SQLAlchemy 2.x (async) + SQLModel + Alembic + Pydantic v2。
- 依赖含 `typer`/`questionary`（CLI）、`sqlalchemy[asyncio]`、`alembic`、`pydantic-settings`、`sentry-sdk`、多种向量后端（pgvector / Turbopuffer）。
- 入口：`letta` CLI（`pyproject.toml [project.scripts]` 区）、`letta/server/rest_api/app.py`（FastAPI app）。

### 3.2 顶层目录
```
letta/
├── agents/        执行循环（base_agent.py / base_agent_v2.py / letta_agent.py / v2 / v3 并存）
├── services/      业务层：agent_manager(3599) / block_manager(1048) / block_manager_git(596)
│                  passage_manager(1103) / summarizer(compact+sliding+self) / archive_manager
├── managers/      22 文件 / 22,928 行（service 之上的编排）
├── schemas/       Pydantic 模型：memory(884) / block(209) / passage / message / agent
├── orm/           Block / BlockHistory / BlocksAgents / Passage / Source / Identity / Organization
├── server/        FastAPI：rest_api/routers/v1（30 个 router）+ auth + middleware
├── functions/     function_sets/base.py（记忆工具 schema 与 legacy 实现）
├── llm_api/       多 provider 客户端
├── local_llm/     本地模型接入
├── prompts/       系统提示词
└── ...
alembic/versions/  167 个迁移
tests/             166 py（integration_test_* / test_* / sdk / 性能）
sandbox/           工具执行沙箱
```

### 3.3 核心数据结构（证据行号）

| 结构 | 位置 | 关键字段 |
|---|---|---|
| `Memory`（in-context） | `letta/schemas/memory.py:68` | `blocks: List[Block]`、`file_blocks`、`git_enabled`；`compile()` :688 |
| `Block` | `letta/schemas/block.py:67`；`letta/orm/block.py:22` | `label/value/limit`、`read_only`、`hidden`、`project_id`、`current_history_entry_id`、`version`（乐观锁） |
| `BlockHistory` | `letta/orm/block_history.py:10` | `sequence_number`、快照字段、`actor_type/actor_id` |
| `BlocksAgents` | `letta/orm/blocks_agents.py:7` | 多对多 `agent_id + block_id + block_label`，唯一约束 |
| `Passage`（archival） | `letta/orm/passage.py` | `text/embedding/embedding_config/archive_id/tags` |
| `AgentState` | `letta/schemas/agent.py` | `memory`（core）、`messages`（recall）、`tools` 等 |

### 3.4 三级记忆的实现对应

| 层级 | 存储 | 证据 |
|---|---|---|
| **Core memory** | `Memory.blocks` → `block` 表；系统提示词编译时注入 | `schemas/memory.py:68`、`:688 compile()` |
| **Recall memory** | 会话 `messages` 表 + 摘要压缩（compact / sliding window / self-summarizer） | `services/summarizer/compact.py`、`summarizer_sliding_window.py` |
| **Archival memory** | `passage` 表 + 向量（pgvector 或 Turbopuffer 双写） | `services/passage_manager.py:543 insert_passage` |

### 3.5 状态与生命周期
- **写路径**：agent 调 memory 工具 → `tool_executor` 改 `agent_state.memory` → `agent_manager.update_memory_if_changed_async` → Postgres `block` 行更新 → `checkpoint_block_async` 写 `block_history`。
- **版本化两套**：Postgres `BlockHistory` 线性 undo/redo（`:842/:952/:1004`）；`GitEnabledBlockManager` 每 agent 一 git repo，`{label}.md` 为文件，`get_block_at_commit` / `get_block_history` / `sync_blocks_from_git`。
- **摘要压缩**：按 token 阈值与 eviction percentage 切消息（`summarizer_sliding_window.py:99-218`），保留尾部消息。

### 3.6 测试体系
- `tests/pytest.ini`；`integration_test_*.py`（需要 DB/外部服务）+ `test_*.py`（单元）。
- 覆盖记忆的测试：`test_memory.py`、`test_block_manager_noop_update.py`、`test_compaction_thresholds.py`、`test_static_buffer_summarize.py`、`integration_test_summarizer.py`、`test_git_http_post_push_sync.py`、`integration_test_git_http_post_push_sync.py`。
- CI 31 个 workflow，含 `core-unit-test.yml`、`core-unit-sqlite-test.yaml`、`core-integration-tests.yml`、`migration-test.yml`、`docker-integration-tests.yaml`。

### 3.7 配置与外部依赖
- `pyproject.toml`：Postgres + pgvector（默认）、可选 Turbopuffer（`archive.vector_db_provider`）。
- LLM providers：`llm_api/` 多 provider + `local_llm/`（ollama/vllm/lmstudio 有独立 CI workflow）。
- 鉴权：`server/rest_api/auth_token.py:11 get_current_user`（password 或 api_key → user）。组织/项目作用域由 `OrganizationMixin` / `ProjectMixin` 提供。

### 3.8 权限与治理机制
- 租户：`organization_id`（所有 ORM 继承 OrganizationMixin）。
- 项目：`project_id`（Block / Passage 等）。
- block 级：`read_only`（注释 "permissions of the agent"）+ `hidden`。
- 共享：`blocks_agents`（agent 多对多）、`blocks_conversations`、`groups_blocks`、`identities_blocks`。
- 审计：`BlockHistory.actor_type/actor_id`；`created_by_id` / `last_updated_by_id`。
- 安全政策：`SECURITY.md` 仅漏洞上报流程；`AGENTS.md` 声明 deprecated + AI 使用政策。

### 3.9 扩展机制
- 工具：`letta/functions/` + `tool_executor`；MCP server 接入（`server/rest_api/routers/v1/mcp_servers.py`、`test_mcp_encryption.py`、`test_mcp_url_validation.py`）。
- 多 agent：`agents/base_agent_v2.py`、`letta_agent_v3.py`；`multi_agent` 工具集。
- 模板：Block template / `TemplateMixin`（human/persona 模板，`preserve_on_migration`）。

## 4. 快照阶段的事实边界

- 未运行任何代码：环境无 Postgres/pgvector/Turbopuffer/LLM key，集成测试不可执行。
- 未读取：`letta/managers/` 全部 22.9k 行、`server/rest_api/routers` 全部 30 个、`llm_api` provider 实现、167 个 Alembic 迁移全文。
- 代码行覆盖：精读约 1.2 万行 / 248,417（≈4.8%），集中在 memory / block / passage / summarizer / executor / ORM / auth。

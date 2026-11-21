# Repository Snapshot · Hy-Memory 2.0

## 1. 元信息

| 项 | 值 |
|---|---|
| repository | https://github.com/Tencent-Hunyuan/Hy-Memory.git |
| branch | `main` |
| commit | `9737431cf8965124b115f0e26cb0ff1e5cafa323` |
| commit time | 2026-11-06 15:26:45 +0800 |
| commit message | `chore(release): bump MemHub build number to 45 (#376)` |
| version | `hy-memory 2.0.14`（`pyproject.toml`） |
| license | Apache-2.0（`LICENSE` 前置 Tencent 声明，明确"无额外限制"；GitHub API 因自定义前缀报 NOASSERTION） |
| clone | `--depth 1 --single-branch --branch main`，只读 |
| repo size | 68 MB（含 docs / 测试 / MemHub 前端资源） |
| 创建时间 | 2026-10-15（GitHub API） |
| stars / forks / open issues | 2,020 / 146 / 153（2026-11-10 查询） |

## 2. 规模（脚本枚举，排除 `.git`）

| 指标 | 值 |
|---|---:|
| 文件总数 | 1,819 |
| `.py` 文件 | 860 |
| Python 行数 | 322,396 |
| `tests/**/*.py` | 138 文件 / 34,300 行 |
| `test_*.py` | 133 |
| CI workflows | 4（`ci.yml` / `hermes-provider.yml` / `linux-lifecycle.yml` / `release.yml`） |
| releases | `v1.0.6`（2026-11-06）、`v1.0.5`、`hy-memory-v2.0.5` |

### 模块分布

| 目录 | py 文件 | py 行数 | 说明 |
|---|---:|---:|---|
| `hy_memory/` | 146 | 102,896 | SDK / 引擎 |
| `memhub/` | 568 | 182,266 | 产品层（Core Python + TS/前端 + CLI + desktop） |
| `tests/` | 138 | 34,300 | 测试 |
| `plugins/` | 5 | 2,319 | Hermes / OpenClaw 集成 |
| `scripts/` | 3 | 615 | 工具脚本 |

### 单文件 Top（`hy_memory/`）

| 行数 | 文件 |
|---:|---|
| 3,858 | `hy_memory/pipelines/writer.py` |
| 3,693 | `hy_memory/pipelines/reader_base.py` |
| 2,476 | `hy_memory/pipelines/system2_agent.py` |
| 1,763 | `hy_memory/coding/store.py` |
| 1,575 | `hy_memory/agent/reconciler.py` |
| 1,422 | `hy_memory/agent/episode_summary.py` |
| 1,401 | `hy_memory/pipelines/specific_supplement.py` |
| 1,274 | `hy_memory/pipelines/system2_writer.py` |
| 1,273 | `hy_memory/models/memory.py` |
| 1,200 | `hy_memory/core/embed_service.py` |

> `hy_memory/client.py` 本身 7,826 行，是 SDK 的公开门面与编排中心。

## 3. 项目基础地图

### 3.1 语言与运行时
- Python `>=3.9`；CI 固定 3.12（`uv sync --extra dev --extra faiss --extra qdrant` + `pytest`）。
- MemHub 为 pnpm monorepo（Node 24），CI 分 `sdk` 与 `memhub` 两个 job。

### 3.2 入口
- SDK：`hy_memory/client.py::HyMemoryClient`（`add/search/get/update/delete/evolve/finalize_session`）；包入口 `hy_memory/__init__.py`。
- MemHub：`memhub/apps/core/src/one_memory/api.py`（FastAPI）+ `cli.py`；桌面 `apps/desktop`。
- 插件：`plugins/hermes`、`plugins/openclaw`。

### 3.3 核心模块
| 层 | 模块 | 职责 |
|---|---|---|
| 门面 | `client.py` | scene 路由、异步事件循环封装、幂等 request_id、CRUD |
| Chat 写入 | `pipelines/writer.py` | L1 raw 落库 → MemAgent 抽取 → reconcile → 版本化落库 |
| Chat 读取 | `pipelines/reader_base.py` | 多通道召回、层级配额、本地/图 rerank、演化链 |
| 抽取 | `agent/extractor.py` / `mem_agent.py` / `abstractor.py` | LLM 抽取 L2/L6/L7 |
| 冲突 | `agent/reconciler.py` | ADD / SUPERSEDE / UPDATE / DELETE(shadow) |
| 摘要 | `agent/episode_summary.py` | 会话 episode 摘要（opt-in） |
| 生产力记忆 | `coding/store.py`、`pipelines/writer.py`（productivity variant） | WorkingFact + Experience 双层 |
| 版本链 | `coding/chain_store.py` | 零 LLM 的 diff 链（SQLite） |
| 存储 | `data/*` | Chroma/Qdrant/FAISS、Kuzu/Neo4j、SQLite/MySQL cache |
| MemHub | `memhub/apps/core` | 管理面：授权、导入、队列、多用户、HTTP API |
| 授权 | `imports/grants.py`、`auth.py`、`user_scope.py`、`workspace_resolution.py` | 来源授权 / 本地 API / 用户隔离 / workspace scope |

### 3.4 核心数据结构
| 结构 | 位置 | 要点 |
|---|---|---|
| `MemoryLayer` | `models/memory.py:25` | L0~L7；L4/L5 为 reserved |
| `MemoryStatus` | `models/memory.py:121` | ACTIVE / SUPERSEDED / NEGATED / CONFLICTED / SHADOW / ARCHIVED |
| `MemoryNode` | `models/memory.py:223` | 统一节点；`owner`、`supersedes`/`superseded_by`/`is_latest`、`valid_from/until`、`evidence_chain` |
| `WorkingFact` | `coding/store.py:97`（表 `working_fact`） | `derived_experience_ids` |
| `coding_memory_meta` | `coding/store.py:73` | `source_fact_ids`、`confidence` |
| `thread/round/repo_symbol` | `coding/chain_store.py:148-170` | 版本链：owner/branch/file/symbol/pkey + 每轮 said/patch |

### 3.5 状态与生命周期
- Chat：raw append-only → L2 事实版本化（supersede 链）→ L3 摘要 → L6 schema（ultra + evolve）→ L7 intention（过期降级为普通事实）。
- 逻辑删除：`status=SHADOW` + `is_latest=False`，不物理删除。
- Working/Coding：fact 每会话恰好 1 条；experience 0~N（需过价值阈值）；superseded fact 通过 `fact_chain_summary` 保留。
- Coding 额外：版本演化链按 `owner + branch + file + symbol + pkey` 建 thread，每轮 round 存真实 patch。

### 3.6 测试体系
- `tests/` 133 个 `test_*.py`，34,300 行；CI 在 PR/push 时对 `hy_memory/**`、`tests/**`、`memhub/**` 触发。
- `pytest` 默认 `-m 'not integration'`（integration 需显式标记）。
- MemHub 有独立 `apps/core/tests` 与前端测试（CI 中 pnpm 链路）。

### 3.7 配置
- 环境变量前缀 `MEMORY_*`（LLM / embedder / vector store / graph / coding / 语言 / 去重阈值等）。
- `MEMORY_LLM_MODEL` 默认 `gpt-4.1-nano`；embedding 默认 `text-embedding-3-small` / 1536 dims。
- 多后端可选：Chroma（默认）、Qdrant、FAISS；Kuzu（默认）、Neo4j；MySQL cache。
- 数据目录默认 `~/.hy_memory`（`MEMORY_DATA_DIR` 可覆盖）。

### 3.8 权限与治理机制（MemHub）
- 本地 API：Origin 白名单 + loopback IP 解析（防 DNS rebinding）+ Bearer token / UI session，`hmac.compare_digest`。
- 来源授权：`source_grants(user_id, source_id, requested_path, root_identity, revoked_at)`；读取时校验 `root_identity`，路径身份变化即 `SOURCE_IDENTITY_CHANGED`。
- 多用户：默认 `local`；仅 `hermes` 可接入第二个用户根；`lane_key(user_id, source_id)` 以 `@` 分隔；user_id 排除 `:`（SDK 用 `:` 拼 history key）。
- workspace scope：`workspace:v1:<sha256>`，调用方只能解析自己所在目录。

### 3.9 外部依赖
- 必需：pydantic、numpy、openai、chromadb、kuzu、langdetect、scikit-learn、tiktoken、json-repair、rank-bm25、jieba。
- 可选：qdrant-client/fastembed、faiss-cpu、neo4j、aiomysql、torch/transformers（本地 rerank）。
- 外部服务：任意 OpenAI 兼容 LLM / embedding endpoint；本地 HuggingFace 模型（30B-A3B）不在仓库内。

## 4. 事实边界（本轮）

- 浅克隆 `depth=1`，无 git 历史；无法回溯某行代码的修复史或 PR 讨论。
- 未执行任何集成测试 / 真实运行；所有结论上限 **S3（Implemented）**。
- 未覆盖：`memhub/apps/desktop`、`memhub/apps/web` 的前端实现；`memhub/packages/*` 全量；`hy_memory/pipelines/system2_*` 的完整演化逻辑（仅入口级）。
- 未验证 40+ `chore/*` 分支的差异；本快照只描述 `main`。
- 厂商自报指标（LoCoMo 72.6% / MemoryArena 55.31 等）**不进入事实层**，只在 Overview 中作为文档声明标注。

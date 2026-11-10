# Repository Snapshot Artifact · OpenViking

## 1. 快照元数据

| 项 | 值 | 来源 |
|---|---|---|
| repository | `https://github.com/volcengine/OpenViking.git` | Job Manifest |
| branch | `main` | `git rev-parse` |
| commit SHA | `f7e83b2d739905b50215cd7739199a564b9b044b` | `git log -1` |
| commit time | 2026-11-10 22:17:06 +0800 | `git log -1 --date=iso` |
| commit subject | `chore(ui): prepare react 0.1.6 release (#7391)` | 同上 |
| clone | `git clone --depth 1 --single-branch --branch main` | 命令记录 |
| clone size | 218 MB（不含 `.git` 语义上的工作树体积；`du -sh repo`=218M） | `du -sh` |
| file count | 6,608（排除 `.git`） | `find ... \| wc -l` |
| license | AGPL-3.0（LICENSE 34,568 B） | `pyproject.toml:19` / API |
| declared version | `0.3.22`（README "Proof it works" 段） | `README.md` |
| python requirement | `>=3.10` | `pyproject.toml:15` |

> 版本注记：`pyproject.toml` 使用 `dynamic = ["version"]` + `setuptools-scm`，仓库内无静态版本号；0.3.22 来自 README 文本，未由代码常量证明。**Epistemic: Observation（文档声明）**。

## 2. 规模与语言分布

| 指标 | 值 | 口径 |
|---|---:|---|
| Python 文件 | 2,252 | `find -name '*.py'` |
| Python 行数 | **733,129** | `wc -l`（全仓库，含 tests/frontend 脚本） |
| Rust 文件 / 行数 | 232 / **130,408** | `find crates -name '*.rs'` |
| Markdown | 650 | 扩展名统计 |
| TypeScript / TSX | 685 / 594 | 同上 |
| 扩展名 Top | `.py 2252` / `.ts 685` / `.md 650` / `.tsx 594` / `.mjs 467` / `.woff2 306` / `.h 251` / `.rs 232` | 同上 |

### 目录文件分布（前 10）

| 目录 | 文件数 | 说明 |
|---|---:|---|
| `tests/` | 932 | 803 个 `test_*.py`，33 个子目录 |
| `docs/` | 900 | 含 18 篇 `docs/en/concepts/` 设计文档 |
| `openviking/` | 895 | Python 核心引擎 |
| `examples/` | 818 | 示例（**scope 外**） |
| `web-studio/` | 817 | Web UI（**scope 外**） |
| `frontend/` | 710 | 前端（**scope 外**） |
| `bot/` | 283 | VikingBot（**scope 外**） |
| `crates/` | 247 | Rust `ragfs` / `ov_cli` / `ragfs-python` |
| `benchmark/` | 178 | LoCoMo / tau2 复现脚本 |
| `third_party/` | 372 | 第三方（**scope 外**） |

### `openviking/` 模块（Python 文件数）

```
storage 151 | session 99 | server 89 | parse 68 | metrics 58 | models 40
service 35  | utils 31   | observability 23 | config 22 | ingest 20
retrieve 20 | privacy 17 | core 16 | scheduling 15 | resource 14
telemetry 13 | usage_reporter 7 | pyagfs 6 | connector 5 | crypto 5
ttl 5 | message 3 | system_assets 3 | prompts 2 | client 1 | web_studio 1
```

## 3. 入口

| 入口 | 定义 | 文件 |
|---|---|---|
| `ov` | `openviking_cli.rust_cli:main`（Rust CLI 极简包装） | `pyproject.toml:234` |
| `openviking-server` | `openviking_cli.server_bootstrap:main`（也可 `ingest`） | `pyproject.toml:236` |
| `openviking-gateway` | `openviking_gateway.cli:main` | `pyproject.toml:237` |
| Python SDK | `SyncHTTPClient` / `AsyncHTTPClient`（延迟导入） | `openviking/__init__.py:19-38` |

启动链：`openviking-server` → `openviking/server/app.py`（FastAPI）→ `openviking/service/core.py`（Service 容器）。

## 4. 核心模块地图（经代码验证的类位置）

| 层 | 类 / 文件 | 行号 | 职责 |
|---|---|---|---|
| HTTP | `openviking/server/app.py` | — | FastAPI 装配、auth plugin 初始化（L134-163） |
| Service | `FSService` `openviking/service/fs_service.py` | 129 | ls/mkdir/rm/mv/tree/stat/read/abstract/overview/grep/glob |
| Service | `SearchService` `openviking/service/search_service.py` | 64 | search / find |
| Service | `SessionService` `openviking/service/session_service.py` | 68 | session / commit |
| Service | `ResourceService` `openviking/service/resource_service.py` | 230 | add_resource / add_skill / wait_processed |
| Retrieve | `IntentAnalyzer` `openviking/retrieve/intent_analyzer.py` | 38 | LLM 生成 typed queries（0–5） |
| Retrieve | `HierarchicalRetriever` `openviking/retrieve/hierarchical_retriever.py` | 64 | 单次向量召回 + 可选 rerank |
| Session | `SessionCompressorV3` `openviking/session/compressor_v3.py` | 244 | 归档后记忆抽取编排 |
| Session/Memory | `ExtractLoop` `openviking/session/memory/extract_loop.py` | 117 | ReAct 式抽取循环 / 操作解析 / 修复 |
| Session/Memory | `MemoryUpdater` `openviking/session/memory/memory_updater.py` | 891 | merge/dedup/写回/删除/向量化 |
| Storage | `VikingFS` `openviking/storage/viking_fs/__init__.py` | 110 | URI → 文件系统 + 向量索引统一抽象 |
| Storage | `rm()` 实现 `openviking/storage/viking_fs/_ops.py` | 240 | 锁 → 删索引 → 删文件 |
| Storage | `SessionCommitProcessor` `openviking/storage/queuefs/session_commit_processor.py` | 25 | 持久队列消费 Phase 2 |
| Rust | `crates/ragfs` | — | AGFS/RAGFS + PathLockManager（Python 侧只留 binding） |

## 5. 核心数据结构（代码存在性）

| 结构 | 位置 | 字段/要点 |
|---|---|---|
| `MemoryTypeSchema` | `openviking/session/memory/dataclass.py:227` | schema 驱动的字段与合并语义 |
| `MemoryFile` | `openviking/session/memory/dataclass.py:294` | `uri/content/links/backlinks/memory_type/extra_fields` |
| `ResolvedOperation` | 同文件 `:346` | 单条已解析操作（uris、memory_type、memory_fields） |
| `ResolvedOperations` | 同文件 `:365` | 批量 + errors + delete_file_contents + resolved_links |
| `MemoryUpdateRequest` | `openviking/session/memory/streaming_memory_updater.py:105` | 批量提交载体 |
| `RequestContext` | `openviking/server/identity.py:295` | account/user/role/group/ACL 上下文 |
| `SessionCommitMsg` | `openviking/storage/queuefs/session_commit_msg.py:10` | 持久任务消息 |
| `TypedQuery` / `QueryPlan` | `openviking_cli/retrieve/types.py:239 / :261` | 检索查询计划 |

## 6. 测试体系

| 项 | 值 |
|---|---|
| `tests/test_*.py` | **803** |
| 测试子目录 | 33（`session` / `storage` / `retrieve` / `transaction` / `server` / `service` / `vectordb` / `acl` 相关分散） |
| CI workflow | **37**（`.github/workflows/`）：`ci.yml`、`pr.yml`、`_test_full.yml`、`_test_lite.yml`、`api_test.yml`、`gateway-e2e.yml`、`host-compat.yml`、`rust-cli.yml` 等 |
| 特殊 | `tests/transaction/` 直接对应 09-transaction 设计；`tests/oc2ov_test/` 疑似 OpenClaw→OpenViking 兼容测试 |

> 未执行测试（只读、不装依赖）。测试体系仅作“系统重视哪些行为”的证据，不作为 S4 验证。

## 7. 配置与依赖

- 配置目录 `openviking/config/`（22 文件）：`account_config.py`（`acl` 字段在 `:167`）、`account_vector.py`（embedding/circuit breaker/vector DB）、`vlm.py`、`ttl.py`、`parser_api.py`、`feishu.py`、`manager.py`、`merge.py`、`validate.py`。
- 关键运行依赖（`pyproject.toml`）：`aiohttp`、`httpx`、`pydantic>=2`、`openai>=1`、`volcengine` + `volcengine-python-sdk[ark]`、`pdfplumber`、`scrapy`、`trafilatura`、`feedparser`、`apscheduler`、`openviking-sdk>=0.1.17`。
- 外部服务：VLM / embedding provider（Doubao 系列可配）、VectorDB（VikingDB / Volcengine 向量库）、AGFS 后端（localfs/s3fs/memory）。
- 模块开关：`acl.enabled` 默认 **false**；`encryption.enabled` 默认 **false**（`openviking/crypto/config.py:35`）。

## 8. 权限与治理机制（快照层）

| 机制 | 位置 | 要点 |
|---|---|---|
| Auth 插件 | `openviking/server/auth/plugins/{api_key,trusted,dev,ldap,oidc}.py` | `auth_mode` 五选一 |
| trusted 模式 | `trusted.py:125-200` | **必须** `X-API-Key == server.root_api_key`（`hmac.compare_digest`），再信任 `X-OpenViking-Account/User/Role/Groups` |
| ACL | `openviking/storage/acl.py` | 字段 `acl_direct_grants` / `acl_inherited_grants` / `acl_mode`；默认关闭 |
| 多租户 | `RequestContext` + URI 前缀 `/local/{account_id}/...` | account 外层、user 内层 |
| 隐私 | `openviking/privacy/`（17 文件） | 写时占位符、读时恢复、版本化 |

## 9. 事实来源登记（Snapshot 层）

| ID | 事实 | 证据 |
|---|---|---|
| SNAP-01 | HEAD / commit time | `git log -1` |
| SNAP-02 | 6,608 文件 / 218 MB | `find` + `du` |
| SNAP-03 | 733,129 Python 行 / 130,408 Rust 行 | `wc -l` |
| SNAP-04 | 803 个 test 文件 / 37 CI workflow | `find` + `ls` |
| SNAP-05 | 三个 console_scripts | `pyproject.toml:234-237` |
| SNAP-06 | ACL 字段与默认关闭 | `acl.py:51` + `15-acl.md:63` |
| SNAP-07 | trusted 模式要求 root key | `trusted.py:139-151` |
| SNAP-08 | rm 先删索引再删文件 | `_ops.py:376-390` |

## 10. Scope 声明（考古前）

- **覆盖**：`openviking/` 的 `session|storage|retrieve|service|server|privacy|crypto|config`；`docs/en/concepts/` 18 篇；`crates/ragfs` 仅目录级与接口边界。
- **不覆盖**：`frontend` / `web-studio` / `examples` / `bot` / `integrations` / `sdk` / `third_party` 的业务实现；Rust `ragfs` 内部算法；模型调用真实行为。
- **后果**：Coverage 上限为“Python 核心子系统抽样”，Flow Atlas 中涉及 Rust 锁实现的 Edge 标 `boundary=Rust-unread`。

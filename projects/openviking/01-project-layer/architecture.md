# 01 · Project Layer — Architecture

## 1. 项目定位

OpenViking 是 **"Agent-native context database"**（`pyproject.toml:14`）。它把三类上下文——**Resource（知识/文档/代码）**、**Memory（用户偏好/经验/事件）**、**Skill（可执行程序）**——统一到 `viking://` 命名空间，并以文件系统原语作为主接口。

代码/文档对"上下文数据库"的操作化定义：

| 能力 | 实现载体 |
|---|---|
| 可导航 | `ls/tree/stat/grep/glob`（FSService，`fs_service.py:129`） |
| 可读且可编辑 | `read_file/write_file` + 目录 sidecar `.abstract.md` / `.overview.md` |
| 可语义检索 | `find/search`（SearchService + HierarchicalRetriever） |
| 可归档 | session commit → archive + memory extraction |
| 可隔离 | account / user / peer + ACL |
| 可迁移 | `ovpack` export/import/backup/restore（PackService） |

## 2. 分层架构（代码验证）

```
CLI(ov, Rust wrapper) / SDK(py,ts,go?) / HTTP
                │
        FastAPI Server            openviking/server/app.py
                │
        Service Layer             openviking/service/{fs,search,session,resource,pack,debug}_service.py
                │
   ┌────────────┼──────────────┬─────────────────┐
   │            │              │                 │
Retrieve     Session        Parse/Ingest     Resource
(intent→     (archive→       (parser→tree→    (R/N/F/V plan
 vector→     extract→         semantic)        →content commit)
 rerank)      update)
   │            │              │                 │
   └────────────┴──────┬───────┴─────────────────┘
                       │
             VikingFS（URI 抽象）  openviking/storage/viking_fs/
                 │                  │
          RAGFS（Rust）        Vector Index（VikingDB）
        crates/ragfs           openviking/storage/
```

### 关键不对称（设计核心）

| 层 | 同步/异步 | 权威性 | 可重建 |
|---|---|---|---|
| 内容（RAGFS/AGFS） | 同步（`add_resource` 的 content commit 在锁内） | **事实源** | 否（靠备份） |
| 目录摘要（`.abstract.md/.overview.md`） | 异步（SemanticQueue，自底向上） | 派生 | 是 |
| 向量索引 | 异步（embedding queue / 写后向量化） | 派生 | 是 |
| session archive | Phase 1 同步发布；Phase 2 异步 | 半权威（发布后不可重算一致） | 否（模型重跑不保证同文） |

> 事实来源：`09-transaction.md`（设计）；`_ops.py:376-390`（rm 顺序）；`compressor_v3.py:425-537`（抽取异常策略）；`tree_builder.py:54`（sidecar 异步）。

## 3. 模块职责（Python 核心）

| 模块 | 文件数 | 职责 | 关键类 |
|---|---:|---|---|
| `storage/` | 151 | URI→路径、双存储、ACL、队列、快照 | `VikingFS`(`viking_fs/__init__.py:110`)、`acl.py`、`queuefs/` |
| `session/` | 99 | 会话、归档、记忆抽取/更新 | `SessionCompressorV3`(`compressor_v3.py:244`)、`ExtractLoop`、`MemoryUpdater` |
| `server/` | 89 | FastAPI 路由、auth、identity | `app.py`、`auth/plugins/*` |
| `parse/` | 68 | PDF/MD/HTML/代码解析、tree build、语义分组 | `tree_builder.py`、`parsers/` |
| `metrics/` | 58 | 指标采集与导出 | — |
| `models/` | 40 | 领域模型 | — |
| `service/` | 35 | 业务门面 | `FSService`/`SearchService`/`SessionService`/`ResourceService`/`PackService`/`DebugService` |
| `retrieve/` | 20 | 意图→召回→rerank→组装 | `IntentAnalyzer`/`HierarchicalRetriever`/`context_assembler/` |
| `privacy/` | 17 | 敏感值占位/恢复/版本 | `PrivacySanitizer`/`PrivacyRestorer` |
| `crypto/` | 5 | 静态文件加密 | `FileEncryptor`（默认关闭） |
| `config/` | 22 | 账户级配置（模型/向量/ACL/TTL/飞书） | `AccountConfig`(`account_config.py:142`) |

## 4. 入口与部署

| 入口 | 命令 | 说明 |
|---|---|---|
| CLI | `ov` | `openviking_cli.rust_cli:main`（Rust 二进制包装；文件系统底层能力来自 ragfs） |
| Server | `openviking-server` | `openviking_cli.server_bootstrap:main`；FastAPI，默认 `1933`（文档示例） |
| Gateway | `openviking-gateway` | `openviking_gateway.cli:main`；多实例/网关层 |
| SDK | `SyncHTTPClient` / `AsyncHTTPClient` | `openviking/__init__.py:19-38` 延迟导入 |

部署形态（文档）：单机 `uv` + Python 3.10+ + 模型 provider；Docker/Compose；Railway；K8s chart（`gateway-chart.yml`）。
认证模式：`api_key` / `trusted` / `dev` / `ldap` / `oidc`（`server/auth/plugins/`）。

## 5. 外部依赖（运行面）

- **模型**：VLM（如 Doubao 2.0 Pro）+ embedding（如 Doubao-embedding-vision-251215）——模型可配，非硬编码。
- **向量库**：VikingDB / Volcengine 向量服务（`config/account_vector.py:117-131`）。
- **文件后端**：`localfs` / `s3fs` / `memory`（`05-storage.md`）。
- **解析器**：`pdfplumber`、`scrapy`、`trafilatura`、`feedparser`、`cairosvg`（`pyproject.toml`）。
- **调度**：`apscheduler`（TTL / 周期任务）。

## 6. Scope 与未覆盖边界

| 覆盖 | 未覆盖 | 影响 |
|---|---|---|
| `openviking/{session,storage,retrieve,service,server,privacy,crypto,config}` | `frontend/`（710）、`web-studio/`（817）、`examples/`（818）、`bot/`（283） | UI/示例不进入知识结论 |
| `docs/en/concepts/01..17` | 其他语言文档、API 参考全量 | 设计意图只取英文概念文档 |
| `crates/ragfs` 目录与 Python binding 调用点 | Rust 锁/文件系统内部算法（130k 行） | 凡依赖 Rust 内部行为的 EK 标 `Rust-unread` |
| — | 运行时行为（未启动服务、未跑测试） | 无 S4/S5 证据；全部结论 ≤ S3 |

## 7. 项目规模事实

| 指标 | 值 |
|---|---:|
| 文件（排除 .git） | 6,608 |
| Python | 2,252 文件 / 733,129 行 |
| Rust | 232 文件 / 130,408 行 |
| 测试文件 | 803（33 个目录） |
| CI workflow | 37 |
| 浅克隆体积 | 218 MB |

> 规模本身是结论的一部分：**733k 行 Python 的仓库无法"全量覆盖"**，本 Package 的 Coverage 只能声明为"核心链路的机制级抽样"，不能声称"已看清整个项目"。

# Snapshot Artifact — ARCH-2026-11-04-001

## Repository Snapshot（事实，可追溯到仓库实际内容）

| 项 | 值 |
|---|---|
| project | durare |
| repository | https://github.com/SamuelXing/durare.git |
| 克隆方式 | 浅克隆（--depth 1）到 `archaeology-jobs/ARCH-2026-11-04-001/repo/` |
| branch | main（origin/HEAD → origin/main） |
| **commit SHA** | `87ed2db147474b2cdb7f0a1ae9d80ccf49d74483` |
| commit message | `fix(provider): commit a workflow's sends and stream writes with their checkpoint (#252)` |
| commit timestamp | 2026-11-01 19:58:26 -0800 |
| version | **0.5.0**（Cargo.toml `version = "0.5.0"`） |
| 最近 release | CHANGELOG `[0.5.0] - 2026-10-06`；`[0.4.0] - 2026-08-01` |
| license | MIT OR Apache-2.0（`LICENSE-MIT` + `LICENSE-APACHE`） |
| rust-version | 1.88（Cargo.toml） |
| edition | 2021 |
| created | 2026-06-10（GitHub API） |
| pushed_at | 2026-11-03（GitHub API） |
| stars/forks | 15 / 2（GitHub API，2026-11-04 抓取） |
| open issues | 30（GitHub API） |
| downloads | 1031 total / 878 recent（crates.io API） |

## 项目基础地图

### 一句话定位
社区维护（非 DBOS 官方）的 **DBOS 兼容 durable-execution 库**：普通 async 函数每步 checkpoint 到 Postgres/SQLite/In-Memory，崩溃/重启/重新部署后从精确断点恢复，已完成步骤绝不重跑；无独立服务器、无 sidecar——引擎是进程内库，直连数据库，使用与 DBOS Transact SDK（Python/Go/TS/Rust）相同的 `dbos` 系统 schema，跨语言在同一数据库上互操作。

### 语言与规模（python3 逐文件统计）
| 分区 | 文件数 | 行数 |
|---|---|---|
| src/（库本体） | — | 32,964 |
| tests/（集成测试） | 61 文件 | 26,797 |
| durare-macros/（proc-macro） | 1 | 524 |
| durare-benchmarks/ | 1 | 440 |
| examples/ | — | 1,168 |
| **合计** | ~110 .rs | **~61,893** |

### 入口与模块结构（src/lib.rs 声明，事实）
- **public 模块**（`pub mod`）：`design` / `determinism` / `durability` / `messaging` / `observability` / `operations` / `queues` / `security` / `transactions`
- **内部模块**（`mod`）：`admin` / `client` / `conductor` / `context` / `datasource` / `debounce` / `engine` / `error` / `execution` / `handle` / `memory` / `postgres` / `provider` / `queue` / `recorded_error` / `replay` / `schedule` / `serialize` / `sqlite` / `tx`
- **公开导出**：`AdminServer` / `Client` / `Conductor{,Config,AlertHandler}` / `DurableContext` / `AuthContext` / `PendingStep` / `StepCtx` / `StepOptions` / `RetryPredicate` / `DataSource{,Pg,Sqlite}` / `Debouncer{,Client}` / `DurableEngine` / `WorkflowResult` / `Error,ErrorCode,RecordedError,Result` / `WorkflowHandle` / `InMemoryProvider` / `PostgresProvider` / `RateLimiter` / `WorkflowQueue` / `ReplayReport,Divergence` / `ScheduledInput` 等
- **workspace**：`durare` + `durare-macros`（proc-macro 伴生 crate，version 0.2）

### 核心模块职责（AGENTS.md repository map + 代码，事实）
| 模块 | 职责 |
|---|---|
| `src/engine.rs` | `DurableEngine`：dispatch / recovery / child starts / fork / verify_replay / launch / shutdown / deactivate / settle；`Runtime` 共享执行核 |
| `src/context.rs` | `DurableContext`：workflow body 的 durable 调用（step / transaction / select / sleep / now / uuid / random / send / recv / set_event / get_event / start_workflow / enqueue_workflow / patch） |
| `src/execution.rs` | per-execution latch（`RecoveryRequired` / `ObservationFailed`）——两层编排的唯一 seam |
| `src/provider.rs` | `StateProvider` trait + 生命周期状态常量 + 去重/目标冲突/不存在映射 + ChangeWait 通道 |
| `src/memory.rs` / `sqlite.rs` / `postgres.rs` | 三后端实现 |
| `src/replay.rs` | `verify_replay` 报告（recorded/matched/divergence） |
| `src/security.rs` | 信任地图：动态 SQL 不变量、secrets 处理、网络暴露面、Authorization |
| `src/determinism.rs` / `durability.rs` | 确定性规则书 / 执行模型（checkpoint-replay） |
| `src/transactions.rs` | 事务步骤：SQL 写与 checkpoint 同一 commit（消除 at-least-once window） |
| `src/schedule.rs` / `queues.rs` / `debounce.rs` | 持久 cron 调度 / 持久队列（fleet-wide）/ 防抖 |
| `src/serialize.rs` | portable_json（跨 SDK 字节一致的序列化，serde_json preserve_order） |
| `src/client.rs` / `admin.rs` / `conductor.rs` | 出站 Client / Admin HTTP 服务器（opt-in `admin`）/ 云控制面 websocket 客户端（opt-in `conductor`） |
| `durare-macros/` | `#[workflow]` / `#[step]` / `#[transaction]` 过程宏（编译期注册 + PendingStep 形状） |

### 核心数据结构（docs/schema.md，事实）
- **10 张表**（40 个 postgres migrations / 35 个 sqlite migrations，与 Go/Python SDK column-for-column）：
  - `workflow_status`（每 workflow 一行：workflow_uuid PK / status / name / inputs / output / error / executor_id / application_version / recovery_attempts / queue_name / queue_partition_key / priority / deduplication_id / rate_limited / delay_until / parent_workflow_id / forked_from / timeout/deadline / started/completed/created/updated / authenticated_user / assumed_role / authenticated_roles / request / attributes / owner_xid / serialization / class_name / config_name / application_id）
  - `operation_outputs`（步骤与 child 调用 checkpoint：workflow_uuid + function_id 复合 PK / function_name / output / error / child_workflow_id / serialization）
  - `notifications`（send/recv 邮箱）、`workflow_events` / `workflow_events_history`（set_event/get_event）、`streams`（write_stream）、`workflow_schedules`（调度）、`queues` 等
- **跨 SDK 字节约定**：时间戳全部存 `BIGINT` epoch 毫秒（非 SQL timestamp）；`STREAM_CLOSED_SENTINEL = "__DBOS_STREAM_CLOSED__"`；`authenticated_roles` 单列 JSON 数组
- **生命周期状态常量**（provider.rs 事实）：`ENQUEUED` / `DELAYED` / `PENDING` / `SUCCESS` / `ERROR` / `CANCELLED` / `MAX_RECOVERY_ATTEMPTS_EXCEEDED`

### 控制流/生命周期（engine.rs，事实）
- `DurableEngine::new(provider)` / `with_config` / `connect(url)` → `launch()`（spawn 每 queue dispatcher + 每 schedule firing loop + 可选恢复 + retention sweeper）→ `start`/`start_with`（直接开始）或 `Client::enqueue`（入队）→ `recover()`（重派 PENDING 到 task tracker，不 inline 运行）→ `shutdown(timeout)`（drain in-flight）→ `deactivate()`（停止认领新活但服务 in-flight 与 admin）
- 内部队列 `_dbos_internal_queue`（100ms poll，重执行/恢复直连 run 的路由）
- settlement backoff：100ms 翻倍、上限 5s；`SCHEDULE_RECONCILE_INTERVAL` 500ms
- 状态流：`ENQUEUED → PENDING → SUCCESS|ERROR`；`DELAYED → ENQUEUED`；恢复超限 → `MAX_RECOVERY_ATTEMPTS_EXCEEDED`（park，显式 resume）；operator → `CANCELLED`

### 测试体系（事实）
- 61 个集成测试文件，26,797 行；`tests/common/mod.rs` 共享 fixtures（`recorded`、`hermetic_pg_db`）
- 关键文件：`crash_consistency.rs` / `adversity.rs` / `interop.rs`+`interop_db.rs`（跨 SDK 互操作）/ `security.rs`（SQL 注入扫描）/ `postgres.rs` / `sqlite.rs` / `recovery_liveness.rs` / `concurrency.rs` / `child_atomic.rs`（`contract(db)` 三后端同跑）/ `step_cancellation.rs` / `cancel_children.rs`
- CI 四门（AGENTS.md + ci.yml）：`cargo fmt --all --check` / `cargo clippy --all-targets --all-features -- -D warnings` / `cargo test --all-targets --all-features` + `--doc` / `RUSTDOCFLAGS="-D warnings" cargo doc`；外加 `cargo semver-checks`（对 crates.io release，`--no-default-features` 单后端矩阵）
- Postgres 测试需 `DATABASE_URL`（docker postgres:16 于 54329）；`pg_client_debounces` 已知 flake（#244）
- 崩溃注入：`fail` crate `failpoints` feature（dev-dependency 统一开启，正常构建零成本）
- 安全 CI（security.yml）：`cargo audit` + `cargo deny`（deny.toml）每周
- benchmark CI（bench.yml）：`steps`（upstream-comparable）/ `concurrent`（200 workflows × 32 concurrency）/ `memory`（1000 parked）三项

### 配置与特性（Cargo.toml，事实）
- features：`default = ["postgres", "sqlite"]`；`conductor`（opt-in，tokio-tungstenite+flate2）；`admin`（opt-in，axum）；零后端构建=编译错误
- EngineConfig：`app_version`（默认 sha-256 of running executable）/ `executor_id` / `recover_on_launch` / `retention`（RetentionPolicy：period/max_rows/sweep_interval）/ `max_recovery_attempts`
- WorkflowOptions：with_id / queue / authenticated_user / assumed_role / attributes / partition_key / dedup_id / dedup_policy / config_name / class_name / app_version / authenticated_roles
- 外部依赖：tokio(full) / serde / serde_json(preserve_order) / async-trait / chrono+chrono-tz / thiserror / tracing / inventory / uuid(v4) / cron / base64 / sha2 / tokio-util(0.7.10+, CancellationToken+TaskTracker) / sqlx(0.8, postgres+sqlite 驱动) / fail；可选：flate2 / axum(0.7) / tokio-tungstenite(0.23, rustls) / futures-util

### 权限与治理机制（事实）
- **AGENTS.md**：完整工程治理——4 门 CI 前置、Conventional Commits（`type(scope): summary`，≤72 字符，无工具署名）、PR 描述模板（What/Why/How/Breaking/Tests/Follow-ups）、边界（禁 push main、禁 force-push 共享分支、禁 `cargo publish`、禁提交 secrets/本地 DB URL/scratch、禁在非自建 checkout 跑破坏性 git 操作）、发布走 RELEASING.md（tag first → `cargo package --list` → publish）
- **Authorization**：`require_roles(name, roles)` 声明式角色门槛，engine 在 **start 之前**检查；`AuthContext`（authenticated_user/assumed_role/authenticated_roles）从 start 贯穿 workflow/children/trace/persisted row
- **动态 SQL 不变量**（security.rs）：无 caller 字符串进入 SQL 文本——所有运行时值走 bind parameter；插值片段仅编译期常量（列名/方言关键字/`&'static str` 表按 enum/flag 选择）；配置来的 Postgres system-schema 标识符先按 `[A-Za-z_][A-Za-z0-9_]*` 校验；`tests/security.rs` 用恶意串（引号/语句终止符/DROP TABLE/注释符）扫全部字符串型 public 输入断言 verbatim round-trip
- **Secrets**：db URL 不 retain；Conductor API key 存于无 `Debug` 的 `ConductorConfig`（`compile_fail` doctest 钉死），连接失败只 log 传输错误
- **网络暴露面 opt-in**：AdminServer（inbound HTTP，无认证——与 DBOS SDK 对齐，需网络层防护；`start_on(loopback)` 可用）/ Conductor（outbound wss only，API key 在 connect URL）/ 数据库（outbound，推荐 TLS）

### 设计文档（事实）
- `docs/design/execution-layers.md`：**两层编排**——Workflow orchestration（步骤/child/position/重放，"replay makes exactly the decisions the original run made"）vs Runtime orchestration（task/ownership claim/queue/dispatcher/settlement，"at most one live execution per id"）；seam = `execution.rs`（durable-call 协议 + control-error 通道 `RecoveryRequired`/`ObservationFailed`）；"新行为属于哪一层"决策表
- `docs/design/explicit-context.md`：**显式 context vs task-local**（对比 DBOS 官方 Rust SDK `dbos-transact-rust`@7937413）——position 在构造时保留（0.5.0 前是 poll 时，曾有 select 乱序 bug）；spawned task 拒绝（原型 40 次运行 9 种 position 顺序）；`&DurableContext` 借用原型评估 94 文件迁移后推迟
- `docs/schema.md`：十表 ER 图 + 时间戳 BIGINT 约定

## 版本里程碑（CHANGELOG，事实）
- **0.5.0（2026-10-06）**：durable 调用在"书写处"claim checkpoint position（select!/乱序 await 不再重排历史）；越 body/spawned task 拒绝；checkpoint 写失败 → 恢复（非业务失败）；错误重放为同一错误；`StepCtx`（position/attempt/idempotency key/cancellation）；`transaction_on` 异步闭包；`verify_replay`；macros 0.2.0
- **0.4.0（2026-08-01）**：additive（宏不变）
- **已知 flake**：`pg_client_debounces`（#244）

## 与已有 Corpus 的连接（对照基线）
- **dbos-transact-py**（10-28 已考古，main）：同一 DBOS 语义 Python 实现——本考古核心对照（跨语言同一 schema/语义的实现差异）
- **官方 dbos-transact-rust**（未考古）：explicit-context.md 直接对比（task-local vs context 参数；step 放置规则差异表）——可作跨项目候选
- **restate / temporal / dapr-agents / substrate / infinity**：durable execution / 挂起恢复品类对照（corpus 已含或 open PR）
- **TeamMind 连接**（KnowlegeMap）：Java 首选 Restate；durare = Rust 轻量嵌入路线候选（DBOS 生态向 Rust 扩展）

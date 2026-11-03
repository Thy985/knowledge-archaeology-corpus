# 01 Project Layer — durare（项目地图）

> 全部为 L0 项目事实，可追溯到仓库实际内容（文件:符号）。证据上限 S3（已实现）/S4（测试断言存在）。

## 1.1 项目定位

| 项 | 事实 | 证据 |
|---|---|---|
| 全名 | durare（拉丁 durāre "to last"） | README.md:1 |
| 一句话 | DBOS 兼容 durable-execution Rust SDK | README.md:8-10 |
| 类别 | durable execution / workflow orchestration 库（嵌入式，非服务） | lib.rs:3-9 |
| 许可 | MIT OR Apache-2.0 双许可 | Cargo.toml license / LICENSE-MIT / LICENSE-APACHE |
| 维护状态 | 社区维护（非 DBOS 官方），与官方 `dbos-transact-rust` 并存 | README.md:19-20 |
| 版本 | 0.5.0（crates.io 同步） | Cargo.toml / CHANGELOG [0.5.0] - 2026-10-06 |
| 活跃度 | 11-01 仍有 fix commit（#252） | git log HEAD |
| 规模 | ~62k 行 Rust（src 33k + tests 27k + macros 0.5k + benchmarks 0.4k + examples 1.2k） | python3 统计 |
| 下载 | crates.io 1031 total / 878 recent | crates.io API 2026-11-04 |

## 1.2 架构总览（L0 事实）

```
┌─────────────────────────────────────────────────────────────┐
│                      应用进程（无服务器/sidecar）              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ DurableEngine（engine.rs）                             │  │
│  │  registry / launch / recover / shutdown / settle      │  │
│  │  Runtime（Arc 共享执行核：dispatchers/schedulers/tasks）│  │
│  └───────────────┬───────────────────────────────────────┘  │
│                  │ DurableContext（context.rs）              │
│                  │  step/transaction/select/sleep/send/recv  │
│                  │  /set_event/get_event/start/enqueue/patch │
│                  └───────────────┬──────────────────────────┘
│                                  │ StateProvider trait（provider.rs）
└──────────────────────────────────┼───────────────────────────┘
                                   ▼
        ┌─────────────────────────────────────────────┐
        │  PostgresProvider / SqliteProvider /        │
        │  InMemoryProvider（memory.rs）               │
        │  10 表 dbos 系统 schema（40/35 migrations）  │
        └─────────────────────────────────────────────┘
```

- **可选表面（feature opt-in）**：`AdminServer`（axum HTTP，`admin` feature）/ `Conductor`（wss 客户端，`conductor` feature）/ `Client`（无 registry 出站控制面，始终可用）
- **伴生 crate**：`durare-macros`（`#[workflow]`/`#[step]`/`#[transaction]` proc-macro，workspace member）

## 1.3 核心模块（L0 事实）

| 模块 | 文件 | 职责 |
|---|---|---|
| DurableEngine | src/engine.rs | 调度/恢复/子 workflow/fork/verify_replay/launch/shutdown/deactivate/settle |
| DurableContext | src/context.rs | workflow body 的全部 durable 调用（step/transaction/select/sleep/now/uuid/random/send/recv/events/start/enqueue/patch） |
| Execution | src/execution.rs | per-execution latch（RecoveryRequired/ObservationFailed）；两层编排唯一 seam |
| StateProvider | src/provider.rs | 存储抽象 trait + 状态常量 + 去重/目标冲突/角色编解码 + ChangeWait |
| 三后端 | src/memory.rs / sqlite.rs / postgres.rs | InMemory / SQLite / Postgres 实现 |
| ReplayReport | src/replay.rs | verify_replay 报告（recorded/matched/divergence） |
| serialize | src/serialize.rs | Serializer::Json（DBOS_JSON base64）/ Portable（portable_json）/ custom codec |
| recorded_error | src/recorded_error.rs | 版本化错误信封（remote derive，tag/content） |
| schedule | src/schedule.rs | 持久 cron 调度（workflow_schedules 表 + reconciler） |
| queue/queues | src/queue.rs + src/queues.rs | 持久队列（ENQUEUED 状态 + dispatcher claim） |
| debounce | src/debounce.rs | 基于既有 durable 原语的防抖（dedup id = key） |
| client | src/client.rs | 无 registry 出站控制面（enqueue/observe） |
| admin | src/admin.rs | AdminServer HTTP 端点（health/recovery/gc/timeout） |
| conductor | src/conductor.rs | DBOS 云控制面 wss 客户端（opt-in） |
| datasource | src/datasource.rs | Pg/Sqlite DataSource 抽象 |
| messaging | src/messaging.rs | 邮箱（send/recv）vs 事件（set/get）两类通信 |
| operations | src/operations.rs | 操作清单文档 |
| observability | src/observability.rs | tracing span 集成 |
| determinism | src/determinism.rs | 确定性规则书（body 内禁止源 → 替代） |
| durability | src/durability.rs | 执行模型（checkpoint/replay/at-least-once window） |
| transactions | src/transactions.rs | 事务步骤语义（SQL+checkpoint 同 commit） |
| tx | src/tx.rs | Tx 方言无关句柄 + Param/params! 宏 + 隔离级别 |
| security | src/security.rs | 信任地图（动态 SQL 不变量/secrets/网络面/授权） |
| error | src/error.rs | Error/ErrorCode（22 变体）/Result |
| handle | src/handle.rs | WorkflowHandle + WorkflowResultOptions |

## 1.4 生命周期（L0 事实，engine.rs）

```
DurableEngine::new(provider) / with_config / connect(url)
  → builder 注册（register / register_configured / register_queue / require_roles / listen_queues）
  → launch()
      ├─ 拒绝：deactivated / 无界 retention / required_roles 声明未注册 workflow
      ├─ 新 CancellationToken（shutdown 后可重启）+ tasks.reopen()
      ├─ create_application_version（注册本进程版本）
      ├─ queue 注册冲突策略（latest version 才可 overwrite）
      └─ 每 queue 一个 dispatcher + 每 schedule 一个 firing loop + 可选恢复 + retention sweeper
  → start / start_with（直接开始，insert created flag 仲裁）
    或 Client::enqueue（ENQUEUED 状态，fleet 认领）
  → recover() / recover_pending_for()（重派 PENDING：task tracker spawn，不 inline）
  → shutdown(timeout)（CancellationToken + TaskTracker drain in-flight）
  → deactivate()（停认领新活，服务 in-flight + admin，幂等）
```

## 1.5 状态机（L0 事实，provider.rs 常量）

```
ENQUEUED（等待 dispatcher claim）
DELAYED（到 delay_until_epoch_ms 转 ENQUEUED）
PENDING（已 claim，运行中）
SUCCESS（终态：output 已记录）
ERROR（终态：error 已记录）
CANCELLED（operator 终止；replay 拒绝）
MAX_RECOVERY_ATTEMPTS_EXCEEDED（恢复超限 park；显式 resume）
```

## 1.6 配置与特性（L0 事实）

- **features**：`default=["postgres","sqlite"]`；`conductor`（opt-in）；`admin`（opt-in）；零 backend 构建=编译错误
- **EngineConfig**：app_version（默认 sha-256 of executable）/ executor_id / recover_on_launch / retention（period/max_rows/sweep_interval）/ max_recovery_attempts
- **WorkflowOptions**：with_id / queue / authenticated_user / assumed_role / attributes / partition_key / dedup_id / dedup_policy / config_name / class_name / app_version / authenticated_roles
- **外部依赖**：tokio / serde / serde_json(preserve_order) / sqlx 0.8 / chrono / cron / uuid / sha2 / base64 / inventory / tokio-util(CancellationToken+TaskTracker) / fail(failpoints) ；opt：axum / tokio-tungstenite / flate2

## 1.7 测试体系（L0 事实）

- 61 个集成测试文件，27k 行；`tests/common/mod.rs` 共享 fixtures（recorded / hermetic_pg_db）
- 关键套件：crash_consistency（7 边界精确计数）/ adversity（多 executor 对抗）/ interop（跨 SDK golden strings）/ security（SQL 注入 sweep）/ durable_call_nesting / child_atomic（contract(db) 三后端）/ step_cancellation / recovery_liveness / concurrency
- CI 四门（AGENTS.md）：fmt / clippy -D warnings / test all-targets+doc / RUSTDOCFLAGS=-D warnings doc；外加 semver-checks 对 crates.io release + feature matrix
- 安全 CI（每周）：cargo audit + cargo deny；benchmark CI（每日 cron）：steps/concurrent/memory 三项
- 已知 flake：`pg_client_debounces`（#244）

## 1.8 权限与治理（L0 事实）

- **AGENTS.md**：完整工程治理（四门禁/Conventional Commits/PR 模板/边界/发布流程）
- **Authorization**：`require_roles` 声明式角色门槛（launch 时校验声明非空转）；AuthContext 贯穿 workflow/children/trace/持久化行
- **动态 SQL 不变量**：无 caller 字符串进 SQL 文本；schema 标识符 `[A-Za-z_][A-Za-z0-9_]*` 校验
- **Secrets**：db URL 不 retain；ConductorConfig 无 Debug（compile_fail doctest）
- **网络面**：AdminServer 无认证 by design（网络层防护）；Conductor 仅出站 wss

## 1.9 外部依赖与可移植性（L0 事实）

- 仅 4 个后端（InMemory/SQLite/Postgres + 未来 custom StateProvider trait）
- SQLite 需要 bundled C library（feature 控制）；Postgres 需要 network/TLS 驱动
- **跨 SDK 互操作**：与 dbos-transact-py/go/ts/java 共享 schema + portable_json（interop.rs golden strings pinned identically 在其他 SDK 测试套件）

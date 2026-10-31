# Project Layer — Restate（项目地图）

> L0 项目事实。全部条目可追溯到 repo 实际内容（file:line）。

## 1. 定位与入口
- 二进制入口：`server/`（restate-server，src/main.rs + lib.rs）、`cli/`（restate CLI）
- 核心 crate：`crates/core`（节点核心）、`crates/worker`（分区状态机）、`crates/bifrost`（日志系统）、`crates/partition-store`（持久化表）、`crates/types`（核心类型）
- 入口拓扑：ingress（HTTP/Kafka）→ worker 分区状态机 → invoker → SDK；日志（Bifrost）贯穿

## 2. 运行时架构（node）
- `crates/node/src/roles/`：**admin / ingress / worker** 三角色（admin.rs / ingress.rs / worker.rs）
- `crates/node/src/failure_detector.rs`：故障检测（Gossip、NodeStatus、GetClusterState）
- `crates/node/src/init.rs` / `republish.rs`：节点初始化与状态重发布
- 分片：`crates/worker/src/partition/`（分区处理器）+ `crates/sharding/` + `crates/types/src/partitions/`

## 3. 核心子系统
| 子系统 | 位置 | 职责 |
|---|---|---|
| 分区状态机 | `crates/worker/src/partition/state_machine/` | 幂等/调度/执行/终止/挂起恢复的唯一权威 |
| Invoker | `crates/invoker-impl/` | 驱动 SDK（service-protocol v4），重试、超时 |
| Bifrost | `crates/bifrost/` | 分布式日志（durable interconnect），loglet 抽象 + 日志链 |
| Partition Store | `crates/partition-store/` | RocksDB 持久化表（状态/journal/inbox/outbox/锁/定时器） |
| Metadata | `crates/metadata-store/` + `crates/core/src/metadata/` | 版本化元数据（schema/logs/nodes config） |
| VQueue | `crates/vqueues/` | keyed 调度（替代 inbox 的演进目标） |
| Ingress | `crates/ingress-http/`、`crates/ingress-kafka/` | 入口 + 幂等 key |
| Timer | `crates/timer-queue/`、`crates/partition-store/src/timer_table/` | Sleep 定时器 |

## 4. 核心数据结构
### 4.1 Journal（步骤持久化的核心）
- 老 journal：`crates/types/src/journal/entries.rs:176` EntryType 21 种（Input/Output/GetState/SetState/ClearState/GetStateKeys/ClearAllState/GetPromise/PeekPromise/CompletePromise/Sleep/Call/OneWayCall/Awakeable/CompleteAwakeable/Run/CancelInvocation/GetCallInvocationId/AttachInvocation/GetInvocationOutput/Custom）
- **journal_v2**（v1.6 起默认）：`crates/types/src/journal_v2/mod.rs:60` EntryType{Command|Notification}；`:147` Entry{Command|Notification}；`command.rs:50` Command 20 种（Input/Output/GetLazyState/SetState/ClearState/ClearAllState/GetLazyStateKeys/GetEagerState/GetEagerStateKeys/GetPromise/PeekPromise/CompletePromise/Sleep/Call/OneWayCall/SendSignal/Run/AttachInvocation/GetInvocationOutput/CompleteAwakeable）

### 4.2 InvocationStatus（storage-api/src/invocation_status_table/mod.rs:144，8 态）
Scheduled / Inboxed / Invoked / Suspended{metadata, awaiting_on} / Paused / Completed / Free（default）

### 4.3 InvocationTarget（types/src/invocation/mod.rs:208）
- Service{name, handler, scope} / VirtualObject{name, key, handler, handler_ty, scope} / Workflow{name, key, handler, handler_ty, scope}
- InvocationTargetType{Service, VirtualObject, Workflow}（:148）；`is_keyed()`=VirtualObject|Workflow（:157）；`can_write_state()`=exclusive handler|Workflow（:172）

### 4.4 幂等/分片
- `types/src/identifiers.rs:215 deterministic_partition_key`：service_key → idempotency_key → 随机
- `:245 UNSCOPED_SERVICE_PARTITION_KEY_FANOUT=512`（unscoped 服务在 512 个 key 上散布）
- `:253-268 controlled idempotent sharding`：bucket = H(idempotency_key) % 512 → 服务专属 key 集（保证同幂等 key 落到同分区）
- `:367 InvocationId::generate`：workflow=wf hash；idempotent=hash；否则随机 ULID

### 4.5 VQueue 调度（vqueues/src/util.rs:102）
`infer_vqueue_id_from_invocation`：Sha256(partition_key || ty || len(scope) || len(limit_key) || exclusive?"1":"0" || service || key)；Service→shared 队列、VirtualObject exclusive→独立队列（天然无锁串行）

## 5. 状态表与持久化（partition-store）
- invocation_status_table / journal_table / journal_table_v2 / inbox_table / state_table / promise_table / locks_table / outbox_table / deduplication_table / fsm_table / journal_events / timers
- 锁：`storage-api/src/lock_table/mod.rs:22 LockState{acquired_at, acquired_by}`；`types/src/locking.rs:137 CanonicalLockId`（scoped 渲染 `s/<scope>/<service>/<key>`，unscoped `u/<service>/<key>`）

## 6. 配置与治理
- AGENTS.md（applyTo '**'）：证据驱动、禁止臆测、cite file:line、`cargo nextest run`（非 cargo test）、WallClock 优先
- CLAUDE.md：同规则 + workspace 内自由读取 + 迭代证据搜寻（"Never speculate"）
- docs/dev/ 12 篇：development-guidelines / rust-guidelines / bilrost-migration-guidelines / release / build-and-ci / cla / debug / datafusion / local-development / profiling-debugging / release-testing
- .github/workflows/：18 个 CI workflow（ci/docker-*/helm/homebrew/npm/pullfrog/release/cla 等）

## 7. License
- **BSL 1.1**（Business Source License 1.1；Licensor: Restate Software, Inc., Restate GmbH；Change Date: 4 years after release；Change License: Apache 2.0）——LICENSE 文件头部原文

## 8. 测试体系
- `cargo nextest run`（AGENTS.md 指定）
- 核心测试：`crates/partition-store/src/tests/`（inbox/invocation_status/journal_events/journal/journal_v2/locks/state 表级测试）、`crates/worker/src/partition/state_machine/tests/`（idempotency.rs / workflow.rs / kill_cancel.rs / delayed_send.rs / fixtures.rs）、`crates/bifrost/src/loglet/loglet_tests.rs`、`crates/codederror/tests`、`server/tests/`

## 9. 外部依赖
RocksDB（crates/rocksdb）、DataFusion（storage-query-datafusion 查询引擎）、Kafka（ingress-kafka）、gRPC/bilrost（service-protocol，迁移中见 docs/dev/bilrost-migration-guidelines.md）、TLS（network/tls.rs）

## 10. 口径注记
- 静态考古（无 Docker 不编译）；"运行时行为"类条目标 Observation
- 本层为 L0 事实底座，升维内容见 02/03 层

# Project Layer — Restate 项目地图（L0 事实）

## 定位与历史
- 分布式 durable execution 运行时；2023-01 创建，v1.8.2-dev（workspace 版本），4609★/228 forks
- **License: BUSL-1.1**（LICENSE：Restate Software, Inc./GmbH；Public Restate Platform Service 禁止；内部部署/GUI/DSL 抽象层豁免；Change Date=release 后 4 年 → Apache-2.0）
- 三 SDK 生态：TypeScript / Java+Kotlin / Python / Go（README）；服务端用 service-protocol-v4 与 SDK 通信

## 架构总览（代码推导）
```
                    ┌─────────────────────────────────────────────┐
  HTTP/gRPC         │  Restate Server（Rust workspace）            │
  ─────────────►    │                                             │
  ingress-http      │  bifrost（日志段：local/replicated/memory）  │
  (SteerRouter)     │      │ append → Lsn（持久化确认点）           │
       │            │      ▼                                      │
       │            │  worker/partition processor（每分区唯一 leader）
       │            │  └─ FSM（last_applied_lsn / durable_point）  │
       │            │  └─ state_machine（ApplyJournalCommand）     │
       │            │  └─ invoker（AttemptState + RetryPolicy）    │
       │            │      └─ service-protocol-v4 ◄──► SDK 服务    │
       │            │  partition-store（RocksDB 13 表族目录）         │
       │            │  vqueues（虚拟对象串行化 + Scheduler）        │
       └───────────►│  metadata-store/server（集群元数据）          │
                    └─────────────────────────────────────────────┘
```

## 核心模块（crate）与职责
| crate | 职责 | 关键符号 |
|---|---|---|
| crates/bifrost | 日志段抽象：append 返回 Lsn；Loglet Provider 可换 | Bifrost/BifrostInner/Appender/Loglet/MaybeLoglet/PreferenceToken（bifrost.rs:67,293,670,699）；local_loglet（RocksDB）/replicated_loglet（sequencer+log_server_manager）/memory_loglet |
| crates/worker | 分区处理器：FSM 状态机 + 命令应用 + 重试 + 订阅 | partition/processor/{fsm,dedup,outbox,leadership,status}.rs；state_machine/entries/{call_commands,sleep_command,set_state_command,…}；lifecycle/{suspend,resume,cancel,purge}.rs |
| crates/partition-store | RocksDB 状态面：invocation 状态/journal/state/锁/定时器/去重 | invocation_status_table（InvocationStatusV2Lazy）；journal_table(v2)；state_table（ScopedStateKey）；locks_table（LockState）；deduplication_table（DeduplicationKey=partition+producer）；timer_table；vqueue_table；fsm_table（CachedEpochMetadata/PartitionDurability） |
| crates/core | 节点核心：task_center/partitions/network/metadata | task_center/{builder,extensions,handle,runtime,task_kind}.rs |
| crates/vqueues | 虚拟对象队列：按 key 串行 + 公平调度 | SchedulerService（scheduler.rs:132）；RefillMode（Async/Blocking）；VQueueHandle/VQueuesMeta |
| crates/invoker-impl | 调用 SDK 服务的 attempt 状态机 | InvocationStateMachine（AttemptState: New/InFlight/WaitingRetry）；RetryPolicyState |
| crates/types | 领域类型/配置/身份 | journal_v2（Command 20 种/Notification 12 类）；invocation（ServiceInvocation/IdempotencyId）；config（invocation/rocksdb/aws/bifrost…）；identifiers/sharding |
| crates/ingress-http | HTTP/gRPC 入口 | SteerRouter/Decision（ingestion/mod.rs:45,117）；handler/{awakeables,invocation,lookup,service_handler,workflow} |
| crates/wal-protocol | 分区间 WAL 命令 | v2/commands.rs（Invoke/PatchState/Terminate/Purge/Resume/RestartAsNew/InvokerEffect/Timer/InvocationResponse/NotifySignal/ProxyThrough 等） |
| crates/metadata-store/server | 集群元数据（分布式） | metadata_store.rs |
| crates/service-protocol-v4 | SDK 协议编解码 | ServiceProtocolV4Codec（entry_codec.rs:60）；discovery（ServiceDiscovery/DiscoveryError.is_retryable） |

## 核心数据结构
- **InvocationStatusV2Lazy**（partition-store/invocation_status_table，RocksDB value，InvocationId+Uuid）
- **JournalEntry = Entry | Completion**（storage-api/journal_table:28）；is_resumable（Entry 已 completion 则重放跳过）
- **ServiceInvocation**（types/invocation:572）：invocation_id/target/argument/source/幂等键/retention/limit_key/response_sink
- **Command**（journal_v2/command.rs）：Input/Output/GetLazyState/SetState/ClearState/GetEagerState/GetPromise/PeekPromise/CompletePromise/Sleep/Call/OneWayCall/SendSignal/Run/AttachInvocation/GetInvocationOutput/CompleteAwakeable——每个声明 related_completion_ids
- **Notification**（journal_v2/notification.rs）：SleepCompletion/CallCompletion/CallInvocationIdCompletion/RunCompletion/GetStateCompletion/CompletePromiseCompletion 等 12 类
- **Bifrost Commit**（bifrost/types.rs:77）；**DeduplicationKey**（partition_id+producer_id）
- **FSM 持久态**：last_applied_lsn / durable_point（PartitionDurability）/ rule_book / enabled_features（PersistedFeatures）

## 生命周期（Invocation）
ingress 请求 → SteerRouter（gRPC/HTTP 分流）→ ServiceInvocation（含幂等键/limit_key）→ 按 partition_key 路由 → partition processor（leader）→ InvokeCommand 入 WAL → FSM apply → invoker（AttemptState）→ SDK 执行 → 每副作用写 journal Command → completion 回填 → 完成/挂起（suspend：OnSuspendCommand 检查 notification 已可用则 resume 而非 suspend）→ retention 后 purge

## 状态机（Invocation 视角）
AttemptState: New → InFlight → WaitingRetry → InFlight …（RetryPolicy max_attempts）→ OnMaxAttempts: Pause（默认，挂起等待人工/信号）| Kill（终止）
Invocation 层：挂起（suspend.rs：转为 vqueue entry）→ resume（notification/signal 到达）

## 测试体系
- 196 个含 #[test] 文件；AGENTS.md 指定 `cargo nextest run`
- partition-store 表族单元测试；state_machine/tests；bifrost appender 测试（append_smoke/appends_correctly_handle_reconfiguration/append_record_too_large：bifrost.rs:801,1198,1310）
- e2e（internal 未展开）；release-notes/v1.8.x.md 记录行为变化

## 配置与治理
- 配置：crates/types/src/config/（admin/aws/bifrost/dynamodb_store/gcp/ingress/invocation/kafka/log_server/metadata_server/networking/object_store/query_engine/rocksdb/throttling/worker）
- RetryPolicy 配置（config/invocation.rs:156 initial_interval 等）；OnMaxAttempts 枚举（schema/invocation_target.rs:73 Pause=0 默认/Kill=1）
- 治理：AGENTS.md（证据引用/迭代搜索/nextest/不臆造）；docs/dev/{development-guidelines,rust-guidelines,build-and-ci,release}；clippy.toml+deny.toml
- 权限：无独立 RBAC 面（单二进制系统）；身份为 Restate 集群内部（node_id/epoch），对外 API 无认证层（开放核心定位）

## 外部依赖
arrow 59 / datafusion 55（query_engine）/ aws-sdk-dynamodb 1.123 / moka 0.12 / opentelemetry 0.33 / prost 0.14 / tonic（gRPC）/ RocksDB（restate_rocksdb）

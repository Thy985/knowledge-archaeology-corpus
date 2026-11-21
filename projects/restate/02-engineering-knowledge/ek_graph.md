# Engineering Knowledge Graph — Restate（L1/L2 宽底座）

> 每条 EK 声明 links（6 类边：mechanism/subsystem/causal/dependency/constraint/contrast）+ evidence（文件:行锚）。
> 配比：24 EK / 平均出边≥1 / 聚合覆盖率 100%（KO 见 03）。

## EK 列表

**EK-R01 Journal 即真相（副作用全持久化）**
- 内容：一切非确定性副作用（call/sleep/state/promise/signal/run）都必须先写 journal Command 再执行，SDK 通过 completions 回填结果；journal 是恢复时重放的唯一事实源
- links: mechanism→EK-R02（Completion 回填）· subsystem→EK-R03（journal_v2）· causal→EK-R06（重放）
- evidence: [code] crates/types/src/journal_v2/command.rs:55-77（Command 17 变体）；[code] storage-api/journal_table/mod.rs:28-53（JournalEntry=Entry|Completion；is_resumable）

**EK-R02 CompletionId 关联（entry 完成标记）**
- 内容：每个 Command 声明 related_completion_ids；completion 写入后 entry 视为完成（is_resumable=true），重放时跳过执行直接用结果
- links: mechanism→EK-R01 · causal→EK-R06（已完成的跳过）
- evidence: [code] journal_v2/command.rs:36-46（CommandMetadata::related_completion_ids）；[code] journal_v2/notification.rs:25-93（NotificationId/NotificationType）

**EK-R03 journal_v2 版本化（Entry=Command|Notification）**
- 内容：EntryType=Command|Notification 双分支（mod.rs:60）；v1/v2 journal_table 并存（storage-api 有 journal_table 与 journal_table_v2），v2 为当前路径
- links: subsystem→EK-R01 · contrast→EK-R04（journal v1 旧表）
- evidence: [code] types/journal_v2/mod.rs:60,147；[code] storage-api/journal_table_v2/mod.rs

**EK-R04 状态表派生而非冗余（state_table）**
- 内容：durable state（SetState/GetLazyState）由 journal Command 派生到 state_table（ScopedStateKey=service_id+key），非用户任意 KV——恢复时 journal 重放可重建状态，表只是物化视图
- links: causal→EK-R01（SetState command→表）· dependency→EK-R03 · contrast→EK-R07（Substrate 整机快照）
- evidence: [code] partition-store/src/state_table/mod.rs:43-83（StateKey/ScopedStateKey）

**EK-R05 幂等键三层组合（IdempotencyId）**
- 内容：幂等 = IdempotencyId::combine(invocation_id, target, key)；is_idempotent = 有 key 或 Workflow 类型；completion_retention_duration 决定完成状态保留期（0=不保留不去重）
- links: mechanism→EK-R18（dedup_table）· constraint→EK-R10（retention）
- evidence: [code] types/src/invocation/mod.rs:669-682（compute_idempotency_id/is_idempotent）；[code] invocation/mod.rs:572-620（ServiceInvocation 字段）

**EK-R06 崩溃恢复=journal 重放（partition 级）**
- 内容：processor 从 fsm_table 的 last_applied_lsn 起重放 bifrost 日志；已 completion 的 entry 跳过执行（幂等恢复）；durable_point 持久化进度
- links: causal←EK-R01 · dependency→EK-R19（bifrost Lsn）· constraint→EK-R22（leader epoch）
- evidence: [code] worker/src/partition/processor/fsm.rs:86-96（create 读 last_applied_lsn/durable_point）；[code] fsm.rs:31-37（字段）

**EK-R07 分区-领导者权威模型**
- 内容：数据按 PartitionKey 分片；每分区唯一 leader processor（epoch_metadata/announce_leader），从日志重放恢复状态后接管；leader 迁移不丢状态
- links: subsystem→EK-R06 · causal→EK-R22（epoch）
- evidence: [code] worker/src/partition/processor/fsm.rs:52-53（epoch_metadata）；[code] processor/commands/announce_leader.rs

**EK-R08 vqueues 虚拟对象串行化**
- 内容：Virtual Object = 按 key 的队列 + LockTable；同一 key 并发请求排队串行执行（单线程幻觉）；Scheduler 调度（RefillMode Async/Blocking 控制阻塞 IO 允许）
- links: mechanism→EK-R09（SchedulerService）· constraint→EK-R15（lock_table）· contrast→EK-R13（无对象服务=并行）
- evidence: [code] vqueues/src/scheduler.rs:47-57（RefillMode）；[code] vqueues/src/lib.rs:23-26（VQueueHandle/VQueuesMeta）

**EK-R09 Scheduler 公平调度与裁决**
- 内容：SchedulerService 每轮产出 Decisions（run/yield 计数），confirm_run_attempt 确认执行；on_inbox_event 触发调度；规则更新（on_rules_updated）
- links: mechanism→EK-R08 · causal→EK-R20（限流）
- evidence: [code] vqueues/src/scheduler.rs:71-132（Decisions/SchedulerService）；[code] scheduler.rs:183-227

**EK-R10 retention 语义（completion vs journal）**
- 内容：completion_retention_duration > journal_retention_duration；from_request 强制 journal ≤ completion（cmp::min）；0=不保留
- links: constraint→EK-R05 · causal→EK-R11（purge）
- evidence: [code] types/src/invocation/mod.rs:630-640（from_request）；[code] invocation/mod.rs:577-583

**EK-R11 purge 生命周期（terminate/purge/restart_as_new）**
- 内容：terminate 停执行，purge 清状态+journal，restart_as_new 保留身份换 journal（新 invocation id）；lifecycle/{purge,purge_journal,restart_as_new}.rs
- links: causal→EK-R10 · subsystem→EK-R06
- evidence: [code] worker/src/partition/state_machine/lifecycle/purge.rs；[code] wal-protocol/src/v2/commands.rs:98-117（PurgeInvocationCommand/PurgeJournalCommand）

**EK-R12 invoker attempt 状态机**
- 内容：InvocationStateMachine 维护 AttemptState（New/InFlight/WaitingRetry）+ RetryPolicyState；InFlight 持 abort_handle（可取消）；达到 max_attempts 触发 OnMaxAttempts 策略
- links: mechanism→EK-R13（重试策略）· causal→EK-R23（错误分类）· contrast→EK-R24（挂起语义）
- evidence: [code] invoker-impl/src/invocation_state_machine.rs:142-186（AttemptState）；[code] invocation_state_machine.rs:57-58

**EK-R13 重试策略（RetryPolicy/OnMaxAttempts）**
- 内容：initial_interval 可配（config/invocation.rs:156）；OnMaxAttempts=Pause（默认，挂起等待信号/人工）| Kill（终止）；重试迭代器来自 target_resolver.resolve_invocation_retry_policy
- links: mechanism→EK-R12 · constraint→EK-R20（限流并存）
- evidence: [code] schema/invocation_target.rs:73-78（OnMaxAttempts）；[code] config/invocation.rs:155-160；[code] invocation_state_machine.rs:303-314

**EK-R14 suspend/resume 语义（notification 优先）**
- 内容：OnSuspendCommand 先检查 notification 是否已可用——可用则 resume 而非 suspend（"Resuming instead of suspending…"）；suspend 转为 vqueue entry
- links: mechanism→EK-R12 · causal→EK-R08（vqueue entry）· contrast→EK-R24
- evidence: [code] worker/src/partition/state_machine/lifecycle/suspend.rs:32,80,125-138；[code] resume.rs:21

**EK-R15 lock_table（虚拟对象锁）**
- 内容：LockState 持久化在 lock_table；acquire_lock（scope+lock_name）；对象级并发控制与 vqueues 协同
- links: constraint→EK-R08 · subsystem→EK-R01
- evidence: [code] partition-store/src/locks_table/mod.rs:127（acquire_lock）；[code] locks_table/mod.rs:17,188

**EK-R16 bifrost 日志段抽象（Provider 可换）**
- 内容：Bifrost.append→Lsn；Loglet Provider 工厂注入（local RocksDB / replicated sequencer+log server / memory）；配置经 Live<Configuration>
- links: mechanism→EK-R17（replicated）· dependency→EK-R06（重放源）· contrast→EK-R19（单机 vs 集群）
- evidence: [code] bifrost/src/bifrost.rs:67-102（Bifrost/factory）；[code] bifrost/src/appender.rs:32-58；[code] bifrost/src/providers/local_loglet/provider.rs:32-37

**EK-R17 replicated_loglet（分布式复制日志）**
- 内容：sequencer 分配序号 + log_server 存储 + network；find_tail ConsistentRead 处理 sealed 边界（chain seal）；watchdog 维护
- links: mechanism→EK-R16 · causal→EK-R22（epoch 迁移）· subsystem→EK-R19
- evidence: [code] bifrost/src/providers/replicated_loglet/{sequencer,log_server_manager,network}.rs；[code] bifrost.rs:370-390（find_tail ConsistentRead/seal）

**EK-R18 dedup_table（生产者级去重）**
- 内容：DeduplicationKey=partition_id+producer_id→DedupSequenceNumber；防日志重放/重复投递
- links: mechanism→EK-R05 · constraint→EK-R06
- evidence: [code] partition-store/src/deduplication_table/mod.rs:33-50（create_key/get_dedup_sequence_number）

**EK-R19 Lsn 即持久化确认点**
- 内容：append 返回 Lsn（日志序号）；调用方以 Lsn 作为"已持久化"的确认，下游从 Lsn 读回
- links: dependency→EK-R06 · mechanism→EK-R16 · causal→EK-R21（durable_point）
- evidence: [code] bifrost/src/appender.rs:92（append→Result<Lsn>）；[code] bifrost/src/bifrost.rs:134,320

**EK-R20 RuleBook 限流（日志一致）**
- 内容：集群级规则经 Command::UpsertRuleBook 写日志跨副本一致；FSM 持 Arc<RuleBook>，apply 路径同时推 node 级 RuleBookCache；limiter 支持 key 级/用户级限制
- links: causal→EK-R09（调度输入）· mechanism→EK-R13 · subsystem→EK-R21
- evidence: [code] worker/processor/fsm.rs:56-60（rule_book 字段）；[code] processor/commands/upsert_rule_book.rs；[code] limiter/src/rule_book.rs:40-85

**EK-R21 durable_point（分区持久化进度）**
- 内容：PartitionDurability 持久化到 fsm_table；processor 恢复时读取；durable_lsn_tracking 表跟踪
- links: dependency→EK-R19 · subsystem→EK-R06
- evidence: [code] partition-store/src/durable_lsn_tracking.rs；[code] fsm_table（ReadFsmTable）

**EK-R22 epoch/leader 迁移（权威可转移）**
- 内容：epoch_metadata（CachedEpochMetadata）持久化分区配置状态（v1.6+）；leader 变更经 announce_leader 命令；日志重放保证新 leader 状态一致
- links: constraint→EK-R06 · causal→EK-R07 · mechanism→EK-R17
- evidence: [code] worker/processor/fsm.rs:52-53；[code] processor/commands/announce_leader.rs

**EK-R23 错误分类与重试性**
- 内容：DiscoveryError.is_retryable()（SDK 发现）；InvocationResponseCommand 携带结果；FAILED_PRECONDITION 类错误进 retry 决策
- links: causal→EK-R12 · subsystem→EK-R13
- evidence: [code] service-protocol-v4/src/discovery.rs:160（is_retryable）；[code] wal-protocol/src/v2/commands.rs:301（InvocationResponseCommand）

**EK-R24 挂起 vs 终止（生命周期两态）**
- 内容：suspend=可恢复暂停（journal 保留、vqueue 排队），terminate/kill=不可恢复；restart_as_new=换 journal 保留对象身份——三态语义分离
- links: contrast→EK-R12 · contrast→EK-R14 · subsystem→EK-R11
- evidence: [code] lifecycle/{suspend,resume,cancel,purge,restart_as_new}.rs；[code] wal-protocol/v2/commands.rs:98-136

## Graph 统计
- EK 24 条；links 声明全部 ≥1；六类边覆盖 mechanism/subsystem/causal/dependency/constraint/contrast
- 聚合覆盖率：24/24 → 8 个 KO（见 03，R1×2/R2×2/R3×2/R4×2）

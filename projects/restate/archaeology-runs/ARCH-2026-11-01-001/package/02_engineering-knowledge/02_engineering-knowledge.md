# Engineering Knowledge — Restate（EK Graph，L1/L2 宽底座）

> v3.1 规范：每条 EK 必须声明 links（≥1 边），KO 从图上的簇按 R1-R4 聚合。证据全部可回溯到 repo file:line。
> 证据强度标注：S3=已实现代码、S4=测试验证、S5=运行时（本考古无 Docker，未到 S5）；Observation=静态推断。

## EK Graph 边类型
`mechanism` 同机制 / `subsystem` 同子系统 / `causal` 因果(A→B) / `dependency` 依赖 / `constraint` 约束 / `contrast` 对照

---

### EK-01 Durable Journal 步骤持久化（journal_v2）
- **内容**：v1.6 起每步 handler 执行写 journal_v2 条目；崩溃后从最后完成 entry 恢复。20 种 Command（Input/Output/Sleep/Call/State/Promise/Run/AttachInvocation 等）。
- **证据**：`crates/types/src/journal_v2/command.rs:50`（Command 枚举）；`journal_v2/mod.rs:60/147`（EntryType/Entry）
- **强度**：S3
- **links**：`mechanism`→EK-02（append 原子性）；`causal`→EK-05（恢复从最后完成 entry 开始）

### EK-02 Journal 追加的原子性（无 gap 断言 + 同事务三写）
- **内容**：append_journal_entry 在同一调用中完成 put_journal_entry + 更新 journal_metadata.length + put_invocation_status；debug_assert journal_meta.length == entry_index（"journal should not have gaps"）。
- **证据**：`crates/worker/src/partition/state_machine/mod.rs:4968-5010`
- **强度**：S3（debug_assert 为开发期不变量，S4 部分测试覆盖）
- **links**：`mechanism`→EK-01；`constraint`→EK-12（幂等恢复依赖无 gap journal）

### EK-03 Pre-flight 写 Input（journal 先于执行）
- **内容**：InputCommand 在 pre-flight 阶段就写入 journal_v2（v1.6 起），保证调用"开始"即持久。
- **证据**：`crates/worker/src/partition/state_machine/mod.rs:729/827`（on_service_invocation / on_pre_flight_invocation）
- **强度**：S3
- **links**：`causal`→EK-02（pre-flight 触发 append）；`mechanism`→EK-01

### EK-04 幂等分派状态机（handle_duplicated_requests）
- **内容**：按 InvocationStatus 分派：Free=新调用；Invoked/Suspended/Paused/Inboxed/Scheduled=append response sink（不重复执行）；Completed=回放响应。
- **证据**：`crates/worker/src/partition/state_machine/mod.rs:1076`；`tests/idempotency.rs`（S4：start_and_complete_idempotent_invocation）
- **强度**：S3+S4
- **links**：`causal`→EK-12（幂等 key→确定性 ID）；`mechanism`→EK-05

### EK-05 响应合并（do_append_response_sink）
- **内容**：重复请求时把新 response sink 插入 invocation_status，不重跑 handler；多客户端共享同一执行结果。
- **证据**：`mod.rs:5247 do_append_response_sink`；`mod.rs:3004 send_response_to_sinks`
- **强度**：S3
- **links**：`mechanism`→EK-04；`subsystem`→EK-06

### EK-06 Ingress 直接响应 vs 分区处理器响应
- **内容**：response sink 分两类：PartitionProcessor→写 outbox（异步）；Ingress→直接回 HTTP（同步）。
- **证据**：`mod.rs:3004-3045 send_response_to_sinks`
- **强度**：S3
- **links**：`contrast`→EK-05（合并 vs 直发）；`subsystem`→EK-07

### EK-07 Workflow 幂等特例（忽略 idempotency key）
- **内容**：workflow run 忽略幂等 key；重复触发返回 `WORKFLOW_ALREADY_INVOKED_INVOCATION_ERROR`——workflow 用 workflow name+key 唯一标识。
- **证据**：`mod.rs:1076` 附近（workflow 分支）；`identifiers.rs:367-390`（InvocationId::generate workflow 分支）；`tests/workflow.rs`
- **强度**：S3+S4
- **links**：`contrast`→EK-04（workflow vs 普通幂等）；`mechanism`→EK-12

### EK-08 InvocationId 确定性生成
- **内容**：workflow→Sha256("wf"||scope||service||key)；幂等请求→scope+idempotency key hash；其他→随机 ULID。
- **证据**：`types/src/identifiers.rs:367-410`
- **强度**：S3
- **links**：`causal`→EK-12（确定性 ID 是幂等键）；`causal`→EK-11（确定性 ID 用于分区键）

### EK-09 确定性分区键（幂等路由到同分区）
- **内容**：deterministic_partition_key：service_key 优先，否则 idempotency_key；unscoped 非幂等用随机。
- **证据**：`types/src/identifiers.rs:215-232`
- **强度**：S3
- **links**：`causal`→EK-04（同分区才可能 dedup）；`mechanism`→EK-08

### EK-10 Unscoped 服务 512 扇出（幂等重分片约束）
- **内容**：UNSCOPED_SERVICE_PARTITION_KEY_FANOUT=512；unscoped 服务在 512 个服务专属 key 上散布；该常量对幂等服务"effectively immutable"（改动会重分片破坏去重）。
- **证据**：`identifiers.rs:245-251`（注释原文："changing it would re-shard idempotent invocations onto different partitions and break deduplication"）
- **强度**：S3（注释为设计意图，S2）
- **links**：`constraint`→EK-12（幂等键全局不变式）；`mechanism`→EK-09

### EK-11 Controlled Idempotent Sharding（桶内确定性）
- **内容**：bucket = H(idempotency_key) % 512 → unscoped_service_partition_key(service, bucket)；开关 CONTROLLED_IDEMPOTENT_SHARDING 控制。
- **证据**：`identifiers.rs:253-268`
- **强度**：S3
- **links**：`causal`→EK-09；`contrast`→EK-10（bucket 级 vs 全随机）

### EK-12 幂等生命周期四阶段
- **内容**：mod.rs:767-772 注释：1 dedup → 2 schedule → 3 inbox（virtual-objects exclusive handler）→ 4 execute。
- **证据**：`mod.rs:767-772`
- **强度**：S2（设计注释）+S3（实现）
- **links**：`causal`→EK-04；`subsystem`→EK-17

### EK-13 Suspend 先尝试解析（能解析不挂起）
- **内容**：OnSuspendCommand 先用现有 notifications resolve `awaiting_on`；能解析→直接 Resume 而非挂起。
- **证据**：`crates/worker/src/partition/state_machine/lifecycle/suspend.rs`
- **强度**：S3
- **links**：`contrast`→EK-14（先解析 vs 直接挂起）；`causal`→EK-16

### EK-14 挂起路径（SuspendedEvent + Suspended 状态 + vqueue 挂起）
- **内容**：不能解析时写 SuspendedEvent 到 journal_events、转 `InvocationStatus::Suspended{metadata, awaiting_on}`、挂起 vqueue entry；panic 分支提示 inbox→vqueue 迁移。
- **证据**：`suspend.rs`（panic 分支原文："Trying to suspend invocation … not exist as a vqueue entry. Have you forgotten to migrate from the old inbox to vqueues?"）
- **强度**：S3
- **links**：`causal`→EK-15（恢复）；`mechanism`→EK-13

### EK-15 Resume 移回 inbox 转 Invoked
- **内容**：ResumeInvocationCommand 通过 vqueue_move_invocation_to_inbox_stage 移回 inbox、转 Invoked 继续执行。
- **证据**：`crates/worker/src/partition/state_machine/lifecycle/resume.rs`
- **强度**：S3
- **links**：`causal`→EK-14；`mechanism`→EK-16

### EK-16 UnresolvedFuture 树（awaiting_on 组合）
- **内容**：awaiting_on 是树形 future：Single/FirstCompleted/AllCompleted/FirstSucceededOrAllFailed/AllSucceededOrFirstFailed/Unknown，对应 Promise.race/allSettled/any/all；PENDING→SUCCEEDED/FAILED 不可变；resolve 返回 bool（能解析即 resume）。
- **证据**：`crates/types/src/journal_v2/unresolved_future.rs`
- **强度**：S3
- **links**：`mechanism`→EK-13/14/15；`subsystem`→EK-20

### EK-17 VQueue 分派（keyed 队列）
- **内容**：infer_vqueue_id_from_invocation 用 Sha256 模板生成 VQueueId；exclusive handler 独立队列、shared service 共享队列（天然无锁串行）。
- **证据**：`crates/vqueues/src/util.rs:102-127`
- **强度**：S3
- **links**：`mechanism`→EK-12；`causal`→EK-18

### EK-18 DRR 调度器（Deficit Round Robin）
- **内容**：VQueue 调度用 DRR；与 ResourceManager（permit 资源控制）协同。
- **证据**：`crates/vqueues/src/scheduler/drr.rs`（文件头+结构）
- **强度**：S3（静态确认结构，未跑调度行为）
- **links**：`subsystem`→EK-17；`mechanism`→EK-19

### EK-19 VQueue 与 inbox 的迁移关系
- **内容**：vqueue 是 inbox 的演进（suspend panic 提示"migrate from old inbox to vqueues"；invocation 生命周期四阶段注释中 inbox 仅对 virtual-objects exclusive handler）。
- **证据**：`mod.rs:767-772`；`suspend.rs` panic 分支
- **强度**：S2（注释）+S3
- **links**：`contrast`→EK-17（inbox vs vqueue）；`causal`→EK-14

### EK-20 Bifrost 日志抽象（durable interconnect）
- **内容**：Bifrost 是"Restate's durable interconnect system"；loglet 抽象：append 返回 offset **必须 durable committed**、内部按 append 顺序缓冲。
- **证据**：`crates/bifrost/src/bifrost.rs`、`crates/bifrost/src/loglet.rs`
- **强度**：S3
- **links**：`mechanism`→EK-21；`causal`→EK-23（journal 依赖日志持久）

### EK-21 LogChainWriter（日志链 seal/extend）
- **内容**：seal_chain/extend_log_chain；MAX_BATCH_SIZE=1024；多次 chain 命令 coalesce 成一次 Logs 更新。
- **证据**：`crates/bifrost/src/log_chain_writer.rs`
- **强度**：S3
- **links**：`mechanism`→EK-20；`subsystem`→EK-22

### EK-22 ErrorRecoveryStrategy（append 失败应对）
- **内容**：Wait(1) / ExtendChainAllowed（default）/ ExtendChainPreferred——append 失败或观察到 sealed loglet 时的应对策略。
- **证据**：`crates/bifrost/src/bifrost.rs:50`
- **强度**：S3
- **links**：`constraint`→EK-21；`subsystem`→EK-20

### EK-23 Local vs Replicated Loglet（两种日志实现）
- **内容**：LocalLoglet（本地 RocksDB 落盘）与 ReplicatedLoglet（sequencer + log-server 复制，含 RemoteSequencer）；日志链可跨 segment 切换。
- **证据**：`crates/bifrost/src/providers/local_loglet/`、`crates/bifrost/src/providers/replicated_loglet/`
- **强度**：S3
- **links**：`contrast`→EK-22（两种恢复策略）；`mechanism`→EK-20

### EK-24 MetadataStore 版本化 kv
- **内容**：get/get_version/put/delete/provision；put/delete 带 Precondition 防并发覆盖。
- **证据**：`crates/metadata-store/src/metadata_store.rs:152`
- **强度**：S3
- **links**：`mechanism`→EK-25；`subsystem`→EK-26

### EK-25 Precondition（乐观并发控制）
- **内容**：Precondition{None, DoesNotExist, MatchesVersion(Version)}。
- **证据**：`crates/types/src/metadata.rs:55`
- **强度**：S3
- **links**：`constraint`→EK-24；`mechanism`→EK-27

### EK-26 MetadataManager 全局元数据同步
- **内容**：对 nodes_config / logs / partition_table / schema 四种 GlobalMetadata 做版本化同步（GlobalMetadataUpdateTask×4）。
- **证据**：`crates/core/src/metadata/manager.rs:136-189`
- **强度**：S3
- **links**：`subsystem`→EK-24；`causal`→EK-27（metadata 版本驱动分区加载）

### EK-27 Raft 链路由外部 provider 提供
- **内容**：metadata 链路的 raft 服务器由外部 provider 提供（代码注释提及），metadata-store 只做版本化 kv。
- **证据**：metadata.rs 注释 + MetadataManager 装配（crates/core/src/metadata/）
- **强度**：S2（注释）+S3
- **links**：`dependency`→EK-24/26

### EK-28 锁模型（LockState + CanonicalLockId）
- **内容**：LockState{acquired_at, acquired_by}；CanonicalLockId = scope+service/key（同 partition key 下串行化 virtual object 访问）；scoped 渲染 `s/<scope>/<service>/<key>`，unscoped `u/<service>/<key>`。
- **证据**：`crates/storage-api/src/lock_table/mod.rs:22`、`crates/types/src/locking.rs:137`
- **强度**：S3
- **links**：`mechanism`→EK-17（锁与 vqueue 共同保证串行）；`subsystem`→EK-29

### EK-29 状态表（state_table）+ 作用域键
- **内容**：state_table 支持 optional scope（OptionalScope），keyed 服务状态按 scope 隔离；scoped state 迁移（migrate_to_scoped_state_table）。
- **证据**：`crates/partition-store/src/state_table/mod.rs`、`crates/partition-store/src/migrations/migrate_to_scoped_state_table.rs`
- **强度**：S3
- **links**：`subsystem`→EK-28；`mechanism`→EK-30

### EK-30 存储迁移双轨（migrations/ + features/）
- **内容**：在线迁移用 PersistedFeatures（fsm_table 存 applied_lsn + PersistedFeatures），VersionBarrierCommand 传播 feature changes；本地迁移标记写 storage features（serde_json）。
- **证据**：`crates/partition-store/src/features/scoped_promise_migration.rs:56`、`fsm_table/mod.rs:103-120/264`
- **强度**：S3
- **links**：`mechanism`→EK-29；`constraint`→EK-31

### EK-31 fsm_table（applied_lsn + feature 标记）
- **内容**：分区 FSM 的持久化状态：applied_lsn 记录日志应用到哪；PersistedFeatures 记录已启用迁移。
- **证据**：`crates/partition-store/src/fsm_table/mod.rs:103-120`
- **强度**：S3
- **links**：`causal`→EK-32（恢复从 applied_lsn 续）；`mechanism`→EK-30

### EK-32 快照恢复（SnapshotPartitionTask + Lsn）
- **内容**：分区快照（SnapshotId+Lsn 元数据）用于恢复；min_target_lsn 控制恢复目标；latest_snapshot_lsn 汇报。
- **证据**：`crates/partition-store/src/snapshots/snapshot_task.rs:31-60`
- **强度**：S3
- **links**：`causal`→EK-31；`dependency`→EK-20（日志必须可重放到快照点）

### EK-33 定时器表（timer_table 按 timestamp 排序）
- **内容**：TimersKey(partition_id, timestamp, kind)；TimerKey 含 timestamp+kind，按时间排序扫描到期。
- **证据**：`crates/partition-store/src/timer_table/mod.rs`（define_table_key!）
- **强度**：S3
- **links**：`mechanism`→EK-34（Sleep 定时器）；`subsystem`→EK-35

### EK-34 Sleep 通过定时器实现（非阻塞挂起）
- **内容**：SleepCommand 落到 timer-queue（TimerKey 按 timestamp），到期后以 Completion/Notification 唤醒。
- **证据**：`crates/timer-queue/src/lib.rs`、journal_v2 SleepCommand
- **强度**：S3
- **links**：`causal`→EK-33；`mechanism`→EK-13（唤醒即 resume）

### EK-35 Invoker 状态机（重试/孤儿定时器防护）
- **内容**：AttemptState::WaitingRetry + retry_timer_key 防孤儿定时器；start_message_retry_count_since_last_stored_command 跟踪；can_retry 检查通知完整性。
- **证据**：`crates/invoker-impl/src/invocation_state_machine.rs:172-209`
- **强度**：S3
- **links**：`subsystem`→EK-33；`mechanism`→EK-36

### EK-36 Retry-After 头解析（HTTP 语义透传）
- **内容**：parse_retry_after 支持秒数与 RFC2822 日期两种格式；无效返回 None（走默认重试策略）。
- **证据**：`crates/invoker-impl/src/invocation_task/retry_after.rs`
- **强度**：S3+S4（单测覆盖两种格式）
- **links**：`mechanism`→EK-35；`constraint`→EK-37

### EK-37 Service Protocol v4（SDK 交互协议）
- **内容**：discovery/entry_codec/message_codec/serdes；bilrost 迁移中（docs/dev/bilrost-migration-guidelines.md）。
- **证据**：`crates/service-protocol-v4/src/`、`docs/dev/bilrost-migration-guidelines.md`
- **强度**：S3
- **links**：`subsystem`→EK-35；`mechanism`→EK-38

### EK-38 InvocationTarget 版本兼容（bilrost Oneof 静默降级）
- **内容**：注释："Do not add a variant without skipping a version first"—旧节点解码未知 tag 回退为 Service（静默当普通服务），因此新 variant 必须跳版本。
- **证据**：`crates/types/src/invocation/mod.rs:145-147`
- **强度**：S2（设计注释，S3 实现）
- **links**：`constraint`→EK-37；`mechanism`→EK-39

### EK-39 幂等分片的不可变常量（跨版本不变量）
- **内容**：UNSCOPED_SERVICE_PARTITION_KEY_FANOUT 幂等服务侧不可变；改变会破坏幂等重分片（注释明确）。
- **证据**：`identifiers.rs:245-251`
- **强度**：S2（注释）
- **links**：`constraint`→EK-09/10/11

### EK-40 Outbox（异步消息出站）
- **内容**：outbox_table 存待发送消息；Outbox enqueue 到 outbox + fsm（OutboxMut）。
- **证据**：`crates/worker/src/partition/processor/outbox.rs:21-83`
- **强度**：S3
- **links**：`mechanism`→EK-06；`subsystem`→EK-31

### EK-41 分区处理器 FSM（日志→状态机循环）
- **内容**：processor fsm：last_applied_lsn 跟踪 + applied_lsn 持久化（set_last_applied_lsn 写 fsm_table）；replay 状态机处理。
- **证据**：`crates/worker/src/partition/processor/fsm.rs:29-161`
- **强度**：S3
- **links**：`causal`→EK-32；`mechanism`→EK-31

### EK-42 Partition-wide 命令分离
- **内容**：AnnounceLeader/UpdatePartitionDurability/UpsertRuleBook/UpsertSchema/TruncateOutbox/VersionBarrier 必须走 apply_partition_command 路径，state_machine 内 panic 提示。
- **证据**：`mod.rs:424-440`（on_apply 内 panic 分支）
- **强度**：S3
- **links**：`constraint`→EK-41；`subsystem`→EK-30

### EK-43 Deduplication Table（Kafka 幂等去重）
- **内容**：deduplication_table 存 ProducerId + DedupSequenceNumber（Kafka 类 ingress 去重）；ProducerId{Unknown, String, Numeric}。
- **证据**：`crates/partition-store/src/deduplication_table/mod.rs`、`crates/types/src/net/ingest.rs:130-136`
- **强度**：S3
- **links**：`contrast`→EK-04（生产者去重 vs 调用幂等）；`subsystem`→EK-44

### EK-44 Ingress 双入口（HTTP + Kafka）
- **内容**：ingress-http（REST + idempotency header）+ ingress-kafka（事件流）；HTTP 侧幂等经 IdempotencyId 查询路径。
- **证据**：`crates/ingress-http/src/handler/invocation.rs:96-101`、`crates/ingress-kafka/`
- **强度**：S3
- **links**：`contrast`→EK-43；`subsystem`→EK-01（调用入口）

---

## EK Graph 汇总（供 KO 聚合与质量门）
| 指标 | 值 |
|---|---|
| EK 总数 | 44 |
| 带 links 数 | 44/44（100%） |
| 孤立 EK | 0 |
| 平均出边 | ~2.1 |
| 主要簇 | 幂等簇（EK-04/05/08/09/10/11/12/39）、挂起恢复簇（EK-13/14/15/16/34）、调度簇（EK-17/18/19/28）、日志簇（EK-20/21/22/23/32）、元数据簇（EK-24/25/26/27）、迁移簇（EK-29/30/31/42） |

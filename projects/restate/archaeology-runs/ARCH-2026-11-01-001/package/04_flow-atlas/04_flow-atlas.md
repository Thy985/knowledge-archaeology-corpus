# Flow Atlas — Restate（七类流）

> 从真实代码导出，关键 Edge 可回溯 symbol/file/condition/state transition。
> 符号：`→` 流程边；`[file:line]` 证据；`‖` 并行。

## 1. Control Flow（控制流：调用如何进入并驱动执行）
```
HTTP Ingress [ingress-http/src/handler/invocation.rs]
  → InvocationTarget 解析（含 IdempotencyId 查询路径 [:96-101]）
  → ServiceInvocation 命令
  → 分区处理器 FSM [worker/src/partition/processor/fsm.rs]
  → on_apply → 状态机 [state_machine/mod.rs:414]
    → pre-flight（写 Input journal_v2）[:827] → on_service_invocation [:729]
    → invoker 启动（service-protocol v4 SDK 调用）[invoker-impl/invocation_task/service_protocol_runner_v4.rs]
    → InvokerEffect（journal entry 回传）
    → 状态机 apply effect → 推进 InvocationStatus
  → OutputCommand → 响应 sink [send_response_to_sinks :3004]
```
关键条件：VQSchedulerDecisions（vqueue 调度决策）与 InvokerEffect 两条命令流在 on_apply 汇合 [:464-540]。

## 2. State Flow（状态流：keyed 状态读写）
```
GetLazyState / GetEagerState / SetState / ClearState / ClearAllState 命令
  → state_table [partition-store/src/state_table/mod.rs]（OptionalScope keyed）
  → VirtualObject 才有读权限（is_keyed）[types/src/invocation/mod.rs:157]
  → Exclusive handler / Workflow 才有写权限（can_write_state）[:172]
  → 状态变更经 journal entry 记录 → 崩溃恢复重建状态
```
不变式：只有 keyed 调用可读写状态；写权限收窄到 exclusive/workflow。

## 3. Data Flow（数据流：journal 数据的产生与消费）
```
InputCommand（pre-flight 写）→ ... → 每步效果写 journal entry
  → append_journal_entry [mod.rs:4968]（put_journal_entry + length 更新 + put_invocation_status 同调用）
  → journal_table_v2 持久化 [partition-store/src/journal_table_v2]
  → 崩溃恢复：读 journal 从 entry 0 → last completed → 续跑
  → 完成时 OutputCommand → 响应；journal 清理（do_drop_journal）
```
journal v1（21 种 EntryType）与 journal v2（20 Command + Notification）双表并存，按 pinned protocol version 选择 [do_drop_journal 分支]。

## 4. Evidence Flow（证据流：持久化与恢复的证据链）
```
Bifrost 日志（durable interconnect）[bifrost/src/bifrost.rs]
  → loglet append 返回 offset=durable committed [loglet.rs]
  → 分区状态机消费 → fsm_table applied_lsn [partition-store/src/fsm_table/mod.rs]
  → 快照 SnapshotPartitionTask(SnapshotId+Lsn) [snapshots/snapshot_task.rs]
  → 恢复：从快照 + 日志重放到 min_target_lsn
```
恢复锚点：last_applied_lsn 在 fsm_table 持久化，processor fsm 启动时读取 [processor/fsm.rs:94]。

## 5. Authority Flow（权威流：谁有权限做什么）
```
InvocationTargetType::can_write_state = Exclusive | Workflow [types/src/invocation/mod.rs:172]
  → 锁：LockState{acquired_at, acquired_by} [storage-api/lock_table/mod.rs:22]
  → CanonicalLockId（scope+service/key）[types/src/locking.rs:137]
  → VQueue 分派（exclusive 独立队列 → 天然串行）[vqueues/src/util.rs]
  → 元数据 Precondition 防并发覆盖（MatchesVersion）[types/src/metadata.rs:55]
```
双权威机制：队列串行（结构性权威）+ 显式锁（资源性权威）。

## 6. Memory Flow（记忆流：持久状态如何被记住）
```
partition-store RocksDB 表：
  invocation_status（当前生命周期）→ journal（步骤历史）→ state（业务状态）
  → inbox/vqueue（待处理）→ outbox（待发送）→ timers（待唤醒）→ promises/awakeables
  → deduplication（幂等痕迹）→ locks（锁状态）
  → 快照机制将 RocksDB 状态固化（SnapshotId+Lsn）
```

## 7. Policy Flow（治理流：决策→批准→策略→执行→后续决策）
```
AGENTS.md/CLAUDE.md（证据驱动、cite file:line、nextest）→ 开发期策略
  → CI 18 workflows（ci/release/cla/helm/docker/npm/homebrew）→ 合入门禁
  → 运行时策略：InvocationStatus 状态机（Free→Scheduled→Inboxed→Invoked→Suspended→Completed）
    [storage-api/invocation_status_table/mod.rs:144]
  → 幂等策略（handle_duplicated_requests 分派）[mod.rs:1076]
  → 重试策略（WaitingRetry + Retry-After 解析）[invoker-impl]
  → 迁移策略（PersistedFeatures + VersionBarrierCommand）[partition-store/src/features/]
```
治理闭环：决策（设计文档/docs/dev + CLAUDE 规则）→ 执行约束（状态机/权限）→ 验证（CI+测试）→ 反馈（失败修复再考古）。

## Flow→KO 交叉校验
| KO | 支撑 Flow Edge |
|---|---|
| KO-01 步骤持久化 | Data Flow（journal 生产/消费/恢复） |
| KO-02 幂等第一公民 | Control Flow（handle_duplicated_requests 分派）+ State Flow |
| KO-03 幂等分片 | Authority/Memory（deterministic_partition_key→同分区） |
| KO-04 挂起/恢复 | Control Flow（suspend/resume 转换） |
| KO-05 组合 future 树 | Control Flow（awaiting_on 解析即恢复） |
| KO-06 keyed 无锁串行 | Authority Flow（vqueue 分派 + 锁） |
| KO-07 日志与状态分离 | Evidence Flow（loglet durable + fsm applied_lsn） |
| KO-08 版本化元数据 | Authority Flow（Precondition） |
| KO-09 迁移即一等公民 | Policy Flow（PersistedFeatures + VersionBarrier） |

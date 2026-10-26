# Flow Atlas — Restate 七类流（代码导出）

> 关键 Edge 可回溯 symbol/file/condition/state transition。来源：代码，非文档。

## 1. Control Flow（控制流：执行如何推进）
```
ingress (SteerRouter) → InvokeCommand → bifrost append(Lsn) → partition processor FSM.apply
  → state_machine (ApplyJournalCommandEffect) → invoker (AttemptState) → SDK → journal Command 追加
  → Completion 回填 → 继续/挂起/完成 → purge
```
- Edge C1: ingress 分流 gRPC/HTTP —— SteerRouter/Decision（ingress-http/ingestion/mod.rs:45,117; is_grpc_request:135）
- Edge C2: 命令入 WAL —— wal-protocol/v2/commands.rs:136 InvokeCommand
- Edge C3: FSM 顺序应用 —— processor/fsm.rs（last_applied_lsn 单调推进）
- Edge C4: 挂起转 vqueue —— lifecycle/suspend.rs:125-138（suspend_entry）
- Edge C5: resume 优先 —— suspend.rs:80（notification 可用则 resume 而非 suspend）

## 2. State Flow（状态流：状态如何变化/持久化）
```
InvocationStatus（InvocationStatusV2Lazy，RocksDB）
  New → InFlight → WaitingRetry → … → Completed/Purged
FSM: last_applied_lsn / durable_point / rule_book / enabled_features（fsm_table 持久化）
Virtual Object: vqueue entry 状态 + LockState（lock_table）
```
- Edge S1: invocation 状态持久化 —— partition-store/invocation_status_table/mod.rs:39-53
- Edge S2: FSM 恢复读 —— processor/fsm.rs:86-96（create）
- Edge S3: 对象锁 —— locks_table/mod.rs:127（acquire_lock）
- Edge S4: durable_point —— partition-store/durable_lsn_tracking.rs

## 3. Data Flow（数据流：payload 如何移动）
```
请求体 → ServiceInvocation.argument(Bytes) → journal Input entry → SDK 反序列化 → 处理
副作用结果 → Completion 回填 → journal_table
durable state → state_table（ScopedStateKey=service_id+key）
```
- Edge D1: 参数入 journal —— types/invocation/mod.rs:572-580（ServiceInvocation.argument）
- Edge D2: state 派生写表 —— partition-store/state_table/mod.rs:43-83
- Edge D3: completion 关联 —— journal_v2/command.rs:36-46（related_completion_ids）
- Edge D4: EntryLite SQL 查询 —— journal_v2/lite.rs:25-40

## 4. Evidence Flow（证据流：什么算"已发生"）
```
journal Command 追加成功（append→Lsn）= 副作用意图已记录
Completion 写入 = 副作用结果已确定
last_applied_lsn = processor 已应用到的位置（可恢复检查点）
dedup_sequence_number = producer 投递去重证据
```
- Edge E1: append→Lsn 即持久化证据 —— bifrost/appender.rs:92
- Edge E2: 重放跳过已 completion —— storage-api/journal_table/mod.rs:28-53（is_resumable）
- Edge E3: 去重证据 —— deduplication_table/mod.rs:33-50
- Edge E4: FSM 进度证据 —— fsm.rs:31-37（last_applied_lsn）

## 5. Authority Flow（权威流：谁有权推进状态）
```
分区 leader processor（epoch）是唯一状态推进权威
  ├─ ingress 请求 → 路由到分区（无本地决策权）
  ├─ invoker 仅执行（无状态权威）
  └─ metadata-server 维护集群拓扑（分区→节点映射）
```
- Edge A1: leader 权威 —— announce_leader.rs；fsm.rs:52-53（epoch_metadata）
- Edge A2: 日志即权威来源 —— bifrost.rs:67-102（Bifrost/factory）
- Edge A3: 跨分区代理 —— wal-protocol/v2/commands.rs:392（ProxyThroughCommand）

## 6. Memory Flow（记忆流：什么被记住/遗忘）
```
journal_table（每步执行历史，按 retention 保留）—— 过程记忆
state_table（durable state）—— 语义记忆（对象状态）
promise_table / awakeables —— 跨调用记忆（异步结果）
retention 到期 → purge（遗忘）
```
- Edge M1: 过程记忆 —— journal_table(v2)
- Edge M2: 语义记忆 —— state_table/mod.rs:43-83
- Edge M3: 异步结果 —— promise_table；journal_v2/command.rs:55-77（Promise/Signal/Awakeable）
- Edge M4: 遗忘 —— lifecycle/purge.rs；commands.rs:98-117

## 7. Policy Flow（策略流：治理闭环）
```
RetryPolicy（initial_interval/max_attempts/OnMaxAttempts）→ 执行失败策略
RuleBook（UpsertRuleBook 日志一致）→ 分区级限流策略
LimitKey（层次化并发控制）→ 调用级策略
```
- Edge P1: 重试策略配置 —— config/invocation.rs:155-160；schema/invocation_target.rs:73-78
- Edge P2: 规则传播 —— processor/commands/upsert_rule_book.rs；fsm.rs:56-60
- Edge P3: 限流决策 —— limiter/src/rule_book.rs:40-85；vqueues/scheduler.rs:191（on_rules_updated）

## Flow→KO 交叉校验
- 每条 KO 可回溯 ≥1 Flow Edge：KO-R01↔E1/E2；KO-R03↔E3；KO-R04↔S3/C4；KO-R05↔A1/A2；KO-R06↔E1；KO-R07↔C4/C5；KO-R08↔P1/P2/P3 —— 全部通过（无矛盾）

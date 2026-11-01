# 04 · Flow Atlas（七类流）— Temporal（ARCH-2026-11-02-001）

> 七类流：Control / State / Data / Evidence / Authority / Memory / Policy。
> 每条 Edge 从真实代码导出，标注 symbol/file 可回溯；关键 Edge 附条件/状态迁移。

---

## F1 · Control Flow（控制流）— 一次 Workflow 任务执行的完整路径

```
用户App ──StartWorkflowExecution──▶ Frontend ──▶ History.StartWorkflowExecution
   └─▶ CreateWorkflowExecution(持久化: History初始化[Started,Scheduled]+MutableState+TransferTask)
        └─▶ QueueProcessor.GetHistoryTasks ──▶ ProcessTask ──▶ Matching.AddWorkflowTask
             └─▶ Worker 执行 workflow 代码（确定性）
                  └─▶ RespondWorkflowTaskCompleted(commands)
                       └─▶ workflowTaskCompletedHandler.handleCommands
                            ├─ ValidateCommandSequence（命令序列校验）
                            ├─ handleCommand（switch 13+ 命令 → 生成 historyEvent）
                            ├─ handleMessage（update protocol 消息）
                            ├─ handleEagerActivityCandidates（eager 分派优化）
                            └─ flushWorkerCommandsTasks ──▶ 持久化事件+状态
```

- **关键 Edge 证据**：`service/history/history_engine.go` L443（StartWorkflowExecution）、L617（RespondWorkflowTaskCompleted）；`workflow_task_completed_handler.go` L177-225（handleCommands）。
- **分支/失败路径**：校验失败→整体拒绝（stopProcessing）；persistence 失败→回滚（historySizeRollback）。

## F2 · State Flow（状态流）— Mutable State 与 Event History

```
Event History（不可变追加）  ←── 命令生成事件（EK-01/02）
        │ 重放
        ▼
MutableStateRebuilder.ApplyEvents ──▶ applyHSMEvent / applyChasmEvent（双引擎）
        │
        ▼
Mutable State（pending activity/timer/signal/child/VersionedTransition/HSM/ChasmTree）
        │
        ▼
持久化：UpdateWorkflowExecution（带 RangeID CAS + VersionedTransition 幂等）
```

- **关键 Edge 证据**：`mutable_state_rebuilder.go` L72/L748-810；`execution_manager.go` L212+（UpdateWorkflowExecution）。
- **状态迁移**：Workflow Task Scheduled→Started→Completed/Failed/TimedOut（`workflow_task_state_machine.go` L62-260）。

## F3 · Data Flow（数据流）— 事件与状态的序列化持久化

```
Command 属性（proto）
   └─▶ ValidateCommandSequence（命令级校验）
   └─▶ serializeWorkflowEventBatches（事件批量序列化：NewWorkflowNewEvents/historyDiff）
        └─▶ Create/UpdateWorkflowExecution（persistence）
             └─▶ NewWorkflowSnapshot / InternalCreateWorkflowExecutionRequest
                  └─▶ sql/cassandra/nosql 插件写入
   └─▶ historySizeRollback（失败回滚 size 统计）
```

- **关键 Edge 证据**：`execution_manager.go` L158-212（serializeWorkflowEventBatches + rollback + Create 路径）。
- **数据不变性**：Event History append-only；branch token 分版本（`history_branch_util.go`）。

## F4 · Evidence Flow（证据流）— 权威事实的生成与消费

```
Worker Command ──(证据：命令意图)──▶ History 校验（attrValidator）
   ──(裁决：event 追加)──▶ Event History（唯一权威事实源）
   ──(派生：apply events)──▶ Mutable State（当前状态视图）
   ──(消费：重放)──▶ Worker 恢复
   ──(复制：事件批)──▶ standby shard（NDC 复制）
```

- **关键 Edge 证据**：`workflow_task_completed_handler.go` L177（ValidateCommandSequence）；`mutable_state_rebuilder.go`（apply 链）；`state_transition_history.go`（版本化）。
- **权威判定**：History 服务是唯一裁决者；worker 只提议（对照 KO-03）。

## F5 · Authority Flow（权威流）— 谁有权写状态

```
用户请求 ──▶ Frontend（认证/授权：common/auth + common/authorization，namespace 级）
   ──▶ History（shard 所有权：RangeID CAS，ShardOwnershipLostError）
        ──▶ persistence（RangeID 条件更新）
   ──复制路径──▶ standby（应用复制事件，不经 CAS）
   ──Chasm──▶ Registry 注册的 component 才可执行（FQN↔ID）
```

- **关键 Edge 证据**：`common/persistence/data_interfaces.go` L156-207（ShardOwnershipLostError/AssertShardOwnership）；`common/authorization/`；`chasm/registry.go` L75-110。
- **边界/例外**：NDC 复制路径 bypass CAS（EK-22）；admin API 直连路径（admin_handler）。

## F6 · Memory Flow（记忆流）— 缓存的派生状态

```
namespace 缓存（common/namespace）──▶ 租户隔离/配置解析
   ├─ dynamicconfig 缓存（findMatchWithCache：weak 引用 + runtime.AddCleanup 防泄漏）
   ├─ worker_versioning 路由缓存（routing_info_cache / version_membership_cache）
   └─ searchattribute 缓存（refreshCache 周期刷新）
```

- **关键 Edge 证据**：`common/dynamicconfig/collection.go` L237-260（findMatchWithCache + AddCleanup）；`common/worker_versioning/routing_info_cache.go`；`common/searchattribute/manager.go` L91-113。
- **失效语义**：dynamicconfig 轮询变化（pollForChanges）；searchattribute 按时间刷新（needRefreshCache）。

## F7 · Policy Flow（策略流）— 治理闭环

```
决策（设计/PRD：docs/architecture/*.md + AGENTS.md 自治规则）
   ──▶ 策略固化（动态配置 Key：EnableChasm / ChasmSkipPersistenceEnabled / ...）
   ──▶ 执行（mutable_state 读取配置 → Chasm 行为分支）
   ──▶ 验证（tests/ + make lint/proto + CI）
   ──▶ 未来决策（开关矩阵迭代：每能力面一个开关）
```

- **关键 Edge 证据**：`AGENTS.md`（开发治理）；`common/dynamicconfig/constants.go` L3438+（EnableChasm 等）；`mutable_state_impl.go` L695-746（配置读取）；`tests/chasm_test.go`（验证）。
- **治理闭环特征**：策略→执行→验证→反馈（对照 skill 的 Policy Flow 第七类流定义）。

---
## Flow Atlas 质量自检

| 流 | Edge 数 | 关键 Edge 可回溯 | 失败/例外路径 |
|----|---------|----------------|--------------|
| Control | 8 | ✓ | ✓（stopProcessing/回滚） |
| State | 6 | ✓ | ✓（复制中间态） |
| Data | 5 | ✓ | ✓（rollback） |
| Evidence | 5 | ✓ | ✓（standby 复制） |
| Authority | 5 | ✓ | ✓（NDC bypass CAS） |
| Memory | 4 | ✓ | ✓（缓存失效） |
| Policy | 5 | ✓ | ✓（配置故障面） |

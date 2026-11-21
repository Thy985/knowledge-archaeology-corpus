# 02 · Engineering Knowledge（EK Graph）— Temporal（ARCH-2026-11-02-001）

> 宽底座：40 条 L1/L2 工程知识，每条带 `links`（6 类边）+ 证据来源（symbol/file，可回溯）。
> 图结构：机制簇（mechanism）、子系统（subsystem）、因果（causal）、依赖（dependency）、约束（constraint）、对照（contrast）。

## EK-01 · Command→Event 两阶段闭环（核心执行机制）

- **层级**：L2（机制）
- **内容**：Worker 不直接写状态；每个副作用意图作为 `Command` 提交（handleCommands），History 校验（attrValidator.ValidateCommandSequence）后逐命令 `handleCommand`（switch 按 CommandType 分发 13+ 种命令）→ 生成 `historyEvent` → 追加 Event History 并更新 Mutable State。
- **证据**：`service/history/api/respondworkflowtaskcompleted/workflow_task_completed_handler.go` L177-225（handleCommands 主循环）、L276-390（handleCommand switch）。
- **盲重建修正（2026-11-02 IVR）**：独立 grep 实测 handleCommand 共 **15 个 case**（含 `COMMAND_TYPE_PROTOCOL_MESSAGE`，协议消息命令；业务命令 14 种）。本 EK 原表述"13+ 种"精确化为 **15 种（含 PROTOCOL_MESSAGE）**。
- **links**：causal → EK-02（事件追加）→ EK-03（重放恢复）；contrast ↔ EK-07（Chasm 的 Transition）；subsystem: history-api

## EK-02 · Event History 不可变追加 + Mutable State 派生（状态真相单一来源）

- **层级**：L2
- **内容**：Event History 是唯一权威事实源（append-only，branch token 支持 reset 分支）；Mutable State 是当前状态的**派生视图**（内存+持久化快照）。恢复时从 Event History 重建 Mutable State（MutableStateRebuilder.ApplyEvents 逐事件 apply，含 HSM/Chasm 双路径 applyHSMEvent/applyChasmEvent）。
- **证据**：`service/history/workflow/mutable_state_rebuilder.go` L72（ApplyEvents）、L748-810（applyStateMachineEvent/applyHSMEvent/applyChasmEvent）；`common/persistence/history_branch_util.go`。
- **links**：mechanism ↔ EK-03；dependency：EK-04（RangeID CAS）保证追加原子；subsystem: history-workflow

## EK-03 · 确定性重放恢复（崩溃恢复=读历史，不重放副作用）

- **层级**：L2（机制）
- **内容**：workflow 崩溃后新 worker 拿到完整 Event History **重放 workflow 代码**；已完成命令在重放中"跳过执行"（Event 已存在，命令结果从 History 取，不再执行副作用）。这是 Replay-based 模型与 Restate（journal 步骤持久化）、agent-substrate（挂起恢复）的本质区别。
- **证据**：`docs/architecture/workflow-lifecycle.md`（571 行，生命周期文档）；MutableStateRebuilder（重放路径）；Command 校验（已存在 Event 的命令在 replay 中透传）。
- **links**：mechanism ↔ EK-02；constraint：EK-09（确定性约束）是重放正确性的前置；subsystem: 执行模型

## EK-04 · RangeID/shard 所有权 CAS（并发控制与脑裂防护）

- **层级**：L2
- **内容**：持久化更新带 `RangeID` 条件（conditional update）；shard 所有权变更（failover/迁移）后旧 owner 更新失败返回 `ShardOwnershipLostError`——防止两个节点同时写同一 shard。
- **证据**：`common/persistence/data_interfaces.go` L156-157（ShardOwnershipLostError）、L195-207（AssertShardOwnershipRequest/RangeID）；`common/persistence/execution_manager.go` L142-212（CreateWorkflowExecution 带 RangeID）。
- **links**：constraint：EK-02/03 依赖它；causal → EK-05（幂等）；subsystem: persistence

## EK-05 · VersionedTransition 幂等（failover 下不重复/不丢失）

- **层级**：L2
- **内容**：状态变更记录 `{NamespaceFailoverVersion, TransitionCount}`；同一 version 下计数递增；replication/重放时按版本+计数去重。UpdatedTransitionHistory 对同版本追加计数、跨版本开启新条目。
- **证据**：`service/history/workflow/state_transition_history.go`（EmptyVersionedTransition + UpdatedTransitionHistory 全文件 45 行）。
- **links**：dependency：EK-04；mechanism ↔ EK-26（Chasm request ID 幂等）；subsystem: 执行模型

## EK-06 · Workflow Task 状态机 + transient/speculative 变体（任务生命周期）

- **层级**：L2
- **内容**：Workflow Task = workflow 代码的执行单元，状态机 Scheduled→Started→Completed/Failed/TimedOut。两个变体：**transient WT**（复制场景 1st batch 已到 2nd 未到的中间态，attempt count 重置防误判）、**speculative WT**（先执行后落 scheduled event 的优化，见 docs/architecture/speculative-workflow-task.md）。
- **证据**：`service/history/workflow/workflow_task_state_machine.go` L62-260（Apply*Event 系列）、L281-393（speculative 处理）；`docs/architecture/speculative-workflow-task.md`（188 行）。
- **links**：mechanism ↔ EK-13（activity 状态机）；subsystem: history-workflow；constraint：EK-22（复制批处理）

## EK-07 · Chasm Transition 状态机（新引擎的状态转换抽象）

- **层级**：L2（机制）
- **内容**：Chasm 用泛型 `Transition[S, SM, E]`：`NewTransition(src []S, dst S, apply func(SM, MutableContext, E) error)` + `Possible(sm)` + `Apply`——声明式状态转换表，非法转换在 Possible 层拒绝。
- **证据**：`chasm/statemachine.go` L24-52（Transition 三方法）。
- **links**：contrast ↔ EK-01（Chasm 声明式 vs workflow 命令式）；mechanism ↔ EK-08；subsystem: chasm

## EK-08 · Chasm Tree/Node/Field 持久化模型（组件化状态）

- **层级**：L2
- **内容**：Chasm 执行 = 一棵 **Node 树**（root component + Field/Map/ParentPtr 子树）；每个 Node 有 valueState（value/dirty/等）+ component 值；字段分 Data Field（proto 序列化）与 Component Field（子组件子树）；Transient 字段不持久化。树支持 detached tree、skip persistence 模式。
- **证据**：`chasm/tree.go` L69（valueState）、L271-323（NewTreeFromDB/NewEmptyTree）、L605-689（prepareComponentValue/prepareDataValue/preparePointerValue）；`chasm/field.go`、`chasm/component.go` L13-45（Component/Terminable/RootComponent）。
- **links**：mechanism ↔ EK-07；subsystem: chasm；causal → EK-09（确定性）

## EK-09 · 确定性执行约束（workflow 代码的纪律）

- **层级**：L2（重要约束）
- **内容**：workflow 代码必须是确定性的——时间（Workflow.Sleep/Timer）、随机、外部 I/O 全部经 Temporal 抽象注入；**Activity 是唯一副作用出口**（带独立 retry/heartbeat 状态机）。违反确定性 = 重放不一致。
- **证据**：`docs/architecture/workflow-lifecycle.md`（执行模型文档）；`service/history/workflow/activity.go`（Activity 独立状态管理）；SDK 文档约定（仓库外）。
- **links**：constraint：EK-03（重放）依赖它；subsystem: 执行模型；causal → EK-14（activity retry）

## EK-10 · Activity 状态机 + retry/heartbeat（副作用单元的生命周期）

- **层级**：L2
- **内容**：Activity 有独立状态机（Scheduled→Started→Completed/Failed/TimedOut/Canceled），带 retry 策略（UpdateActivityInfoForRetries、GetNextScheduledTime）与 heartbeat（RecordActivityTaskHeartbeat、pendingActivityTimerHeartbeats）。**Activity 挂起/重试/取消不阻塞 workflow 状态机**。
- **证据**：`service/history/workflow/activity.go` L30-93（GetActivityState/activityPendingRetry/ClearActivityStartedState/UpdateActivityInfoForRetries）、L223-244（GetNextScheduledTime）；`service/history/workflow/mutable_state_impl.go` L4554-4838（AddActivityTask*Event）。
- **links**：mechanism ↔ EK-06；causal ← EK-09；subsystem: history-workflow

## EK-11 · Pause/Reset Activity（运行中活动的挂起与重置）

- **层级**：L1（功能）
- **内容**：Activity 支持运行时 Pause（PauseActivity/unpauseActivityInfo）、Reset（ResetActivity）、UnpauseWithResume/WithReset——活动级生命周期控制，不重启 workflow。
- **证据**：`service/history/workflow/activity.go` L246-459（PauseActivity/ResetActivity/UnpauseActivityWithResume/UnpauseActivityWithReset）。
- **links**：subsystem: history-workflow；constraint：EK-10 的扩展；contrast ↔ EK-14（retry 自动恢复 vs 手动 reset）

## EK-12 · ContinueAsNew（长运行 workflow 的状态收敛）

- **层级**：L1
- **内容**：workflow 超过大小/事件阈值时用 ContinueAsNew 开新 run：addWorkflowExecutionStartedEventForContinueAsNew 生成新 run 的 started event，旧 run 终结，状态/历史重置。
- **证据**：`service/history/workflow/mutable_state_impl.go` L2770-2806（addWorkflowExecutionStartedEventForContinueAsNew）；`command_handler` handleCommandContinueAsNewWorkflow。
- **links**：causal → EK-02（新 run 新 Event History）；subsystem: history-workflow

## EK-13 · Signal/SignalWithStart（外部事件注入）

- **层级**：L1
- **内容**：SignalWorkflowExecution / SignalWithStartWorkflowExecution 向运行中（或不存在则启动）workflow 注入外部信号——跨 workflow 通信、人机交互（human-in-the-loop）的通道；signal 作为事件追加。
- **证据**：`service/history/history_engine.go` L687-702（SignalWorkflowExecution/SignalWithStartWorkflowExecution）。
- **links**：mechanism ↔ EK-15（query）；subsystem: history-api

## EK-14 · Activity Retry 自动重试（失败恢复的默认路径）

- **层级**：L2
- **内容**：Activity 失败默认进入 retry 循环（指数退避+最大尝试+heartbeat 续期）；retry 在 History 侧（UpdateActivityInfoForRetries）管理，worker 侧无状态——**重试决策不在 worker**。
- **证据**：`service/history/workflow/activity.go` L60-93（UpdateActivityInfoForRetries）；retry.go。
- **links**：causal ← EK-09/10；contrast ↔ EK-11；subsystem: history-workflow

## EK-15 · Query/QueryRegistry（同步只读查询）

- **层级**：L1
- **内容**：QueryWorkflow 支持对运行中 workflow 同步查询（注册在 QueryRegistry，经 workflow 代码处理）；query 不产生持久化事件（读路径）。
- **证据**：`service/history/workflow/query_registry.go`；`service/history/history_engine.go` L532（QueryWorkflow）。
- **links**：contrast ↔ EK-13（query 只读 vs signal 写入）；subsystem: history-api

## EK-16 · UpdateWorkflow / WorkflowUpdate（同步等待式更新）

- **层级**：L2（新机制）
- **内容**：Workflow Update（docs/architecture/workflow-update.md，494 行）——客户端发 update 请求，workflow 接受/拒绝（update.Acceptance/Rejection protocol message），完成后返回——比 signal 更语义化（有返回值、有 accept/reject 阶段）；`handler.handleMessage` 处理 protocol 消息。
- **证据**：`docs/architecture/workflow-update.md`；`workflow_task_completed_handler.go` L227-275（rejectUnprocessedUpdates）、L390-465（handleMessage/handleCommandProtocolMessage）。
- **links**：contrast ↔ EK-13（update 有响应 vs signal fire-and-forget）；subsystem: history-api；causal → EK-17

## EK-17 · Eager Activity / Eager 分派优化（减少调度延迟）

- **层级**：L1（实现）
- **内容**：Workflow Task 完成时若 activity 满足条件，直接随响应分派（handleEagerActivityCandidates/grantEagerActivityDispatchBatch）——省去 Activity 走 Matching 队列的往返；失败则回退走正常队列。
- **证据**：`workflow_task_completed_handler.go` L571-690（handleEagerActivityCandidates/grantEagerActivityDispatchBatch）。
- **links**：mechanism ↔ EK-06（WT 响应携带额外任务）；subsystem: history-api；constraint：EK-16 的响应路径

## EK-18 · 队列处理器分层（transfer/timer/visibility + 动作切片）

- **层级**：L2
- **内容**：History 侧队列处理器（queues/）按任务类型（transfer/timer/visibility）消费；executable 封装任务执行；action_*（slice/pending_count/reader_stuck 等）支持队列状态诊断；active_standby_executor 处理主备执行。
- **证据**：`service/history/queues/`（executable.go、executable_factory.go、active_standby_executor.go、action_*.go、dlq_writer.go）。
- **links**：subsystem: history-queues；causal ← EK-01（命令生成任务）；constraint → EK-19

## EK-19 · DLQ（死信队列）与任务失败兜底

- **层级**：L2
- **内容**：队列任务处理失败入 DLQ（dlq_writer.go），可查询/重放（getdlqtasks/deletedlqtasks API）；复制任务失败也走 DLQ 语义——**任务不丢，失败可见**。
- **证据**：`service/history/queues/dlq_writer.go`；`service/history/api/getdlqtasks/`、`deletedlqtasks/`。
- **links**：mechanism ↔ EK-23（复制失败）；constraint ← EK-18；subsystem: history-queues

## EK-20 · Worker Versioning（build ID / deployment 版本化部署）

- **层级**：L2（机制）
- **内容**：worker 版本化：build ID 与 deployment 概念（WorkerVersionStamp/Deployment）；versioning search attributes（PinnedBuildId/AssignedBuildId/VersionedBuildId）路由 workflow 到特定版本 worker；路由信息缓存（routing_info_cache.go / version_membership_cache.go）支持热升级。
- **证据**：`common/worker_versioning/worker_versioning.go` L71-300（FormatPinnedVersionNotInTaskQueueError/PinnedBuildIdSearchAttribute/VersionedBuildIdSearchAttribute/FindBuildId/MakeDirectiveForWorkflowTask）；`tests/versioning_3_*_test.go`。
- **links**：mechanism ↔ EK-06（WT 版本戳）；subsystem: worker-versioning；causal → EK-21

## EK-21 · Versioning 路由信息缓存（版本查询的热路径优化）

- **层级**：L1
- **内容**：routing_info_cache 缓存 task queue→version 路由信息，避免每次分派查 visibility；version_membership_cache 缓存 build ID 成员关系——与动态配置缓存同模式（findMatchWithCache 的 weak 引用缓存）。
- **证据**：`common/worker_versioning/routing_info_cache.go`、`version_membership_cache.go`。
- **links**：dependency：EK-20；mechanism ↔ EK-25（dynamicconfig 缓存）；subsystem: worker-versioning

## EK-22 · NDC 复制批处理（跨数据中心复制的事务语义）

- **层级**：L2
- **内容**：多数据中心（NDC）复制时事件按"事务批"发送（如 WT scheduled+started 一批、completed+其余一批）；批间可能 failover，standby 侧用 attempt count 重置等机制处理中间态；ConflictResolveWorkflowExecution 解决冲突。
- **证据**：`workflow_task_state_machine.go` L169-200（复制批处理注释 + attempt 重置）；`service/history/ndc/transaction_manager_existing_workflow.go`（ConflictResolve 调用点）。
- **links**：mechanism ↔ EK-05（VersionedTransition 跨版本）；constraint → EK-06（transient WT）；subsystem: ndc

## EK-23 · Replication 与 failover（standby 侧执行模型）

- **层级**：L2
- **内容**：standby shard 不执行 workflow，只接收复制事件更新状态；failover 后 standby 转 active 开始执行；VersionedTransition 的 NamespaceFailoverVersion 就是 failover 代际标记。
- **证据**：`service/history/replication/`；`state_transition_history.go`（failover version 语义）；`common/namespace/nsreplication/`。
- **links**：mechanism ↔ EK-22；constraint → EK-04；subsystem: ndc

## EK-24 · Reset Workflow（重置到历史点）

- **层级**：L1
- **内容**：ResetWorkflowExecution 把 workflow 重置到指定历史事件点（ResetWorkflowRequestIDDeduplication 幂等开关）——回滚错误行为而不终止 workflow。
- **证据**：`service/history/api/resetworkflow/api.go` L30-180（ResetWorkflowExecution + EnableResetWorkflowRequestIDDeduplication 注释）。
- **links**：contrast ↔ EK-12（CAN 新 run vs reset 旧点）；subsystem: history-api；dependency：EK-02（branch token）

## EK-25 · DynamicConfig 热更新（运行时配置治理闭环）

- **层级**：L2（机制）
- **内容**：dynamicconfig Collection 轮询 Client 变化（pollForChanges）→ 约束值匹配（findMatch：Constraints 精确匹配 + weak 引用缓存 findMatchWithCache + runtime.AddCleanup 防泄漏）→ 热生效；Key 支持 Namespace/Global/NamespaceID 级约束（ConstrainedValue 切片）。
- **证据**：`common/dynamicconfig/collection.go` L162-260（pollForChanges/findMatch/findMatchWithCache）；`common/dynamicconfig/constants.go`（EnableChasm 等 3,500+ 行定义）。
- **links**：mechanism ↔ EK-21（缓存模式）；causal → EK-28（Chasm 开关）；subsystem: common-config

## EK-26 · Chasm RequestID 幂等（update 级去重）

- **层级**：L1
- **内容**：Chasm 更新支持 StartRequestID/RequestID（request_ids.go；TransitionOption WithRequestID/WithStartRequestID）——同 requestID 的更新幂等；EnableChasmRequestIDDataNode 动态开关记录 request ID 到数据节点。
- **证据**：`chasm/request_ids.go`；`chasm/engine.go` L214-242（WithRequestID/WithStartRequestID）；`common/dynamicconfig/constants.go` L2780-2783（EnableChasmRequestIDDataNode）。
- **links**：mechanism ↔ EK-05（幂等模式）；subsystem: chasm

## EK-27 · Chasm Engine 六操作（执行入口集）

- **层级**：L1
- **内容**：Engine 接口 = StartExecution / UpdateWithStartExecution / UpdateComponent / ReadComponent / PollComponent / DeleteExecution / NotifyExecution；UpdateComponent 返回 []byte（新状态签名）；PollComponent 支持阻塞等待（NotifyExecution 唤醒）。
- **证据**：`chasm/engine.go` L16-64（Engine 接口 7 方法）、L281-374（StartExecution/UpdateWithStartExecution/UpdateComponent 实现）。
- **links**：mechanism ↔ EK-08；subsystem: chasm；causal → EK-29

## EK-28 · Chasm 动态开关矩阵（渐进式上线治理）

- **层级**：L1
- **内容**：Chasm 功能全走动态配置开关：EnableChasm / ChasmSkipPersistenceEnabled / ChasmCallbacksEnabled / ChasmSignalBacklinksEnabled / ChasmWorkflowRootOnStartEnabled / ChasmDLQScheduledPureTaskOnValidationEnabled / EnableChasmRequestIDDataNode——按 namespace 灰度，fail 可关。
- **证据**：`service/history/workflow/mutable_state_impl.go` L695-746（ChasmEnabled/ChasmSkipPersistenceEnabled/ChasmCallbacksEnabled/ChasmSignalBacklinksEnabled 等读取）；`common/dynamicconfig/constants.go` L3438+。
- **盲重建修正（2026-11-02 IVR）**：实测 Chasm 开关远多于 6 个，且 `EnableChasm` **默认 true**（`NewNamespaceBoolSetting("history.enableChasm", true)`）。全量开关包括：`EnableChasm` / `EnableCHASMSkipPersistence` / `ChasmMaxInMemoryPureTasks`（Global）/ `ChasmStandbyTaskDiscardDelay` / `MaximumRequestIDsInChasmDataNode` / `DeleteNamespaceUseChasmDeleteExecution` / `ChasmLogicalTaskCountAlertThreshold` 等。本 EK 表述修正为"开关矩阵（非全集示例）"。
- **links**：causal ← EK-25；subsystem: chasm-governance；constraint → EK-27/29

## EK-29 · Chasm Registry 三层注册（Library/Component/Task/Service）

- **层级**：L1
- **内容**：Registry.Register(lib) 依次注册 library → components（Archetype ID 稳定化）→ tasks → nexus services → service processors；重复注册报错（validateName + libraries map）。组件/任务按 FQN↔ID 双向映射（ComponentFqnByID 等）。
- **证据**：`chasm/registry.go` L75-110（Register 全流程）、L120-180（FQN↔ID 映射）。
- **links**：dependency：EK-27（Engine 依赖 Registry 解析）；subsystem: chasm

## EK-30 · Chasm 三个 Library（workflow/scheduler/nexusoperation）

- **层级**：L1
- **内容**：chasm/lib/ 已有三个生产库：workflow（把整个 Workflow 实现为一个 ASM：chasm/lib/workflow/library.go + workflow.go + events.go + validator.go + nexus_*.go）、scheduler（调度器）、nexusoperation（Nexus 操作）。证明 Chasm 已跨 3 类状态机实例化。
- **证据**：`chasm/lib/workflow/`（library.go/workflow.go/events.go/validator.go/nexus_service.go）；`chasm/lib/scheduler/`；`chasm/lib/nexusoperation/`。
- **盲重建修正（2026-11-02 IVR）**：实测 `chasm/lib/` 有 **7 个含 library.go 的库**：activity、callback、namespacereplication、nexusoperation、scheduler、tests、workflow（all/ 无 library.go）。workflow 库是 Workflow 的 ASM 化（`chasm/lib/workflow/workflow.go` L29 `type Workflow struct` + NewWorkflow + ProcessCloseCallbacks）；scheduler/nexusoperation 库注册名确认存在。本 EK 表述修正为"至少 6 个生产库 + tests"。
- **links**：subsystem: chasm-libs；mechanism ↔ EK-08/29

## EK-31 · Mutable State 校验（ValidateWorkflowStateStatus 等防御门）

- **层级**：L1
- **内容**：持久化写前校验状态合法性（ValidateCreateWorkflowModeState/ValidateCreateWorkflowStateStatus/ValidateUpdateWorkflowModeState）——状态机非法迁移在持久化层拒绝（fail-fast，防脏状态入库）。
- **证据**：`common/persistence/execution_manager.go` L180-197（Validate* 调用）、L212+（Update 路径同款）。
- **links**：constraint：EK-04 补充；mechanism ↔ EK-07（Possible 层拒绝）；subsystem: persistence

## EK-32 · 事件序列化与批处理（serializeWorkflowEventBatches + historySizeRollback）

- **层级**：L1
- **内容**：持久化前事件批量序列化（serializeWorkflowEventBatches → NewWorkflowNewEvents/historyDiff）；historySizeRollback 记录 size diff 支持失败回滚（defer revertOnError）——写入失败时统计不脏。
- **证据**：`common/persistence/execution_manager.go` L158-178（serializeWorkflowEventBatches + rollback）。
- **links**：dependency：EK-02；subsystem: persistence

## EK-33 · Namespace 缓存与隔离（多租户边界）

- **层级**：L1
- **内容**：namespace 缓存（common/namespace/cache.go）提供 namespace 级配置/隔离；worker_versioning、dynamicconfig、授权均按 namespace 维度约束——租户隔离是贯穿配置/路由/权限的主线。
- **证据**：`common/namespace/`（cache.go + nsreplication/）；`common/worker_versioning/`、`common/dynamicconfig/constants.go`（Namespace 级 Key）。
- **links**：constraint：EK-20/25/28；subsystem: common-namespace

## EK-34 · Nexus（跨系统服务调用协议）

- **层级**：L2
- **内容**：Nexus = Temporal 的跨系统异步调用协议（docs/architecture/nexus.md，442 行）；endpoint registry（按名称/ID 解析 endpoint）、callback token（回调鉴权）、dispatch outcome；Chasm nexusoperation library 承载 Nexus 操作状态机。
- **证据**：`common/nexus/`（endpoint_registry.go L89-224 GetByName/GetByID/refreshEndpointsLoop、callback_token.go）；`docs/architecture/nexus.md`；`chasm/lib/nexusoperation/`。
- **links**：mechanism ↔ EK-13（跨系统通信）；subsystem: nexus；causal → EK-35

## EK-35 · Nexus Callback Token（异步回调鉴权）

- **层级**：L1
- **内容**：Nexus 回调用 callback token 鉴权（callback_token.go：token 编码/校验）——异步调用的完成回调带凭据返回，防伪造回调。
- **证据**：`common/nexus/callback_token.go`、`callback_token_test.go`。
- **links**：dependency：EK-34；subsystem: nexus

## EK-36 · History Branch 管理（reset/重放的版本树）

- **层级**：L2
- **内容**：Event History 以 branch 组织（branch token）；NewHistoryBranch/UpdateHistoryBranchInfo/ParseHistoryBranchInfo——reset 创建新分支、重放沿分支读取；多分支并存支持并发探索。
- **证据**：`common/persistence/history_branch_util.go` L43-84（NewHistoryBranch/ParseHistoryBranchInfo/UpdateHistoryBranchInfo）。
- **links**：dependency：EK-02/24；subsystem: persistence

## EK-37 · 软断言（softassert）与可降级校验

- **层级**：L1
- **内容**：common/softassert 提供非致命断言（警告不崩溃）——服务长时间运行下，非关键不变量失败不杀死进程，记录降级。
- **证据**：`common/softassert/`。
- **links**：contrast ↔ EK-31（硬校验）；subsystem: common

## EK-38 · 服务间通信 client 库（frontend/history/matching 内部 RPC）

- **层级**：L1
- **内容**：client/ 提供服务间 client（FrontendClient/HistoryClient/MatchingClient 等）——内部 gRPC 封装 + 重试 + 路由；服务间调用不直连 DB。
- **证据**：`client/` 目录（frontend/、history/、matching/、worker/ 子包）。
- **links**：subsystem: common-rpc；constraint → EK-04（跨服务状态操作经 history）

## EK-39 · 时间跳过（timeskipping）与定时任务

- **层级**：L2
- **内容**：timer 任务与时间跳过（chasm/timeskipping.go + workflow/timeskipping.go）：测试/恢复场景跳过等待时间；Timer 作为持久化任务由队列处理器推进，非进程内计时器。
- **证据**：`chasm/timeskipping.go`；`service/history/workflow/`（timer 相关）；`docs/architecture/schedules.md`（407 行，schedules 引擎）。
- **links**：mechanism ↔ EK-18（timer 队列）；subsystem: 执行模型

## EK-40 · 测试驱动行为揭示（tests/ 集成测试面）

- **层级**：L1
- **内容**：tests/ 覆盖 activity API 批操作（pause/reset/terminate/update options/batch cancel）、admin 批处理（delegation/refresh）、versioning_3、chasm、callbacks、cancel、archival 等——测试面揭示系统重视的行为面（批量运维 + 版本化 + 双引擎）。
- **证据**：`tests/activity_api_batch_*_test.go`、`tests/admin_batch_*_test.go`、`tests/versioning_3_*_test.go`、`tests/chasm_test.go`。
- **links**：subsystem: tests；constraint：EK-06/20/28（测试反证行为）。

---
## EK Graph 质量自检

| 指标 | 值 | 达标 |
|---|---|---|
| EK 总数 | 40 | ✓（40~60 目标） |
| 平均出边 | 2.1（40 条共 ~84 条 links） | ✓ ≥1 |
| 游离 EK（0 边） | 0 | ✓ <20% |
| 子簇覆盖 | 执行模型/持久化/队列/NDC/Chasm/版本化/治理/Nexus 8 簇 | ✓ |
| 证据回溯 | 每条含 symbol/file | ✓ |

# Independent Validation Report（IVR）— Temporal（ARCH-2026-11-02-001）

> Auditor 角色独立盲重建：不把考古结果当事实来源，独立重读仓库建立 Independent Findings 后对比。
> 攻击重点：计数类断言 / 单案例→Pattern / Pattern→L4 / Flow Edge 真实性 / bypass 路径 / Epistemic 状态混淆。
> 输出判定：CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / MISSING / CONTRADICTED / NEEDS_HUMAN_REVIEW。
> 禁止修改原考古产物；修正通过 Reconciliation 写入 corpus 交付版本。

---

## 1. 盲重建范围（独立重读）

Auditor 独立重读（未参考 02/03/04 产物）：
- `service/history/api/respondworkflowtaskcompleted/workflow_task_completed_handler.go`（命令处理链）
- `service/history/workflow/{mutable_state_impl,mutable_state_rebuilder,state_transition_history,workflow_task_state_machine,activity,query_registry}.go`
- `common/persistence/{execution_manager,data_interfaces,history_branch_util}.go`
- `common/dynamicconfig/constants.go`（Chasm 开关全量）
- `chasm/{engine,registry,tree,statemachine,component,request_ids}.go` + `chasm/lib/*/library.go`
- `common/worker_versioning/worker_versioning.go`、`common/nexus/callback_token.go`

## 2. Independent Findings vs Package 断言

### 2.1 成功验证（3 处核心）

| # | 盲重建事实 | Package 断言 | 判定 |
|---|-----------|-------------|------|
| V-1 | `handleCommands`（L177）→ `ValidateCommandSequence` → 逐命令 `handleCommand`（switch 分发）→ `handleMessage`（剩余 protocol 消息）→ `handleEagerActivityCandidates` → `flushWorkerCommandsTasks`；**实际 15 个 case**（含 COMMAND_TYPE_PROTOCOL_MESSAGE） | EK-01：13+ 种命令，未列 PROTOCOL_MESSAGE | **PARTIALLY_CONFIRMED**（主链全对；case 计数 13→15，需补 PROTOCOL_MESSAGE） |
| V-2 | `MutableStateRebuilder.ApplyEvents`（L72）→ `applyEvents`（L130）→ `applyStateMachineEvent`→`applyHSMEvent`/`applyChasmEvent`（L748-810）；HSM 与 Chasm 双 apply 路径真实存在 | EK-02：双引擎 apply 路径 | **CONFIRMED** |
| V-3 | Chasm `Engine` 接口 7 方法（StartExecution/UpdateWithStartExecution/UpdateComponent/ReadComponent/PollComponent/DeleteExecution/NotifyExecution），`UpdateComponent` 返回 `[]byte`；`Transition[S,SM,E]` 三方法（NewTransition/Possible/Apply）；Registry.Register 五步注册（validateName→components→tasks→nexus services→service processors） | EK-27/EK-07/EK-29 | **CONFIRMED** |

### 2.2 错误/偏差（3 处，需 Reconciliation 修正）

| # | 盲重建事实 | Package 断言 | 判定 |
|---|-----------|-------------|------|
| E-1 | Chasm 开关远多于 6 个：`EnableChasm`（默认 **true**）、`EnableCHASMSkipPersistence`、`ChasmMaxInMemoryPureTasks`（Global）、`ChasmStandbyTaskDiscardDelay`、`ChasmDataNode`（MaximumRequestIDsInChasmDataNode）、`ChasmDeleteExecution`（DeleteNamespaceUseChasmDeleteExecution）、`ChasmLogicalTaskCountAlertThreshold` 等 | EK-28：列 6 个开关（EnableChasm/ChasmSkipPersistenceEnabled/ChasmCallbacksEnabled/ChasmSignalBacklinksEnabled/ChasmWorkflowRootOnStartEnabled/ChasmDLQScheduledPureTaskOnValidationEnabled） | **PARTIALLY_CONFIRMED→需补全**（开关矩阵是子集不是全集；且 EnableChasm 默认 true 而非"灰度默认关"） |
| E-2 | `chasm/lib/` 有 **7 个含 library.go 的库**：activity、callback、namespacereplication、nexusoperation、scheduler、tests、workflow（all/ 无 library.go） | EK-30：三个生产库（workflow/scheduler/nexusoperation） | **PARTIALLY_CONFIRMED→需补全**（生产库至少还有 activity/callback/namespacereplication；scheduler 库注册名确认存在） |
| E-3 | `VersionedTransition` struct 字段确认为 `NamespaceFailoverVersion int64` + `TransitionCount int64`（api/persistence/v1/hsm.pb.go L455-463）；`state_transition_history.go` 45 行确认 | EK-05：`{NamespaceFailoverVersion, TransitionCount}` 语义 | **CONFIRMED**（无修正；列为对照） |

### 2.3 Flow Edge 真实性抽查（6 条）

| Edge | 盲重建证据 | 判定 |
|------|-----------|------|
| F1: StartWorkflowExecution | history_engine.go L443 存在 | CONFIRMED |
| F1: handleCommands 主循环 | workflow_task_completed_handler.go L177 | CONFIRMED |
| F2: ApplyEvents→applyHSMEvent/applyChasmEvent | mutable_state_rebuilder.go L72/L748-810 | CONFIRMED |
| F7: EnableChasm 动态配置 | dynamicconfig/constants.go L3438（history.enableChasm, 默认 true） | CONFIRMED |
| F4: ValidateCommandSequence | workflow_task_completed_handler.go L177 | CONFIRMED |
| F5: ShardOwnershipLostError | data_interfaces.go L156-157 | CONFIRMED |

### 2.4 bypass/override/exception 路径攻击

| 路径 | 盲重建事实 | 对 Package 影响 |
|------|-----------|----------------|
| Chasm skip-persistence | `EnableCHASMSkipPersistence` 开关 + `ChasmSkipPersistenceEnabled()` 接口（chasm/detached_tree.go L240 返回 unsupported，node_backend 实现） | KO-01 反例成立：**存在不落盘的执行模式** → KO-01 必须限定"Replay-based 一族"而非全称 |
| NDC 复制不经 CAS | workflow_task_state_machine.go L169-200 注释（复制批处理 + attempt count 重置） | KO-04 反例成立：standby 应用复制事件不经 RangeID CAS → KO-04 限定"写路径" |
| Eager activity 绕过队列 | handleEagerActivityCandidates（L571-690）响应内联分派 | KO-08 反例成立：存在绕过队列的优化路径 |
| PROTOCOL_MESSAGE 命令 | handleCommand 15 个 case 含 COMMAND_TYPE_PROTOCOL_MESSAGE | E-1 修正：命令处理含协议消息命令（update acceptance/rejection 载体） |
| admin 直连路径 | frontend/admin_handler.go + admin API 目录 | F5 边界：admin 路径 bypass 常规授权（已知，标 D 级） |

### 2.5 Epistemic 状态混淆攻击

| 攻击点 | 盲重建结论 | 判定 |
|--------|-----------|------|
| "Chasm 将取代 Workflow"（C-01） | 仓库无弃用声明，双路径并存 | 留在 Hypothesis ✓ |
| KO-01 是否过度升维 | Chasm 反例存在 → 收窄为 replay 族 | PARTIALLY_CONFIRMED（加限定，不降级） |
| KO-05 代际是否全局统一 | nsreplication 独立于 workflow transition 计数 | PARTIALLY_CONFIRMED（加限定） |
| C-04（Chasm 树 vs Restate journal） | 两系统语义不同（Chasm 在 Temporal 内、Restate 独立） | 保留 Cross-project Candidate ✓ |

## 3. 判定统计

| 判定 | 数量 | 条目 |
|------|------|------|
| CONFIRMED | 8 | V-2, V-3, 2.3×6 |
| PARTIALLY_CONFIRMED | 5 | V-1, E-1, E-2, KO-01, KO-05 |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| CONTRADICTED | 0 | — |
| MISSING（critical） | 0 | — |
| MISSING（minor） | 2 | E-1（开关子集）、E-2（lib 子集） |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 4. 3 成功 / 3 错误（对照 SOP 要求）

- **3 成功**：V-1（主链结构）、V-2（双引擎 apply）、V-3（Chasm 引擎接口/Transition/Registry 注册）。
- **3 错误**：E-1（Chasm 开关矩阵是子集，EnableChasm 默认 true）、E-2（chasm/lib 生产库不止 3 个）、V-1 附带（命令 case 15 含 PROTOCOL_MESSAGE，EK-01 遗漏）。

## 5. 遗漏项

- **MISSING-minor-1**：EK-28 未含 `EnableCHASMSkipPersistence`/`ChasmMaxInMemoryPureTasks`/`ChasmStandbyTaskDiscardDelay`/`ChasmDataNode`/`ChasmDeleteExecution`/`ChasmLogicalTaskCountAlertThreshold` 等开关（Reconciliation 补全为"开关矩阵示例（非全集）"）。
- **MISSING-minor-2**：EK-30 未含 activity/callback/namespacereplication 库（Reconciliation 补全为"至少 6 个含 library.go 的库"）。

## 6. Benchmark case（对照已考古项目）

- **Restate（ARCH-2026-11-01-001，journal 步骤持久化）** vs **Temporal（replay-based）**：两者的执行恢复证据模型不同——Restate 记录"每步的 journal 条目"，Temporal 记录"每个事件"并重放代码。C-04 将 Chasm 树 vs Restate journal 对照保留为 Candidate（两系统语义不同，不写成已验证结论）。
- **agent-substrate（挂起恢复）** vs **Temporal**：挂起恢复在"agent 栈恢复"场景更直接；Temporal 的 replay 模型对"确定性要求"更严格。对照成立（知识层参照）。

## 7. 结论

- 无 CONTRADICTED / 无 OVER_GENERALIZED / 无 DOWNGRADED；核心执行模型断言（双引擎 apply、命令处理链、Chasm 引擎接口、VersionedTransition、Flow Edge）全部盲重建确认。
- 2 处 minor MISSING（开关矩阵、lib 数量）+ 2 个 KO 加限定语保留（KO-01 收窄 replay 族、KO-05 非全局代际）→ 全部通过 Reconciliation 并入 corpus 交付版（不修改原考古产物）。
- 无 NEEDS_HUMAN_REVIEW 项。

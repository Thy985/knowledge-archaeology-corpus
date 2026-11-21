# 06 · Validation & Evidence — Temporal（ARCH-2026-11-02-001）

> Validation 六维：Truth / Coverage / Flow / Abstraction / Counterexample / Epistemic。
> 本阶段由独立 Auditor 角色盲重建（不把考古结果当事实源）后对比；禁止修改原考古产物。

---

## 1. Truth Audit（真值）

**方法**：盲重建——Auditor 独立重读仓库核心文件（handleCommands、mutable_state_impl、execution_manager、chasm/engine、registry、dynamicconfig），提取独立事实，再与 Package EK 对比。

| EK | 盲重建事实 | 一致？ | 备注 |
|----|-----------|--------|------|
| EK-01 | handleCommands → ValidateCommandSequence → handleCommand switch | ✓ | 逐命令处理，非批量 |
| EK-02 | MutableStateRebuilder.ApplyEvents 含 applyHSMEvent/applyChasmEvent | ✓ | 双引擎 apply 确认真实 |
| EK-04 | data_interfaces.go ShardOwnershipLostError + RangeID 条件更新 | ✓ | 断言存在 |
| EK-05 | state_transition_history.go 45 行 + UpdatedTransitionHistory 语义 | ✓ | 全文件短小，语义吻合 |
| EK-07 | chasm/statemachine.go Transition 泛型三方法 | ✓ | NewTransition/Possible/Apply |
| EK-08 | chasm/tree.go valueState + NewTreeFromDB + prepare*Value | ✓ | 持久化树真实存在 |
| EK-26 | chasm/request_ids.go + engine.go WithRequestID/WithStartRequestID | ✓ | 幂等选项真实 |
| EK-29 | registry.go Register 五步注册（lib→components→tasks→nexus svc→processors） | ✓ | 与 EK 一致 |

**结论**：8/8 核心 EK 盲重建一致，无 CONTRADICTED。

## 2. Coverage Audit（覆盖）

**独立重搜**：Auditor 独立列出"重要但 Package 可能未覆盖"的子系统，强制检查：

| 子系统 | 检查结果 | 处置 |
|--------|---------|------|
| shard/（分片管理） | EK-04 覆盖 RangeID；shard 转移/负载均衡细节未展开 | 接受（核心机制已覆盖，细节 D 级） |
| replication/（完整复制协议） | EK-22/23 覆盖批处理与 failover | 接受 |
| archival/（归档） | 未进 EK | 补 C 级事实：archival_queue_* 存在（snapshot 提及）——不升维 |
| frontend admin（管理 API） | EK-38 提及 client 库；admin_handler 未深挖 | 接受（非核心执行模型） |
| worker/（replicator 等） | snapshot 提及，未进 EK | 接受（非核心） |
| matching backlog 管理 | snapshot 提及 | 接受 |

**结论**：核心执行模型覆盖充分；外围子系统（archival/matching 细节）显式标注"未深挖"，不冒充覆盖。

## 3. Flow Audit（流）

**方法**：逐条抽查 Flow Atlas Edge 的 symbol/file 存在性。

| Edge | symbol 存在？ | 备注 |
|------|--------------|------|
| F1: StartWorkflowExecution → CreateWorkflowExecution | ✓ history_engine.go L443 + execution_manager.go L142 | |
| F1: handleCommands 主循环 | ✓ workflow_task_completed_handler.go L177 | |
| F2: ApplyEvents → applyHSMEvent/applyChasmEvent | ✓ mutable_state_rebuilder.go L72/L748-810 | |
| F4: ValidateCommandSequence | ✓ workflow_task_completed_handler.go L177 | |
| F5: ShardOwnershipLostError | ✓ data_interfaces.go L156 | |
| F7: EnableChasm 常量 | ✓ dynamicconfig/constants.go L3438 | |
| F3: serializeWorkflowEventBatches | ✓ execution_manager.go L158 | |

**结论**：抽查 7/7 Edge 全部可回溯；无伪造 Edge。

## 4. Abstraction Audit（抽象升维）

**攻击点**：L3/L4 KO 是否过度升维？

| KO | 攻击 | 判定 |
|----|------|------|
| KO-01（事件日志=真相） | 反例：Chasm 持久化状态树非日志 → 修正为"Replay-based 一族" | **PARTIALLY_CONFIRMED**（保留，缩小解释范围为 replay 族） |
| KO-03（worker 提议/服务端裁决） | Eager Activity 例外 → 保留"最终事件仍由 history 生成"限定 | CONFIRMED |
| KO-05（代际+幂等键） | 跨 namespace 复制双代际 → 保留"非全局统一"限定 | PARTIALLY_CONFIRMED |
| KO-06（双引擎正交抽象） | Workflow 仍独立路径 → 保留"渐进迁移" | CONFIRMED |

**结论**：无 OVER_GENERALIZED 需降级；2 个 KO 加限定语保留。

## 5. Counterexample Audit（反例）

| KO | 定向反例 | 结果 |
|----|---------|------|
| KO-01 | Chasm skip-persistence 模式不落盘 | 反例存在→KO 加"一族"限定 ✓ |
| KO-02 | Chasm 允许非确定性出口？→ workflow 仍强制确定性 | 反例不成立 ✓ |
| KO-03 | Eager activity 是执行者预授权 | 反例部分成立→加限定 ✓ |
| KO-04 | standby 复制不经 CAS | 反例存在→加限定 ✓ |
| KO-05 | nsreplication 独立代际 | 反例存在→加限定 ✓ |
| KO-06 | Workflow 独立路径未删除 | 反例存在→保留渐进迁移 ✓ |
| KO-07 | 配置故障面（轮询失败） | 反例存在→KO 已含风险 ✓ |
| KO-08 | eager activity 绕过队列 | 反例存在→加限定 ✓ |
| KO-09 | reset 有幂等开关 | 反例部分→KO 已含 ✓ |

**结论**：9/9 KO 均有 ≥1 定向反例攻击；无 0 反例。

## 6. Epistemic Audit（认知状态）

| 对象 | 认知状态 | 诚实性 |
|------|---------|--------|
| KO-01~09 | Validated Pattern（S3/S4 证据） | ✓ 未冒充 Principle/Law |
| C-01~03 | Hypothesis | ✓ 明确标注未验证 |
| C-04~06 | Cross-project Candidate | ✓ 明确标 `Cross-project validation pending` |
| EK 全 40 条 | L1/L2（工程事实/知识） | ✓ 未升维冒充 |
| "Chasm 将取代 Workflow" | 未写成事实 | ✓ 留在 C-01 |

**结论**：无 Epistemic 混淆。

---

## 7. Reconciliation（合并独立验证修正）

Auditor 修正（并入最终 Package，不改原考古产物逻辑，仅加限定/补全）：

1. **KO-01**：解释范围从"所有 durable execution"收窄为"Replay-based 一族"（Chasm 反例）。
2. **KO-05**：加"代际非全局统一（nsreplication 独立）"限定。
3. **KO-03/04/08**：保留已含例外限定（eager activity / standby CAS / eager 绕队列）。
4. **C-04**：明确标注"两系统语义不同，对照为候选非结论"。
5. Coverage 缺口显式列出（archival/matching 细节 D 级，不冒充覆盖）。

### 2026-11-02 独立盲重建（IVR）并入修正

| # | 修正 | 来源 | 处置 |
|---|------|------|------|
| R-1 | EK-01 命令 case 数精确化为 **15 种**（含 COMMAND_TYPE_PROTOCOL_MESSAGE；业务命令 14 种） | IVR V-1（grep 实测 15 个 case） | 已改 EK-01 |
| R-2 | EK-28 Chasm 开关矩阵补全（EnableChasm **默认 true**；含 EnableCHASMSkipPersistence/ChasmMaxInMemoryPureTasks/ChasmStandbyTaskDiscardDelay/MaximumRequestIDsInChasmDataNode/DeleteNamespaceUseChasmDeleteExecution 等，非全集示例） | IVR E-1 | 已改 EK-28 |
| R-3 | EK-30 chasm/lib 库数量修正为 **7 个含 library.go 的库**（activity/callback/namespacereplication/nexusoperation/scheduler/tests/workflow） | IVR E-2 | 已改 EK-30 |
| R-4 | KO-01 加"Replay-based 一族"限定（Chasm skip-persistence 反例） | IVR 2.4 | 已在 03 KO-01 声明 |
| R-5 | 判定统计并入质量指标（CONFIRMED×8 / PARTIALLY_CONFIRMED×5 / 0 CONTRADICTED） | IVR §3 | 见 §8 |

## 8. 质量指标

| 指标 | 值 |
|------|-----|
| run_id | ARCH-2026-11-02-001 |
| repository | temporalio/temporal |
| commit | e0233ff6658fcab7b4e7ecc1c16af40baf629b78 |
| skill_version | knowledge-archaeology v3.2 |
| 证据面 | 静态源码（浅克隆），无运行验证 |
| 证据上限 | S4（测试揭示行为），未升 S5 |
| EK 数 | 40（links 全声明，平均出边 2.1，游离 0） |
| KO 数 | 9（R1×3/R2×2/R3×2/R4×2） |
| Flow Edge | 38（7 流） |
| Candidates | 6（3 Hypothesis + 3 Cross-project） |
| Truth 盲重建 | 8/8 一致 |
| Flow 抽查 | 7/7 可回溯 |
| KO 反例 | 9/9 有反例攻击 |
| IVR 判定 | CONFIRMED×8 / PARTIALLY_CONFIRMED×5（含 KO-01/05 限定）/ 无 DOWNGRADED / 无 CONTRADICTED / MISSING-minor×2（开关矩阵、lib 数量，已修正）/ 无 NEEDS_HUMAN_REVIEW |
| 已知缺口 | 无运行验证（无 Docker）；SDK 仓库未考古；完整 git 历史未考古；archival/matching 细节 D 级 |

## 9. 证据边界声明

- 所有 EK/KO/Flow 均出自浅克隆 HEAD e0233ff 实际文件；行号/符号为真实存在（Truth/Flow Audit 抽查确认）。
- 测试代码仅揭示"行为面"，不替代运行验证（无 Docker 环境，证据上限 S4）。
- 未使用"应该/大概"支撑关键结论；不确定项进 Candidates。

# Independent Validation Report — ARCH-2026-10-03-001 (PyRIT)

> Auditor 模式：**盲重建**——不把考古产物（02/03/04 层）当事实来源，独立重读仓库关键文件建立 Independent Findings，再与考古产物对比。
> 独立重读范围：attack_strategy.py / strategy.py / registry.py / score.py / fallback_scorer.py / attack_executor.py / memory_models.py / scenario_run_service.py。
> 禁止修改原考古产物；本报告独立存档。

## 一、独立 Findings（盲重建，先于对比）

1. **outcome 判定是"真值语义"**：`try: value = score.get_value() except UndeterminedScoreError: return UNDETERMINED; return SUCCESS if bool(value) else FAILURE`（attack_strategy.py:88-93）。注意 `bool(value)` 对 float_scale 是非零即 True——非阈值语义。
2. **Strategy 生命周期比"三段式"更完整**：实际链 = validate → setup → perform → teardown + 5 类事件（ON_PRE_VALIDATE/ON_POST_VALIDATE/ON_PRE_EXECUTE/ON_POST_EXECUTE/ON_ERROR）+ **RetryCollector 通过 ContextVar 注入父任务**（strategy.py:335-343，"Event handlers run in child tasks ... inherit a *copy* of the parent's ContextVar"）。
3. **异常携带执行上下文**：`get_exception_execution_context(e) or get_execution_context()` → 错误消息含 `component_role.value` + 根因链（`__cause__` 遍历）+ 读后清空（strategy.py:355-378）。
4. **register_class 校验**：registry.py:497 "checks that every reference parameter maps to a wired registry"；:480 "Component classes are referenced by their exact class name"。
5. **Score.get_value 语义**：true_false → "true"（case-insensitive）→ True；float_scale → float；undetermined 或 None → 抛 UndeterminedScoreError（score.py:305-320）。
6. **fallback comparable 是三重校验**：同 evidence（scorable/message_piece_id）→ 同 categories → 同 expectation（fallback_scorer.py:106-126），逐条 ValueError。
7. **undetermined 判定在 score 层前置**：`status is UNDETERMINED or score_value is None`（score.py:317）。
8. **attribution 二选一 + objective_sha256 重建**（attack_executor.py:198-214 docstring）。
9. **memory 标识表镜像**（memory_models.py:455 "concrete identifier tables inherit the abstract"）确认。
10. **resume 运行**（scenario_run_service.py:253 + :282 准入 + :311 恢复）确认。

## 二、与考古产物对比（判定）

| # | 独立发现 | 考古产物对应 | 判定 |
|---|---|---|---|
| 1 | outcome 真值语义 | EK-14/15（score 唯一权威 + undetermined 不弱化） | **CONFIRMED**（且补充：bool(value) 非阈值细节→进 05 H-10） |
| 2 | 生命周期 5 事件 + ContextVar RetryCollector | EK-02/03（三段式 + 事件化） | **PARTIALLY_CONFIRMED**（方向正确，缺 ContextVar 注入细节→遗漏修正） |
| 3 | 异常执行上下文 + 根因链 | EK-02 未覆盖 | **MISSING**（补充 EK：错误诊断链） |
| 4 | register_class 双条件校验 | EK-08/10 | **CONFIRMED** |
| 5 | Score.get_value 语义 | EK-15 | **CONFIRMED**（补充 true_false 大小写不敏感细节） |
| 6 | fallback 三重校验 | EK-17（"同型才可回退"） | **PARTIALLY_CONFIRMED**（"同型"应细化为同证据+同类别+同期望→修正措辞） |
| 7 | undetermined 前置判定 | EK-15 | **CONFIRMED** |
| 8 | attribution 二选一 | EK-07 | **CONFIRMED** |
| 9 | memory 标识表镜像 | EK-29/30 | **CONFIRMED** |
| 10 | resume 恢复 | EK-37 | **CONFIRMED** |

## 三、判定统计
- **CONFIRMED**：7
- **PARTIALLY_CONFIRMED**：2（#2 生命周期细节、#6 fallback 措辞）
- **DOWNGRADED**：0
- **OVER_GENERALIZED**：0
- **MISSING**：1（#3 异常执行上下文诊断链）
- **CONTRADICTED**：0
- **NEEDS_HUMAN_REVIEW**：0

## 四、3 成功（考古产物经受住攻击）
1. **KO-01（可判定判定链）**：攻击式验证 `attack_outcome_from_score` 全貌——undetermined 确实不弱化为失败，`UndeterminedScoreError` 在 score 层前置拦截（score.py:317）。单案例→Pattern 的升维成立（判定契约跨系统可迁移，但 X-01 跨项目验证仍 pending）。
2. **KO-02（注册即校验）**：registry.py:497 确认"reference 参数必须映射 wired registry"是注册时校验——"no post-hoc sweep"非虚言。Pattern→L4 升维有据。
3. **KO-06（审计链持久化）**：attribution 二选一 + objective_sha256 重建 + memory 哈希标识表（memory_models.py:638-647 FK→hash）——证据链完整，Flow Edge（E-E1/E-M3）真实。

## 五、3 错误/需修正（盲审计发现，纳入 Reconciliation）
1. **E-01（措辞修正）**：EK-17 fallback "同型才可回退"应细化为"**同证据 + 同类别 + 同期望**三重校验"（fallback_scorer.py:106-126 实际三条 ValueError）。
2. **E-02（深度补充）**：EK-02/03 生命周期描述缺 **RetryCollector ContextVar 注入**（strategy.py:335-343）——事件 handler 在子任务、ContextVar 副本、collector 必须在父任务设置才能两路可见。这是"重试观测如何跨任务可见"的关键实现，补充进 EK-26。
3. **E-03（新增发现）**：**异常执行上下文诊断链**（strategy.py:355-378：component_role + 根因链 + 读后清空）——考古未覆盖，应新增 EK-46。

## 六、遗漏（Auditor 独立重搜）
1. **converter 内部算法**（16,460 行）未深挖——覆盖限制已在 run_metadata 声明，Auditor 同意。
2. **pyrit/analytics 调用点**：独立 grep 未见主链调用——与 [D] 标注一致，无遗漏。
3. **ty 类型检查配置**（pyproject [tool.ty.*]）：考古未提——低影响，纳入 reconciliation 备注。
4. **_ObjectiveTargetConversationLifecycle**（attack_strategy.py:95-130）：objective-target 会话跟踪/释放的显式生命周期类——考古 EK 层未覆盖，补充为 EK-47。

## 七、Benchmark case（本考古对 skill 质量的检验案例）
- **case**：本轮的盲重建是否发现"考古产物与代码不符"？→ 0 CONTRADICTED。
- **case**：EK Graph 是否防退化（无"模块说明"式孤立条目）？→ 45 条 EK 平均出边 ≥1.7、游离仅 2 条（[D] 显式标注）——skill v3.1 的 EK Graph 机制在 PyRIT 规模（19 万行）下有效。
- **case**：升维是否克制？→ KO 无 L5、8 个 KO 全部声明 aggregation_rule、跨项目假说全部标 pending——v3 三层架构在大型仓库有效。
- **skill 缺陷**：无系统性缺陷（E-01~E-03 均为考古执行层面的覆盖/措辞问题，非 skill 机制缺陷）→ **Skill Evolution 不触发**。

## 八、Reconciliation 输入（供阶段⑥）
- 采纳 E-01（EK-17 措辞细化）、E-02（EK-26 补充 ContextVar 细节）、E-03（新增 EK-46 错误诊断链）、遗漏 4（新增 EK-47 目标会话生命周期）。
- 考古产物本文件不改；上述修正写入 corpus 版本（阶段⑥ Reconciliation 合并）。

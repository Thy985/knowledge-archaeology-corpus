# 05 · Candidates — PyRIT（ARCH-2026-10-03-001）

> 未验证假说与跨项目候选。认知状态：Hypothesis（未验证）——**不得写成已验证 Fact/Principle**。
> 每条标注：类型 / 当前证据 / 缺失证据 / 验证路径。

## A. 项目内假说（H-01 ~ H-09）

**H-01 ScenarioRunService 的 resume 恢复在实际故障下的完整性**
- 当前证据：`resume_run_async`(scenario_run_service.py:253) + `_restore_launch_request`(311) + `_validate_resume_admission_async`(282)——机制存在且有准入校验。
- 缺失证据：未见端到端故障注入测试证明"进程崩溃→恢复"不丢任务状态（tests/end_to_end 仅 5 文件）。
- 验证路径：end_to_end 测试中断言 kill 后 resume 的结果一致性。

**H-02 `_DynamicStopAfterAttempt/_DynamicWaitRandomExponential` 的环境变量热更新在真实重试场景被使用**
- 当前证据：`exception_classes.py:74-95`——"reads the environment variable on each retry check"。
- 缺失证据：未在调用链中确认实际生产路径读这些类（retry_collector 存在，但具体接线未穷尽）。
- 验证路径：grep 调用点 + integration 测试。

**H-03 CapabilityName 枚举是中心化词汇表而非完全目标自描述**
- 当前证据：`discover_target_capabilities.py`(1004) 定义 `CapabilityName`；`modality_router.py` 按能力路由。
- 缺失证据：新目标是否必须改中心枚举（若必须，则"自描述"是半自描述）。
- 验证路径：读 discover_target_capabilities.py 的注册机制。

**H-04 memory 标识表"具体继承抽象"模式在跨项目（不同 PyRIT 部署）查询可复现性**
- 当前证据：`memory_models.py:455` ComponentIdentifierEntry docstring + hash 主键（memory_models.py:638-647）。
- 缺失证据：未见跨部署迁移/复制验证测试。
- 验证路径：alembic 迁移链 + partner_integration。

**H-05 SequentialAttack 的 SequenceCompletionPolicy 在真实基准场景中被大量使用**
- 当前证据：`sequential_attack.py:48` 定义；场景层存在 benchmark/adversarial.py(972)。
- 缺失证据：未确认基准场景实际启用顺序攻击的占比。
- 验证路径：grep scenario/ 下 SequentialAttack 使用。

**H-06 `analytics/` 模块（803 行）是规划中或遗留模块**
- 当前证据：analytics_exception.py:7-27 异常族存在；主攻击链未见调用。
- 缺失证据：调用点。
- 验证路径：全仓 grep Analytics。

**H-07 `_uses_legacy_*` 兼容层对应的旧 API 是否仍被外部依赖**
- 当前证据：`memory_interface.py:432,522` legacy override 存在。
- 缺失证据：外部（第三方）依赖情况不可见。
- 验证路径：GitHub issues/依赖图。

**H-08 modality_router 对多模态目标的实际路由覆盖率**
- 当前证据：`modality_router.py` 存在；converter 族（16,460 行）覆盖 text/audio/video。
- 缺失证据：未逐目标验证路由矩阵。
- 验证路径：integration tests 按目标×模态矩阵。

**H-09 数据集（pyrit/datasets 17,937 行，58 .prompt 文件）的版本化与引用纪律**
- 当前证据：datasets 目录规模；`.pyrit_conf_example` 的 `load_default_datasets`。
- 缺失证据：数据集是否随仓库版本化、prompt 是否可溯源。
- 验证路径：datasets/ 目录审查。

**H-10 float_scale 评分在 outcome 映射中的"非零即成功"语义（bool(value)）**
- 当前证据：`attack_outcome_from_score` 用 `bool(value)` 判定（attack_strategy.py:88-93）；`Score.get_value` 对 float_scale 返回原始 float（score.py:305-320）。
- 缺失证据：float_scale 场景的 expectation/阈值是否在更上游处理（若否，0.0 以外的任意值均读作 SUCCESS）。
- 验证路径：float_scale 评分测试 + 上游调用点。

## B. 跨项目假说（X-01 ~ X-05，Cross-project validation pending）

**X-01 [跨项目] 攻击/评测框架的"undetermined 不弱化"是可推广的评估诚实性模式**
- 当前证据：PyRIT `UndeterminedScoreError` + outcome 映射（score.py:53 / attack_strategy.py）。
- 对照：deepseek-harness 的 evidence_strength 收紧（RUN-013：Emulator PASS ≠ release gate PASS）——同族"不弱化证据标准"。
- 缺失证据：仅两个项目样本。
- 验证路径：promptfoo/garak/Giskard 考古对照。

**X-02 [跨项目] "注册即校验（单入口 + 注册时验证）"是插件系统抗漂移共性**
- 当前证据：PyRIT registry.py docstring。
- 对照：codex 考古（已见）中无同构记录——待验证。
- 验证路径：下一插件化项目（如 omnigent/agent-framework）考古时对照。

**X-03 [跨项目] 攻击框架的"结果行自带 attribution + objective 哈希"可推广为审计性系统的标识模式**
- 当前证据：attack_executor.py:198-214 + memory hash 主键。
- 对照：rampart（同厂 MS）权限审计链——待对照。
- 验证路径：rampart 考古结果重读。

**X-04 [跨项目] 防御侧与攻击侧框架共享"判定权威唯一化"约束**
- 当前证据：PyRIT score 唯一权威（attack_strategy.py）。
- 对照：guardian/aigis/ai-protector（corpus 已考古）判定链——待对照。
- 验证路径：重读 guardian 考古包判定部分。

**X-05 [跨项目] "能力自描述 + 前置条件契约"适配层在多目标系统可复用**
- 当前证据：discover_target_capabilities + TargetRequirements。
- 对照：browser-use（corpus 09-22 已考古）目标适配——待对照。
- 验证路径：重读 browser-use 考古包。

---
## 反例候选（Counterexample candidates，供 06 使用）
- **C-01** SequentialAttackResult envelope 与子结果一等行并存：组合层结果语义可能重复（sequential_attack.py:114 vs AttackResult）——已验证非矛盾（envelope 聚合 + 子行保留审计）。
- **C-02** CapabilityName 中心枚举 vs "目标自描述"：见 H-03。
- **C-03** legacy override 层（memory_interface.py:432）与"现代架构"叙事并存——兼容层存在暗示迁移未完成，非"完美重构"故事。

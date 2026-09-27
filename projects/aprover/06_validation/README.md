# 06 · Validation & Evidence — AProver

> 先 Blind Reconstruction（独立重读仓库建立事实），再对比；不弱化证据标准换全绿。
> run: ARCH-2026-09-28-001 ｜ commit b9314c7

## 1. Truth Audit（事实审计）
- 全部 EK 证据来自仓库实际内容（文件/行号见 02）；本机实测 soundness_policy 行为 6 例全符合文档
- pytest 抽样: 213 passed / 1 failed / 2 skipped / 2220 deselected
- 失败项: test_agentic_components::test_agentic_keeps_classifier_on_realism_tools_on_triage_off_dynval_on
  → 实现符合 README(agentic 模式 realism 默认 OFF), 测试未同步 = 三方漂移（EK-16, 保留为工程观察非缺陷）

## 2. Coverage Audit（覆盖审计）
- 子系统覆盖: 管线(✓) 规格引擎(✓) 验证引擎(✓) 判定治理(✓) 动态验证(✓) realism(✓) 报告(✓) 配置(✓)
- 未深读: web/ FastAPI 前端、integrations/ CLI 集成、nix/ 构建——非核心认知面（如实披露）

## 3. Flow Audit（流审计）
- 七类流全部从真实代码导出（04）; 关键 Edge 可回溯 symbol/file/condition（Control: AMCPipeline.run→_run_impl; State: feedback_cleared 免疫; Authority: soundness_policy.resolve_action; Data: dsl_to_cbmc.translate_atom）
- Flow→KO 交叉校验: 无矛盾（04 §7 校验表）

## 4. Abstraction Audit（升维审计）
- KO-06(智能≠权威): 3 项目收敛(AProver/Aigis/Tafcm) → L4 成立但标注 3 项目证据; 不写 Principle/Law
- KO-02/05: L4 认知模型, 单一项目导出 → 解释范围扩大论证明确列出反例与边界
- KO-07: L5 方法论, 标 Cross-project validation pending（单一项目）
- 升维审查: 无跳步（每条 KO 可回溯 EK/证据链）; 无'结论漂亮就升层'

## 5. Counterexample Audit（反例审计）
- 每个 L3+ KO 配置 ≥2 反例（03 矩阵）; 反例均保留在文档（不因通过而删除）
- 关键反例: KO-02 信任链假设 solver 可信(元验证缺失); KO-03 模式库跨项目迁移性存疑; KO-04 stub postcondition 质量无默认门禁(Phase4 opt)

## 6. Epistemic Audit（认知状态审计）
- Fact/Observation/Hypothesis/Pattern/Model/Methodology 严格区分（02 EK 为 Fact/工程知识; 03 标 epistemic_status; 05 标 Hypothesis/Cross-project）
- EK-16(漂移) 为 Observation 级（实测确认, 非假设）; C2 是 Hypothesis（单例不可外推）
- 无 Hypothesis 冒充 Fact; 无 Cross-project Candidate 冒充已验证 Principle

## 7. Contradictions & Counterexamples（保留项）
- C-1: 测试(test_agentic_components) 与 实现/文档 矛盾（期望 realism=True vs 实际 False）→ 判定: 实现符合文档, 测试过时; 保留
- C-2: README 称 'agents propose, tools dispose' 严格性 vs LLM fallback 在 reachability 存在 → 判定: fallback 不授予删除权, 不违反; 保留为边界说明
- C-3: PIPELINE.md 'confirmed_dynamic immune to realism downgrade except main.assertion.N' vs bug_reporter.py L127-129 存在 Phase4b 开关(可让 immunity 失效) → 判定: 配置可选, 默认免疫成立; 保留

## 8. Reconciliation
- 初稿 EK-16 曾标为'缺陷', 经独立审计修正为'三方漂移(实现符合文档, 测试滞后)' —— 见 independent-audit 判定 #16
- 初稿 KO-06 曾标 Principle, 盲验证后收紧为 L4 Cognitive Model（3 项目收敛, 仍需更多领域实例）


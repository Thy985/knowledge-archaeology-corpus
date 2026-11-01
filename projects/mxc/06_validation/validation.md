# Validation & Evidence — ARCH-2026-10-24-001（MXC）

> 六 Auditor 视角 + Blind Reconstruction 原则。先独立重读，再对比结论。

## 1. Truth Auditor（这句话是真的吗）
抽查 8 个关键断言（全部重新 grep 验证）：
| 断言 | 验证 | 结果 |
|---|---|---|
| closed JSON 拒绝 unknown field | config_parser.rs:2013/2533/2587/3819 测试锚 | ✅ |
| deny wins over allow | network_rules.rs:678-679 + :756-757 | ✅ |
| FS fail-closed trigger | filesystem_object.rs:94（"fail-closed trigger when deniedPaths are present"） | ✅ |
| 后端注册 10 项 + ProbeCoverage 强制 | backend_registry.rs BACKENDS[10] + 注释 | ✅ |
| telemetry 未知值 Blocked | telemetry.md 表格（"anything else — Blocked (fail closed)"） | ✅ |
| IsolationSession 必须全 allow 显式 | policy.md + invalid/isolation-session-without-network.json | ✅ |
| 三轴分离 | versioning.md 权威表 | ✅ |
| 双 Learning capability 语义分离 | logging-access-denied.md 表 + "Weakened" | ✅ |
**Truth: 8/8 通过**

## 2. Coverage Auditor（还有什么重要东西没发现）
强制子系统覆盖检查：
- [x] 策略契约（policy.md / schemas stable+dev / schema-version.json）→ EK-M01/M02/M14/M20
- [x] 引擎（backend_registry / run / state_aware / experimental / probe）→ EK-M05/M06/M07/M15
- [x] 后端（9 实现 + docs/backends 8 文档）→ EK-M03/M04/M10/M11/M12/M16
- [x] Learning Mode（plm / learning_mode_core / logging-access-denied）→ EK-M08/M09
- [x] 治理（AGENTS / copilot-instructions / review skill / SECURITY）→ EK-M17/M18
- [x] 测试体系（regression_tests / policy sdk-v1 / fuzz / testing 工具）→ Project Layer
- [ ] telemetry 内部（verbose_telemetry / mxc_telemetry 细节）→ **缺口：留 Candidates/C-M05 邻域（本轮 EK-M13 覆盖策略层，实现层未深挖）**
- [ ] CI 流水线细节（ci-validation-infrastructure 运行矩阵）→ 部分覆盖（EK-M17 提及），未逐 job 读
**Coverage: 声明缺口 2 处，不弱化全绿**

## 3. Flow Auditor（Flow Atlas 是否对应代码）
- resolve_runner Edge：run.rs 实读 ✅
- lifecycle 前缀 Edge：policy.md + state-aware-wslc 测试 ✅
- DenialAnalyzer 窗口过滤 Edge：analyze.rs 实读 ✅
- telemetry ceiling Edge：telemetry.md 实读 ✅
- proxy 约束 Edge：policy.md 100-130 实读 ✅
**Flow: 5/5 Edge 可回溯；未见虚构 Edge**

## 4. Abstraction Auditor（L3/L4 是否过度升维）
- KO-M02（三轴分离 L4）：从 versioning.md 明确权威表提炼——解释范围确实扩大，但仅 MXC 单项目 → **降为 L4 + Cross-project validation pending**（不加 L5）
- KO-M05（单一引擎 L4）：backend_registry 的"决策成本显式化"有直接注释证据（"Declaring this is mandatory"）→ 维持 L4 + pending
- KO-M06（治理代码化 L4）：AGENTS.md 是 guideline 非强制机制（反例已列）→ 维持 L4 + pending；**严禁升 L5**
- KO-M01（fail-closed L3）：strands-box 跨项目对照支持 → 维持 Pattern（不升 Principle，因反例存在）
**Abstraction: 无过度升维；4 条 L4 全部 pending 标记**

## 5. Counterexample Hunter（反例预算）
| KO | 定向反例攻击 | 结果 |
|---|---|---|
| KO-M01 | permissiveLearningMode 允许一切（audit 模式"Weakened"）——fail-closed 不是不变量 | **反例成立**（已在 KO 记录：fail-closed 是默认态） |
| KO-M01 | Seatbelt profileOverride 完全绕过生成策略 | **反例成立**（显式逃生门） |
| KO-M02 | dev 契约 1.1.0-alpha 可变无 notice——"immutable"非全局 | **反例不成立**（三轴内状态位，非矛盾） |
| KO-M03 | proxy 明示"不保证任意直连被拦"——能力边界诚实有反例吗？ | 反例：ProcessContainer 代理需 ingress allow 是能力妥协，已被 EK-M10 覆盖（非反例） |
| KO-M06 | AGENTS.md 无强制（lint 不执行） | **反例成立**（已在 KO 记录） |
**反例预算: 3 成立 / 2 驳回；全部保留在产物中**

## 6. Epistemic Auditor（认知状态诚实）
- Fact（L0）: 项目地图全部可回溯
- Engineering Knowledge（L1/L2）: 20 条 EK 全部 S3+（implemented）或 S4（测试验证）；无"我认为"
- Pattern（L3）: KO-M01/M03/M04 —— MXC 内多实例一致；KO-M01 有 strands-box 跨项目
- Cognitive Model（L4）: KO-M02/M05/M06 全部标 `Cross-project validation pending`
- Candidates: 7 条全标 Hypothesis；C-M07 明确"未验证前不得写成 Principle"
- **Contradictions 保留**: 文档-实现无重大矛盾（versioning 文档与 schema-version.json 一致）；README "experimental" 标记与 backend_registry experimental 标志一致
- **Counterexamples 保留**: 3 条（permissive/profileOverride/AGENTS 无强制）全部写入 KO 反例段

## 质量指标（初版，pending Auditor 复核）
- EK 20（links 100%，avg 出边 ~1.9，游离 0）；KO 6（R1×1 R2×2 R3×1 R4×2）；Candidates 7；Flow 7 类；evidence 引用 ~60 处
- Truth 8/8；Coverage 缺口 2 声明；Flow 5/5；Abstraction 无过升维；反例 3 成立保留

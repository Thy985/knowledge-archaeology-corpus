# 06 Validation & Evidence — Transect（ARCH-2026-10-13-001）

> 六类审计。全部先 Blind Reconstruction（独立重读仓库 → 建立 Independent Findings → 对比考古包），不弱化证据标准换全绿。独立 Auditor 报告另见 `independent_validation_report.md`（阶段⑤）。

## 6.1 Truth Auditor

| # | 声明 | 裁决 | 证据 |
|---|---|---|---|
| T-1 | 默认扫描批含 4 个零 LLM 结构扫描器 | CONFIRMED | api.py:_scanners() 4 个 base 扫描器；judge_models=None 分支 WARNING |
| T-2 | 三 regime 互斥且 cohort 时 verifier 自动 off | CONFIRMED | phases.py resolve_verify + cohort docstring；test_reliability test_judge_setup_classifies_the_regime |
| T-3 | 投票规则 modal → tie mean conf → tie order | CONFIRMED | phases_cohort.py:vote_turns docstring；cohort.py:vote_ballots |
| T-4 | verifier 触发 4 类且 overturn gate 0.6 | CONFIRMED | phases_verify.py:select_for_verify；cohort.py:LOWCONF=0.60；VerifierAudit 字段 |
| T-5 | Spec extra="forbid" 三模型 | CONFIRMED | spec.py Phase/SubagentLabel/Spec model_config |
| T-6 | Mean CI 仅 N≥8 显示 | CONFIRMED | reliability.py _CI_MIN_N=8 + NO_MEAN |
| T-7 | SECTION_KEYS 7 键且 reliability audit 恒最后 | CONFIRMED | report/render.py SECTION_KEYS；render_report 顺序 |
| T-8 | 279 测试 | CONFIRMED | grep -c "def test_" tests/ 实测 279 |
| T-9 | 论文演示 1300 万 token | CONFIRMED(论文证据) | arXiv 2610.08364（仓库非复现内容；报告标注为论文侧证据） |
| T-10 | OpenClaw span 结束合成 | CONFIRMED | scanners/base.py ends_recorded 分支 |

**未发现 Contradiction**（独立 Auditor 攻击见阶段⑤）。

## 6.2 Coverage Auditor

| 维度 | 覆盖 | 说明 |
|---|---|---|
| 入口/API | ✓ api.py 全链（transect/load/render/_run/_scanners） |
| 配置契约 | ✓ spec.py / selection.py / layers.py |
| 判定核心 | ✓ phases / phases_cohort / phases_common / phases_verify / phases_narrate / cohort |
| 可靠性 | ✓ reliability.py 全 + report/reliability.py flags |
| 结构扫描 | ✓ base.py 4 扫描器 |
| 帧/报告 | ✓ frames（builtin_frame_names）+ report/render（SECTION_KEYS） |
| 执行记录 | ✓ scan_status.py |
| 分发 | ✓ skills.py + pyproject force-include + .claude/ |
| 导入 | ✓ openclaw_telemetry_hal（vendored 接口语义） |
| 测试揭示 | ✓ test_reliability / test_token_accounting / test_decision_phases / test_scan_status / test_skills 等 279 测试抽样 |
| 未覆盖（诚实披露） | tests 全部 279 条未逐条精读（抽样关键族）；ingestion hal 内部实现未深读（vendored 上游，只记录接口语义）；scout viewer 运行时 UI 未实测（无图形环境） |

## 6.3 Flow Auditor

| 检查 | 结果 |
|---|---|
| Control/Data Flow 边可回溯 symbol/file | ✓（04 表每边带 file:symbol） |
| Evidence Flow anchor_event_id 真实性 | ✓ phases.py:turn_digests → anchor_of → decision_phases |
| Authority 分层（narrator 无改判权） | ✓ phases_narrate.py（NarratorAudit 只有 n_fallback，无 label 改写字段） |
| Memory Flow cache 语义 | ✓ phases.py cache 默认 + test_token_accounting |
| Policy Flow 阈值常量集中 | ✓ cohort.py/reliability.py/report/reliability.py 模块级常量 |
| bypass/override/exception 路径 | ✓ verify=False 显式降级；judge_models=None 结构层独立；cohort 时 verifier off；OpenClaw anchor=None；narrator fallback |

## 6.4 Abstraction Auditor（L3+ 升维审查）

| KO | 升维裁决 | 依据 |
|---|---|---|
| KO-01 L4 | 保留（Cross-project pending） | 链条证据完整但单项目；不升 L5 |
| KO-02 L4 | 保留（Cross-project pending） | 溯源不变量在仓库内强证据；跨项目未验证 |
| KO-03 L3 | 保留 | 跨两判定面 + 契约化自定义层 → 跨实例性成立 |
| KO-04 L3 | 保留 | 高自洽设防为本项目独特证据；反例 C-2 说明 α 高未设防 → 不升 L4 |
| KO-05 L4 | 保留（Cross-project pending） | 策略常量治理模式仓库内强证据；通用性未验证 |

无降级；无"单案例→Pattern"违规（KO-03 跨两实例）。

## 6.5 Counterexample Hunter（反例预算：每个 L3+ KO ≥3 定向攻击）

| KO | 攻击 | 结果 |
|---|---|---|
| KO-01 | OpenClaw anchor=None 断链；narrator fallback；solo 无 CI | 链条"尽量内建"非绝对——模型表述已收紧，不推翻 |
| KO-02 | solo 无 agreement/CI；layer 无 judge-identity 抛错 | 溯源与可信度量分离成立；违反即显式失败 |
| KO-03 | cohort 时 verifier off（三层→两层）；solo 同模型二轮 | 模式是"可组合校正栈"——表述已收紧 |
| KO-04 | 低自洽仅 amber；α 高无 flag | 高自洽设防确认；"一致性-精度无关"仅提示未 flag（C-03） |
| KO-05 | 5% 抽检与 0.6 gate 为启发式；verify=False 可关 | 治理可显式降级是特性——表述已收紧 |

**Counterexamples 未推翻任何 KO，但触发 3 处表述收紧**（记录于 03 层论证块）。

## 6.6 Epistemic Auditor

| 检查 | 结果 |
|---|---|
| Hypothesis 不冒充 Fact | ✓ 05 Candidates 全部标 Hypothesis/暂定/跨项目 |
| Cross-project Candidate 不写成已验证 Principle | ✓ KO 均标 `Cross-project validation pending` |
| Fact/Observation/Hypothesis/Pattern/Model 状态区分 | ✓ 03 层状态列；EK 层全部 L1/L2 |
| 论文证据与仓库证据分离 | ✓ T-9 标注"论文侧证据"；snapshot 边界声明 |
| 结论范围不超过检查范围 | ✓ 6.2 诚实披露未覆盖项 |

## 6.7 Quality Metrics（本包）

| 指标 | 值 |
|---|---|
| EK 数（EK Graph） | 16，links 覆盖率 100%，游离 0，六类边全用 |
| KO 数（Generalized） | 5，aggregation_rule 覆盖率 100%，簇内 EK 全可回溯 |
| Candidates | 6（全部显式 Hypothesis/暂定/跨项目） |
| Flow 类 | 7/7（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 测试锚点 | 279 测试（仓库实测）；本包引用测试族 5 个 |
| Truth 审计 | 10 条声明 10 CONFIRMED，0 Contradiction |
| Counterexample | 5 KO × ≥3 攻击，0 推翻，3 处表述收紧 |
| 证据来源密度 | 每条 EK/KO/Flow 带 file:symbol 或测试/论文引用 |

## 6.8 Reconciliation（阶段⑥执行时合并独立 Auditor 修正）

- 独立 Auditor 报告：`independent_validation_report.md`（阶段⑤）
- 修正合并结果：见 corpus `projects/transect/06_validation/`（Reconciliation 记录）
- **规则**：发现问题的修正写入 Corpus Artifact，不修改生产 Skill；本包与 Auditor 报告均保留原状（Auditor 禁改原考古产物）

## 6.9 Skill Evolution 评估

- 是否暴露系统性 skill 缺陷：**否**（本包六类流/三层架构/EK Graph 全部在现有 v3.2 契约内完成，无流程性失败）
- 结论：**不产生 Skill Improvement Proposal**（SIP=none）

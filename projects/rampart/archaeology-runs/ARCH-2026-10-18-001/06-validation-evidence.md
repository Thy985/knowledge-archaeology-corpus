# 06 · Validation & Evidence — RAMPART refresh（ARCH-2026-10-18-001）

> 本文件 = 主考古自检（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic 六审）。独立 Auditor 盲重建见 06_validation/independent-validation.md（阶段⑤产出，独立对比）。

## 1. Truth Audit（Blind Reconstruction 摘要）
主考古的每条 refresh 增量事实均已直接回溯 HEAD 0fb9cc2 仓库内容（文件:行）。关键声明的盲重建对照（独立 Auditor 独立重读后给出）见阶段⑤报告。此处记录主考古自检时主动寻找的反证：
- **反证搜索 1**（EK-R05 渐进迁移）：grep 确认 _xpia.py/_single_turn.py **无** run_trace_async 引用 → 证实"未迁移"是事实而非假设 ✓
- **反证搜索 2**（EK-R01 双视图）：trace.py:51-59 docstring + 实际 append 逻辑（raw_turn 无 eval_result 先 append raw，再 replace 后 append turns）→ 双视图实现真实存在 ✓
- **反证搜索 3**（EK-R02 证据分层）：docs 明示 "not simply the last online evaluation" + codec 注释 "never infers" → 代码与文档一致 ✓

## 2. Coverage Audit
| 子系统 | 主考古覆盖 | 状态 |
|---|---|---|
| core/trace.py | 全部（run_trace_async / evaluate_final_trace_async / EvaluationRecord / _matches） | ✓ |
| core/serialization.py | 全部（信封/decoder/strict JSON） | ✓ |
| core/_schema.py | adapter-local 策略 / population bounds / payload 约束 / json_value | ✓ |
| core/_population.py | 全部 5 个 validator | ✓ |
| pytest_plugin/_trial.py | resolve_trial_config / parse_positive_int / CLI override | ✓ |
| plugin.py trial 校验 | _validate_trial_declarations / trial_config fixture | ✓ |
| _xdist.py | SCHEMA_VERSION / 权威 nodeid/index / 信任边界 | ✓ |
| result.py | PopulationRef / final_trace_evaluation / turn_evaluations | ✓ |
| reporting/json_file.py | duplicate timestamp 修复 | ✓ |
| skills/rampartner | conversation contract 全文 | ✓ |
| schemas/ + scripts/ | compatibility gate 全流程 | ✓ |
| docs（trace-schema / results-and-reporting / onboarding） | 语义权威文档 | ✓ |
| 09-04 基线核心 | evaluator 三值 / xdist 边界 / XPIA / SingleTurn / _enforce_incomplete | 抽查确认未变 ✓ |

## 3. Flow Audit（关键 Edge 真实性抽查）
- **Edge 1**（Control）：`stop_when.detected → trace_end_reason=STOP_CONDITION_MET → return`——trace.py:172-174 属实 ✓
- **Edge 2**（Data）：`deserialize → version 未知 → UnsupportedSchemaVersionError`——serialization.py:182-185 属实 ✓
- **Edge 3**（Policy）：`同 major 变更但 decision≠COMPATIBLE → ValueError`——check_trace_compatibility.py:152-157 属实 ✓
- **Edge 4**（Authority）：`envelope nodeid != report nodeid → WorkerOutputError`——_xdist.py deserialize_report_data 属实 ✓

## 4. Abstraction Audit
| 对象 | 判定 | 理由 |
|---|---|---|
| KO-R01（终局证据分离） | L4 成立 | 解释范围扩大到任何长流程验证系统；代码+docs 双重证据 |
| KO-R02（契约显式决策） | L4 成立 | 指纹→决策→门禁闭环；test 锁定；跨领域可迁移 |
| KO-R03（raw 视图隔离） | L4 成立 | 评估污染回路是普遍问题；但**跨项目验证 pending**，不冒充 Law |
| KO-R04（schema/live 分离） | L3 成立 | Pattern 级；本项目证据充分 |
| KO-R05（类型纪律） | L3 成立 | 常见工程实践；本项目证据充分 |
| 降级检查 | 无过度升维 | 所有 L4 均标 cross-project pending；无 Principle 级断言 |

## 5. Counterexample Audit（反例预算）
| KO | 反例攻击 | 结果 |
|---|---|---|
| KO-R01 | 若 status 已含终局评测信息，推断是否有害？ | 反例成立——但 codec 明示不推断，设计主动防御；结论维持 |
| KO-R02 | 若契约从不变化，gate 是否多余？ | 反例部分成立——gate 是防御未来变更；成本可接受（CI 脚本快）|
| KO-R03 | 若 evaluator 需要看历史评估（如级联判断），raw 隔离是否过严？ | 反例成立——级联评估场景需要历史；KO scope does_not_apply 已覆盖（"评估器无状态或历史不可见"）|
| KO-R04 | 若运行时也需要严格约束，副本分离是否绕远？ | 反例部分成立——但本项目运行时明确宽松（XPIA 构造合法，序列化可拒）；维持 |
| KO-R05 | bool 在某些 DSL 中有意义（0/1 枚举）？ | 反例部分成立——但 population/trial 参数语义明确无 bool；维持 |

## 6. Epistemic Audit
- 全部 refresh 增量 Fact 均带 文件:行 证据，无无源断言 ✓
- Hypothesis（C-R01~C-R06）全部进 Candidates，未冒充知识 ✓
- Pattern/Model 均标 `(RAMPART validated; cross-project pending)`，无 Law 级断言 ✓
- "未迁移"（EK-R05）是 Observation+Implementation 事实，不是推测（grep 证实）✓

## 质量指标（refresh 子图）
| 指标 | 值 |
|---|---|
| refresh 增量 EK | 15 条（EK-R01~R15） |
| refresh 增量 KO | 5 个（KO-R01~R05） |
| Candidates | 6 条（C-R01~C-R06，其中 1 Observation / 5 Hypothesis） |
| EK 平均出边 | ~2.2（子图 33 条边 / 15 EK） |
| 游离 EK | 0（全部 ≥1 出边） |
| KO 聚合规则覆盖率 | 100%（R1×4 + R2×1） |
| 反例预算 | 5 KO × 各 1+ 攻击，无 0 反例 |
| 证据强度 | S3~S4（实现 + 单测锁定） |
| 与 09-04 基线连接 | 5 条（EK-R01→EK-01, R02→EK-08, R10→EK-25, R12→EK-26, R13→EK-20） |

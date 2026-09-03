# 06 · Validation & Evidence

> 验证原则：Blind Reconstruction——独立重做一遍再判断，不是"证明这个答案没问题"。
> 本 run 的独立重建 = ① 重读全部核心模块源码 ② 用独立脚本实测核心机制（resolve 语义/三值逻辑/Payload 校验/PayloadStore 安全）③ 对照 docs 设计意图。

## 0. 重建方法（Blind）
- 未以"已有报告"为事实来源；从 `repo/rampart/` 源码 + `tests/` + `docs/` 独立提取
- 核心机制实测（python 脚本，见下方结果）：resolve_as_attack/probe 优先级、&/|/~ 三值逻辑、Payload 格式一致性、PayloadStore 路径逃逸 7/7 拒绝
- 环境限制：pytest/pyrit 未安装 → 未跑官方单测套件；以源码精读 + 独立实测替代（已在 01/02 标注证据位置）

## 1. Truth Auditor（Source Truth）
| claim | 判定 | 证据 |
|---|---|---|
| ObservabilityLevel 三档 + 属性方法 | ✅ CONFIRMED | types.py:41-96 |
| EvalOutcome 三态 | ✅ CONFIRMED | types.py EvalOutcome |
| resolver 优先级（attack/probe） | ✅ CONFIRMED + **实测** | result.py:191-239；脚本 assert 通过 |
| 三值组合算子短路 | ✅ CONFIRMED + **实测** | evaluator.py；脚本 assert 通过 |
| bool(result)==safe（derived） | ✅ CONFIRMED | result.py:120-126 |
| trial gate 语义 | ✅ CONFIRMED | _session.py TrialGroupResult.status + _evaluate_gates |
| xdist trust boundary | ✅ CONFIRMED | _xdist.py:10-18 头注释 |
| PayloadStore 路径防护 | ✅ CONFIRMED + **实测**（7/7 恶意引用拒绝） | _store.py + 脚本 |
| 懒加载不拉 pyrit | ✅ CONFIRMED + **实测**（import rampart 无 pyrit 报错；仅 payloads 包级链会拉） | __init__.py + 脚本观察 |

## 2. Coverage Auditor
- **子系统覆盖**：core/attacks/probes/evaluators/drivers/payloads/pyrit_bridge/pytest_plugin/reporting/surfaces/converters/common —— 全部 13 子包已覆盖（01 Project Map + 02 EK）
- **文档对照**：docs/concepts/overview.md（组件/生命周期/Result Contract/Polarity）与 docs/contributing/architecture.md（设计决策/扩展点/import 约定）全部核对
- **测试揭示行为**：test_xpia（observability 降级 5 例/清理/早停）、test_payload_store_security（6 类逃逸）、test_public_api（懒加载）、test_execution（生命周期/broken handler）已纳入 EK
- **潜在遗漏**：
  - pytest_plugin/_xdist.py（1596 行）细节（controller merge 的增量合并）未逐行深读 → 标 C-07 候选
  - llm_judge.py 的 prompt 模板（llm_judge.yaml）未精读 → 模板内部措辞不在本 run 范围
  - onedrive surface / docx converter 为外围扩展，未深读（非核心机制）

## 3. Causality Auditor
- EK-01→05→09 因果链：observability 声明 → 降级 → 判定影响 —— 代码链成立（_adjust_for_observability 调用点确认）
- EK-16→19→20 因果链：trial 聚合 → incomplete 强制 → 大小限—— 代码链成立
- **未被支持的主张**：无（所有 causal link 均落在实际调用/依赖上）

## 4. Flow Auditor
- Control Flow：BaseExecution→_execute_async→XPIA/SingleTurn 阶段顺序，逐一匹配源码行
- 七类流 Edge 全部可回溯（见 04）
- **deviation 记录**：plugin.py 的 handler factory 实现 ≠ architecture 声明的直接写全局（有显式注释）→ 已记录 EK-11

## 5. Abstraction Auditor
| KO | 层 | 判定 | 理由 |
|---|---|---|---|
| KO-01 诚实的不确定 | L4 | ✅ 保留 | 4 子系统跨实例 + 解释范围扩大 + 可命名 + S4 |
| KO-02 极性分离 | L3 | ✅ 保留 | 明确 Pattern（检测/价值解耦） |
| KO-03 observability 上界 | L4 | ✅ 保留 | 因果链 + 可命名 |
| KO-04 执行骨架韧性 | L3 | ✅ 保留 | Template Method 明确实例 |
| KO-05 自噬防御 | L4 | ✅ 保留 | 5 通道跨实例（终端/xdist/存储/judge/size） |
| KO-06 统计验证契约 | L4 | ✅ 保留 | 因果链 + 语义清晰 |
| KO-07 清理不变量 | L3 | ✅ 保留 | 明确设计决策 + 测试锁定 |
| KO-08 LLM 防自欺 | L3 | ✅ 保留 | 3 实例（隔离/校验/降级） |
- 全部 KO 带 scope（applies_when/does_not_apply_when）——主动声明边界，避免 deepseek-harness KO-03 式 scope 泛化
- **未升 L5**：单项目证据不足，Methodology 需要跨项目

## 6. Counterexample Auditor（反例预算：每个 L4 KO ≥2 定向反例）
| KO | 反例 1 | 反例 2 | 结论 |
|---|---|---|---|
| KO-01 | 确定性白盒断言下三态会掩盖真实失败（已入 scope does_not_apply_when） | 空 turns EvalContext 抛 ValueError（"No turns"）而非返回 UNDETERMINED——不确定机制有边界 | scope 收紧，保留 |
| KO-03 | test_response_only_with_tool_calls_stays_safe：有工具调用可见 → SAFE 不降级——证明"工具可见可洗白"边界（C-04 矛盾） | TOOL_ONLY 下 side_effect 不可见 → UNDETERMINED（test_side_effect_undetermined_under_tool_only） | 保留但 C-04 标为未决矛盾 |
| KO-05 | 单进程 sink 失败只 warning 不 FAIL（C-07）——防御链有未覆盖通道 | docs 构建等非攻击面通道无防护（不适用） | 保留（scope 声明"渲染/存储/传输/解析"通道） |
| KO-06 | ERROR trial 计入 pass_rate 分母（CI 文档：ERROR trials count against pass rate）——与"ERROR 从分母排除"的报告口径矛盾 | incomplete 只在 xdist 触发，单进程"报告未持久化"不强制失败（C-07） | 保留（两处口径差异标为 C-04/C-07） |
| KO-07 | 注入激活顺序失败（首个错误被 raise）时已成功 sibling 仍清理（test_partial_activation_failure）——反向验证成立 | 若 handle.wait_until_ready 无限阻塞无超时兜底（TaskGroup 内）——ready 阶段无超时保护 | 保留（ready 超时缺口标 C-10） |
| KO-08 | LLMJudge 需要 CentralMemory 初始化（from_target 注释）——依赖前置未在构造时校验 | 懒加载仅在公共 API 层；payloads 包级 __init__ 链会 eager import pyrit（实测发现）——隔离不完整 | 保留（C-11 标为隔离缺口） |

## 7. Epistemic Auditor
- Fact/Observation/Hypothesis/Pattern/Model 严格区分：EK 层标 L1/L2（Fact/Knowledge），KO 层标 L3/L4（Pattern/Cognitive Model）
- 跨项目声明全部标 `cross-project pending`（KO-01..08）
- Hypothesis 未冒充 Fact：C-01/C-02/C-03 明确标"待验证"
- 未出现"单项目→通用 Principle/Law"的越级

## 8. 反例预算达标声明
- 6 个 L4 KO 均完成 ≥2 定向反例攻击（见上表）
- 反例驱动出 3 个新 Candidate：C-04（工具可见洗白）、C-07（单进程报告缺口）、C-11（payloads eager import）
- 0 反例结论：无（全部 L4 KO 均找到边界）

## 9. 新增 Benchmark / Regression 候选（供 skill 或 corpus）
- **B-RAMPART-1**：observability 降级必须触发（RESPONSE_ONLY+零工具→UNDETERMINED）
- **B-RAMPART-2**：UNDETERMINED 必须是有意显式的（不可静默转 SAFE/UNSAFE）
- **B-RAMPART-3**：路径逃逸防护（PayloadStore 恶意引用全拒）
- **B-RAMPART-4**：trial gate 语义（ERROR→FAIL）
- **B-RAMPART-5**：判断句 scope 必须声明边界（承接 deepseek-harness KO-03 教训的 RAMPART 侧实例）
- **B-RAMPART-6**（独立验证新增）：trial gate 全 no_result → UNDETERMINED → gate FAIL
- **B-RAMPART-7**（独立验证新增）：trial gate pass_rate 达标但含 unsafe → PASS（统计容忍）
- **B-RAMPART-8**（独立验证新增）：非 xdist 序列化超限 → incomplete → 强制非零退出
- **B-RAMPART-9**（独立验证新增）：`und & not=not` / `und | det=det`（UNDETERMINED 组合顺序无关性）

## 9b. Reconciliation 记录（独立验证 → 本包修正）
| 独立验证判定 | 修正动作 |
|---|---|
| EK-14 PARTIALLY_CONFIRMED | 改写为"混合并发策略（entry gather / ready TaskGroup）" |
| EK-16 PARTIALLY_CONFIRMED | 补充 no_result→UNDETERMINED→FAIL 分支 + 统计容忍语义 |
| EK-23 强化 | 补充 NaN/Infinity/bool 拒绝 + outcome 字面量契约 |
| C-07 修正 | 收窄为"`_absorb_results` 异常仅 warning 不设 incomplete" |
| MISSING ×4 | 新增 EK-27（safe_* 解析族）、EK-28（@harm 卫生检测）；judge 契约/ToolCalled 顺序并入 EK-23/EK-01 |
| Policy Flow | incomplete 措辞改为"worker 丢失或序列化超限均强制" |

## 10. Validation 总结
| Auditor | 结果 |
|---|---|
| Truth | 全 CONFIRMED（含 4 项独立实测） |
| Coverage | 13/13 子包覆盖；3 项明确边界缺口（xdist 细节/llm_judge 模板/外围扩展） |
| Causality | 无未支持因果 |
| Flow | 七类流全可回溯；1 处实现-文档偏差已记录 |
| Abstraction | 8/8 KO 保留（带 scope）；未升 L5 |
| Counterexample | 6/6 L4 KO ≥2 反例；驱动 3 个新 Candidate |
| Epistemic | 层级诚实；跨项目全部 pending |
| Independent | 18 CONFIRMED / 3 PARTIALLY / 0 DOWNGRADED / 0 OVER_GEN / 4 MISSING / 0 CONTRADICTED / 1 NEEDS_HUMAN_REVIEW |

**质量判定**：QUALITY GATE PASS（fidelity/critical coverage 达标；无 False Acceptance 候选——8 KO 全部有代码/测试/实测支撑；独立验证 3 处描述精度修正已 Reconciliation）。

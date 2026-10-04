# 06 Validation & Evidence — Webwright（ARCH-2026-10-05-001）

> 阶段④自检（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic），先 Blind Reconstruction，不弱化证据标准换全绿。
> 阶段⑤独立 Auditor 盲审计见 `independent-audit/report.md`（本文件保留原考古判断，不被审计修改）。

## 1. Blind Reconstruction（先重建再对比）
考古者先独立重读仓库（不依赖任何外部二手资料），从零构建以下重建物后，再与本包结论对比：
- 重建物 A：从 `run/cli.py`+`agents/default.py` 手绘一次 run 的消息流与 done 门控顺序 → 与 04_flow Control/Evidence 一致 ✅
- 重建物 B：从 `skill_factory/update.py` 手写"一个技能如何入库"的步骤链（gate→group→canonicalize→replay→grade）→ 与 KO-03 一致 ✅
- 重建物 C：从 `config/base.yaml` system_template/instance_template 归纳 Completion Gate 五条件 → 与 EK-17/EK-53 一致 ✅

## 2. Truth Auditor（这句话是真的吗？—— 只引用仓库证据）
| 判断 | 结论 | 证据 |
|---|---|---|
| "code-as-action：动作=可执行代码" | CONFIRMED | models/base.py `_query_async` + base.yaml system_template（bash_command 字段） |
| "done 需 predicted_label==1" | CONFIRMED | agents/default.py `_self_reflection_gate_error` + `_tool_gate_error` |
| "answer 文件是契约非退出码" | CONFIRMED | build.py + test_build_init（`exit_non_zero_but_wrote_answer` 计 solved） |
| "on_fail=reference 永不覆盖已验证技能" | CONFIRMED | update.py `_refine` guard + test_evolve（v1 存活断言） |
| "replay 是无模型回归裁判" | CONFIRMED | update.py `_replay`（临时目录 subprocess，SKILL_RUN_TIMEOUT 240s）+ replays.json 并入断言 |
| "skills/webwright 插件版替换 image_qa/self_reflection 为宿主原生能力" | CONFIRMED | SKILL.md（"replace that loop directly… No OPENAI_API_KEY required"） |
| "step_limit 默认 15" | PARTIALLY_CONFIRMED | 代码默认 15 但配置层 base.yaml 覆盖 100——"默认"以配置生效值计应为 100 |
| "require_self_reflection_success 默认开启" | PARTIALLY_CONFIRMED（Reconciliation R4） | 与 step_limit 同构：代码默认 False（agents/default.py#AgentConfig:39），base.yaml 配置 true——默认值以配置生效值计 |

## 3. Coverage Auditor（还有什么重要东西没发现？）
- 已覆盖：全部 src 核心（agents/environments/models/run/config/tools/skill_factory/exceptions）+ docs/manual.md + skills/webwright/SKILL.md + tests 关键断言 + CI + README 全量。
- 未细读（记录在案，不影响本包结论）：docs/skill_factory/reference.md、commands/*.md、reference/*.md 细节、config/{local_browser,persistent_browser,crafted_cli,task_showcase}.yaml 全文、model_{anthropic,openrouter}_model.py 细节。
- 覆盖结论：核心机制/决策/失败路径/测试行为/配置/边界均已入包；无 ADR 目录（决策证据转用 README News/代码注释，已在 00 边界声明）。

## 4. Flow Auditor（Flow Atlas 是否真的对应代码？）
- Control/Evidence/Authority/Memory/Policy 五流逐 Edge 回读代码核对 ✅（symbol/file/condition 标注见 04）
- State Flow 中 "step_limit 15 vs 100" 已在 Truth 表标 PARTIALLY_CONFIRMED（配置覆盖语义）
- Data Flow 中 OpenAI `developer` role 映射与 `output_text` 优先抽取已回读 openai_model.py 验证 ✅

## 5. Abstraction Auditor（L3/L4 是否过度升维？）
| KO | 判定 | 论证 |
|---|---|---|
| KO-01 code-as-action | 维持 L4 | 与 browser-use 的对照（Corpus 09-22）构成跨实例证据；标 Cross-project validation pending |
| KO-02 可丢弃浏览器+持久工作区 | 维持 L4 | persistent_local_browser 反例已并入模型（"按需持久"分层） |
| KO-03 验证即重放 | 维持 L4 | gate→replay→grade 链跨 4+ 独立测试断言 |
| KO-04 污染防护 | 维持 L4 | 8 EK 汇聚同一不变量；test 族直接覆盖 |
| KO-05 库查找前置 | 维持 L4 | prompt.py + route 测试（run success 不启 agent） |
| KO-06 诚实认知状态 | 维持 L4 | grade 三态互斥测试 + gate.py 盲区声明 |
| KO-07 严格性递进 | 维持 L3 | _norm 测试族强证据；跨项目性低（偏实现层） |
| KO-08 重画优于修复 | 维持 L3 | draws/rounds 测试强证据；普遍性待验证 |

## 6. Counterexample Hunter（反例预算：每个 L3+ KO ≥3 定向攻击，0 反例须给搜索证据）
| KO | 反例攻击 | 结果 |
|---|---|---|
| KO-01 | ①browser-use 工具调用路线（Corpus）②坐标预测基线（README 自报更差）③插件版动作=宿主工具非脚本 | KO 收窄为"程序化动作子类" ✅ 反例已并入 |
| KO-02 | ①persistent_local_browser 需要持久②Browserbase 云会话③SKILL.md Firefox 每次重建 | "默认可丢弃、按需持久"分层模型 ✅ |
| KO-03 | ①shape 容忍漂移（live data）②unverified grade 存在③replay 仅覆盖训练实例 | "验证=重放"有边界；held-out 泛化未验证（C-04）✅ |
| KO-04 | ①learned_library 允许词汇表命中（窄判据）②manual credentials 允许存在（非零泄露面） | 判据经测试校准，非绝对纯净 ✅ |
| KO-05 | ①route 保留 agent fallback（未取消 agent）②catalog 全量注入的规模退化（C-02） | 前置是"可预计算部分"非全部 ✅ |
| KO-06 | ①self_verify 盲区（误信错答案）②verify off 产生 unverified 但库仍存在 | 系统自我声明盲区，诚实性成立 ✅ |
| KO-07 | ①strict 曾拒 AS26（spacing 抖动）②答案含日期波动 | 归一化=去噪非放松 ✅ |
| KO-08 | ①draws 全坏+rounds 全坏→不落地（预算下限）②~40% 首抽通过是单仓观测 | 收敛模型成立；比例跨仓待验证 ✅ |

## 7. Epistemic Auditor（认知状态标注是否诚实）
- 全部 L1/L2 知识标 Fact/Observation（带 file:symbol 证据）；L3/L4 标 Validated Pattern 且跨项目项挂 `Cross-project validation pending` ✅
- 未验证假设全部在 05 Candidates（7 条，类型+缺失证据+验证路径齐备），未冒充知识 ✅
- Hypothesis 未升格：C-01~C-07 无一条进入 03 层 ✅
- Contradictions 与 Counterexamples **保留**（见 §5/§6，未删除未弱化）✅

## 8. 本包自身缺陷声明
- benchmark 数字未独立复现（C-07）；MSR writeup/arXiv 论文未纳入（卡内四重核验提到但本轮未读取全文）
- docs/manual.md 之外的产品文档（reference.md/commands）未全量精读
- 插件四宿主行为一致性未实测（C-04）

## 9. Reconciliation（阶段⑥，应用独立审计修正）
> 独立审计见 `independent_validation_report.md`（独立 Auditor 盲重建，未修改原考古产物；本文件为此修正后版本）。

| # | 对象 | 修正 | 来源 | 状态 |
|---|---|---|---|---|
| R1 | 02 EK-17 | 补充"代码默认 require_self_reflection_success=False，base.yaml 配置 true；与 step_limit 同为配置覆盖代码默认模式" | 独立审计 §3.1（agents/default.py:39） | 已应用 |
| R2 | 02 EK-25 | 补充"webwright 主 solve 不写 agent_response.json（skill_factory 执行器才写）；trajectory exit message 是主场景答案来源，始终 STRING" | 独立审计 §3.2（learn.py:87-90 注释） | 已应用 |
| R3 | 03 KO-02 | 措辞收紧："默认记忆在文件系统；persistent/browserbase 显式会话是例外层" | 独立审计 §3.3 | 已应用 |
| R4 | 本文件 §2 Truth 表 | 增加 require_self_reflection_success 默认值行（代码 False/配置 True） | 独立审计 §3.1 | 已应用 |

- 修正类型：管道顺序/遗漏/scope 收紧；无 CONTRADICTED / DOWNGRADED / OVER_GENERALIZED 项；无 NEEDS_HUMAN_REVIEW 项。
- 原考古产物（ARCH-2026-10-05-001/package/）保持独立审计前原样，未修改。

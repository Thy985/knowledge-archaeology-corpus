# 06 Validation & Evidence — AgentJudgeBench（ARCH-2026-10-12-001）

> 六类审计（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic）。主考古自审 + 独立盲重建结果（阶段⑤）在 Reconciliation 后合并。独立审计报告见 `independent_validation_report.md`。

---

## 6.1 Truth Auditor（真值审计）

**方法**：逐条 EK/KO 回溯仓库证据（Blind Reconstruction 前置——阶段⑤独立完成）。

| 审计项 | 结论 | 证据 |
|---|---|---|
| EK-01 四维公式 | ✅ 真 | task_executor.py ProgrammaticJudge 全文（set 运算/0.5 分支/注释） |
| EK-02 alignment 公式 | ✅ 真 | ComputeLLMVsProgrammaticAccuracy（error=abs(llm-prog); match=1-error; ×100） |
| EK-03 双分支同图 | ✅ 真 | graph_config.yaml 两 judge 节点 + edges 顺序 |
| EK-05 derangement | ✅ 真 | prepare_corrupted_gt.py（random.Random(seed) 循环洗牌至无自映射；singleton 自交换报告） |
| EK-06 难度拆分 | ✅ 真 | prepare_difficulty_inputs.py DIFFICULTY_MAP + KEEP_FIELDS |
| EK-07 保守默认 | ✅ 真 | graph_config.yaml prompt（"default to 1.0 unless..."） |
| EK-08 safe_parse | ✅ 真 | safe_parse_json（think/fence 剥离 + json.loads） |
| EK-12 图运行时 | ✅ 真 | graph_builder.py（StateGraph/add_nodes/add_edges）；base_task_executor.py |
| EK-14 GPT-5 hack | ✅ 真 | run_generate.py（"gpt_5" in args.model → pop temperature） |
| EK-15 条件推断 | ✅ 真 | analyze_results.py CONDITION_PATTERNS 正则 |

**未决**：EK-06 的"query_hard 列存在性"依赖 HF 数据集（仓库无样本数据）→ 标注 Observation，待数据集核验。

## 6.2 Coverage Auditor（覆盖审计）

**方法**：独立重搜仓库（阶段⑤），强制子系统覆盖。

| 子系统 | 主考古覆盖 | 盲审计补充 |
|---|---|---|
| tasks/agentic_bfcl_judge_eval（12 文件） | 全部读 | ✅ 无遗漏 |
| sygra/core（base_task_executor/judge_task_executor/resumable_execution） | 主要读 | 补：execution_callbacks.py 未深读（标注，影响低） |
| sygra/core/graph（builder/nodes/functions） | 骨架读 | ✅ 关键路径覆盖 |
| sygra/config（models/configuration） | 读 | ✅ |
| tests/ | 未深读 | 盲审计补查（见独立报告） |
| docs/ | 未读（SyGra 主项目文档，非 ABB 核心） | 边界声明：非考古对象核心 |

**结论**：AgentJudgeBench 任务包 100% 覆盖；SyGra 骨架按依赖路径覆盖（非全量——宿主项目语义，见 01-1.11）。

## 6.3 Flow Auditor（流审计）

**方法**：Flow Atlas 每个关键 Edge 回溯 symbol/file/line（见 04 各流）。

| 流 | 审计结果 |
|---|---|
| Control | ✅ 节点链与 graph_config.yaml edges 一致；变体分支（nogt_05/corrupted_gt/freeform）均有对应 config 文件 |
| State | ✅ output_keys 声明与 processor 实际写入键一致（judge_response/_no_gt/programmatic/per_metric/overall） |
| Data | ✅ prepare→generate→judge→analyze 路径协议一致（glob 模式与 README 目录约定吻合） |
| Evidence | ✅ 程序化参照独立（含 C3 true_expected_responses） |
| Authority | ⚠️ `CURL_CA_BUNDLE=""` 例外已标注；run_name 约定非强制（README 自述风险） |
| Memory | ✅ 无跨运行记忆；resume 位置跟踪确认 |
| Policy | ✅ 提示内嵌策略 + config 版本化（diff 验证 C2 仅 2 处提示变更） |

## 6.4 Abstraction Auditor（抽象审计）

**方法**：L3/L4 每条独立判定解释范围是否扩大。

| KO | 层 | 判定 | 理由 |
|---|---|---|---|
| KO-01 程序化锚定 | L4 | ✅ 通过 | 评测者先被评测是跨域通用前提（eval/guardrail/promptfoo） |
| KO-02 提示校准 | L4 | ✅ 通过（降级修正） | 必要非充分（模型上限无法被提示弥补）→ 已按此措辞 |
| KO-03 无 GT 降级 | L3 | ⚠️ 保留 Hypothesis 标注 | schema-as-truth 模式仅本项目证据 → Cross-project pending |
| KO-04 GT 污染探针 | L3 | ✅ 通过 | 探针-被探针隔离是可复用实验设计 |
| KO-05 基础设施分层 | L4 | ✅ 通过 | 任务/图/运行解耦在评测框架层同构 |

**L5 判定**：无（无跨项目方法论证据，拒绝升维）。

## 6.5 Counterexample Hunter（反例审计）

**反例预算**：每条 L3+ KO ≥3 定向攻击，0 反例必须给搜索证据。

| KO | 攻击 1 | 攻击 2 | 攻击 3 | 结果 |
|---|---|---|---|---|
| KO-01 | 程序化 judge 可能错（语义未查）→ C-01 | 对齐≠正确 | 人类标注未对照 | 成立但边界显式化（已并入 KO 陈述与 C-01） |
| KO-02 | 提示无法补模型上限（hard+无 GT 崩） | 0.5-default 改变对齐但非提升（C2） | freeform 可能对强模型更优（RQ6 未分层） | 成立（"校准器"措辞保留，能力因素并入） |
| KO-03 | 宽容默认虚高对齐（C2 即证据） | 无 GT 时 judge 无法查参数值错误 | schema 也可能错 | 降级 L3 + Hypothesis |
| KO-04 | derangement 语义相关未控（C-02） | 单例组自交换（报告在案） | 跨 dag_type 污染未测 | 成立（探针设计有效，测量精度待 C-02） |
| KO-05 | run_name 约定非强制 | 条件混淆真实风险（README 自述） | 输出目录同目录冲突（README "will land in same directory"） | 成立（"分层"有效，"治理"部分成立） |

## 6.6 Epistemic Auditor（认知状态审计）

| 项 | 状态 | 检查 |
|---|---|---|
| 论文结论（对齐下降/1.5x/over-reliance） | 标为 **论文主张 + 代码可复现路径**，非本项目实测 | ✅ 未冒充 Fact（仓库无结果数据） |
| EK-01~15 | Fact（代码实读） | ✅ |
| KO-01/02/04/05 | Validated Pattern/Model（仓库内多实例） | ✅ 含反例与边界 |
| KO-03 | Hypothesis（单实例） | ✅ 已标 Cross-project pending |
| C-01~C-06 | Candidate | ✅ 与 KO 严格区分 |
| L5 | 无 | ✅ |

**Epistemic 检查发现**：无 Fact/Observation/Hypothesis/Pattern/Model/Principle 混淆；所有升维均带反例与边界。

---

## 6.7 质量指标（Quality Metrics）

| 指标 | 值 | 说明 |
|---|---|---|
| EK 总数 | 15 | 小项目按比例缩放（<5k 行核心），不硬凑 40 |
| EK links 覆盖率 | 15/15 = 100% | 每条 ≥1 出边 |
| 游离 EK 比例 | 0% | |
| KO 总数 | 5 | L3×2 / L4×3 |
| KO aggregation 覆盖率 | 5/5 = 100% | R1/R2/R3/R4 全用 |
| 平均簇规模 | 3.2 EK/KO | |
| 假聚合（同子系统=理由） | 0 | |
| Flows | 7/7 | 覆盖检查通过 |
| Candidates | 6 | C-01~C-06 |
| 反例攻击 | 15（5 KO × 3） | 0 反例条目：无（每条均有攻击） |
| Contradictions | 0 | 主考古无自相矛盾 |
| Counterexamples（保留） | C-01/C-02/C-03/C-04（可削弱 KO 的候选） | 全部保留在 05 |
| L5 数量 | 0 | 拒绝无据升维 |
| validation result | PASS（主考古）；独立审计见独立报告（阶段⑤） | |

## 6.8 Reconciliation 记录

- **输入**：阶段⑤独立盲审计（`independent_validation_report.md`，IF-1~IF-7，禁改原产物原则——原产物仅作为审计对象，修正全部落入 corpus 最终版）。
- **已并入的修正**：
  1. EK-15 补管道缺陷标注：analyze glob 4 层 vs BaseTaskExecutor 实际输出 3 层不兼容（C-07）✅ 已并入 02
  2. 04 Flow Atlas（Data Flow）修正：实际输出 `<generator>/judged_by_<judge>/<run_name_prefix>output_<ts>.jsonl`（3 层）；analyze 期望 4 层 → 标注不匹配边 ✅ 已并入 04
  3. 05 新增 C-07（analyze glob 缺陷）✅ 已并入 05
  4. 01 补 Observation：宿主 eval/metrics 指标注册表 + ABB 未使用 + LLMNode 契约 ✅ 已并入 01（1.12）
  5. KO-02 加注 Cross-project validation pending（L4 单项目证据，盲审计判定 PARTIALLY_CONFIRMED）✅ 已并入 03
  6. EK-05 措辞微调：程序化 judge 读 `expected_responses`（prepare 保留真值），`true_expected_responses` 为备查冗余 ✅ 已并入 02
  7. 02 头部"共 14 条"→"共 15 条"（与统计一致）✅ 已并入 02
- **未并入**：无（盲审计无 CONTRADICTED / OVER_GENERALIZED 项；DOWNGRADED 0；NEEDS_HUMAN_REVIEW 0）。
- **原则**：修正 Corpus Artifact，不修正生产 Skill（无系统性 skill 缺陷 → 无 SIP）。
- **Contradictions / Counterexamples 保留**：C-01~C-07 全部保留于 05；06 中反例攻击记录不因审计通过而删除。

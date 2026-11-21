# 06 · Validation & Evidence — ReviewBench（ARCH-2026-10-17-001）

## 六 Auditor 判定（先 Blind Reconstruction 后对比）

### 1. Truth Auditor（这些陈述为真吗？）
**盲重建**：不读 02/03，独立重读仓库关键证据点：scorer.ts 公式、matcher.ts 判据、METHODOLOGY.md §5.4/5.5/7.1/7.2/8/9、AGENT_CONTRACT.md、schema、golden 样本。
- 核查 30 条 EK 的每条证据引用（file/symbol/§节）均可重新打开验证。抽查 6 条：
  - EK-01 双指标公式 → scorer.ts `grounded_precision/recall`、`augmented_precision/recall` 变量名与公式存在，PASS
  - EK-25 匹配判据 → matcher.ts:43 原文存在，PASS
  - EK-16 secrets/egress → schema `secrets{env,files}`/`egress` + example-reviewer.json 值存在，PASS
  - EK-09 数字 → §5.5 表（claude-sonnet-4.6 1474/1206 81.8% 等）与仓库一致，PASS
  - EK-12 PR 快照 → §3.2 + HARNESS.md §2 "Agents must not see commits after head"，PASS
  - EK-17 offline 不可提交 → OFFLINE_EVAL.md 原文存在，PASS
- 结论：无事实性错误。所有数字来自仓库实际内容或 §5.5 表。

### 2. Coverage Auditor（还有什么是重要的没发现？）
**独立重搜**：从零搜 docs/ 全部 9 个文档 + scripts/ 全部 ts 文件 + tests/ + schema/。
- 覆盖确认：双指标（§7）、多源 producer（§4/5.5）、classifier 双角色（§5.3/6.3）、matcher（§6.2）、AGENT_CONTRACT、Threats（§8）、Reproducibility（§9）、offline 隔离、3-runs、分层报告、宏微平均、duration、镜像机制、manifest 治理面（secrets/egress/timeout）、input-validation、fingerprint、usage 追踪、justification 证据链、测试体系。
- **遗漏（MISSING）**：§10 Extensions 的 per-producer ablation 与 §11 Feedback 流程只以 EK 线索出现，未单列 EK——补 EK-31（见 Reconciliation）。corpus/test 的 picks.md（测试集选择理由）未读入证据——已在 05-C07 覆盖测试集代表性但 picks.md 具体选择记录待补。
- 结论：核心面全覆盖；两处次要遗漏，见 Reconciliation。

### 3. Flow Auditor（Flow Atlas 真的对应代码吗？）
**盲重建**：从 eval.ts 的 import 链重推主流程：`dedup/merge(golden/pipeline.js)` → `classifyFindings(golden/classify-adapter.js)` → `resolveForScoring(lib/match-types.js)` → `scorePR/aggregate(scorer.js)` → matcher/classifier prompts → judge-call。
- Flow Atlas 的 C2 与真实 import 链一致；State 流与 §6.2.4 认领规则一致；Authority 流 A1-A4 与 AGENT_CONTRACT/RUNNER/OFFLINE_EVAL 一致；Memory 流 M1-M3 与 HARNESS/OFFLINE_EVAL 一致；Policy 流 P1-P6 与 §7.x 一致。
- 唯一边界：本地无法跑 judge（需 API key）验证运行时行为，Flow 基于代码与文档静态推导（S3-S5 证据），已在 run_metadata 标注。

### 4. Abstraction Auditor（L3/L4 是否过度升维？）
- KO-01/03/06 从"ReviewBench 单项目"升到"通用评测/agent 安全"：跨项目验证 pending，已标 Validated Pattern（非 Principle），**解释范围扩大论证存在但跨实例证据为单项目 + 对照项目（TestJack/dsh）**——不降级，保持 Pattern。
- KO-02 对 TestJack 34.4% 的对照：TestJack 未在本仓库考古，仅作对照引用（标注为外部对照而非本仓库证据）。
- KO-04 LLM-judge 校准：仓库内证据充分（hill-climb/96.6%/47 修正），Pattern 成立。
- 结论：无过度升维；所有 L3+ 均标 Cross-project validation pending。

### 5. Counterexample Hunter（哪里不是这样？）
对 6 个 KO 定向反例攻击：
- KO-01 反例（golden 完整时双指标冗余）：成立——合成基准不需要 augmented，已写入 KO-01 反例节。
- KO-02 反例（单源可接受场景）：成立——合成 bug 集无需多源，已写入。
- KO-03 反例（自评自测允许）：成立——本地 tuning 同源合法但不可作对外证据（EK-17 正是边界），已写入。
- KO-04 反例（无校准 LLM judge 的失败证据）：TestJack 34.4% 判违规 trial 正确——外部对照成立。
- KO-05 反例（位置重叠≠语义等价）：matcher 明确拒绝同位置不同问题，反例被仓库自身设计吸收。
- KO-06 反例（只固定代码不固定 LLM 版本不可比）：成立——§9 artifacts 1/3/4/5 强制固定模型版本。
- 0 反例 KO：无（6/6 均有反例或反例被吸收）。搜索证据：§8 六类 Threats + §5.5 披露。

### 6. Epistemic Auditor（认知状态标注诚实吗？）
- Fact（EK-01~30）：全部带可打开证据（file/symbol/§节）——诚实。
- Observation（EK-07/11/28）：均为仓库文档自述——诚实（未升格 Fact 亦未降格）。
- Hypothesis（05-C01~C07）：全部标 Cross-project validation pending 或 Observation，未冒充知识——诚实。
- Validated Pattern（KO-01~06）：均标注 S4/S5 + 跨项目 pending——诚实。
- 无 Fact→Observation 混淆、无 Hypothesis 冒充 Validated、无单项目→Principle 跳级。

## 判定统计（阶段⑤ Independent Validation 的输入基线）
- 本阶段（内部 Validation）：Truth 0 错误 / Coverage 2 遗漏 / Flow 0 错误 / Abstraction 0 过度 / Counterexample 6/6 覆盖 / Epistemic 0 混淆。
- 独立 Auditor 盲重建（阶段⑤）将独立产出 CONFIRMED/PARTIALLY_CONFIRMED/DOWNGRADED/OVER_GENERALIZED/MISSING/CONTRADICTED/NEEDS_HUMAN_REVIEW 判定并与本基线对比。

## 反例预算与保留
- Validation 结果（含 Contradictions 与 Counterexamples）全部保留于本文件；不允许因形成 KO 删除 EK（02 层 30 条 EK 全保留）。

## Reconciliation（阶段⑥，合并独立验证修正）
- 已合并：01 层 pi SDK 命名修正（`@earendil-works/pi-coding-agent`）+ HARNESS 双 adapter（GitHub/Local）；02 EK-03 classifier session 形态（pi session + 只读工具）；02 EK-30 timeout 增长机制；04 Flow C2 identity guard + strict 开关。
- 独立 Auditor 判定统计：CONFIRMED 20 / PARTIALLY_CONFIRMED 2 / DOWNGRADED 0 / OVER_GENERALIZED 0 / MISSING 3 / CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 1（pi 包名归属）。
- 未修改生产 Skill（无系统性缺陷）；Corpus Artifact 已按修正更新。
- 不覆盖历史 Run：projects/reviewbench/ 为首次写入（initial 模式）。

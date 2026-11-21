# 03 Knowledge Layer — Transect（ARCH-2026-10-13-001）

> Generalized Knowledge（L3/L4）。5 个 KO，全部声明 aggregation_rule（R1-R4）与簇内 EK。升维铁律：单项目证据 → 标 `Cross-project validation pending`。

## KO 聚合规则矩阵

| KO | 层 | aggregation_rule | 簇内 EK | 解释范围扩大论证 |
|---|---|---|---|---|
| KO-01 | L4 Cognitive Model | R2 因果链簇 | EK-02→EK-03→EK-04→EK-05→EK-06→EK-07→EK-08 | 从"本项目判定管道"升维为"任何 LLM 判定依赖链都需要的校正闭环" |
| KO-02 | L4 Cognitive Model | R3 不变量簇 | EK-02, EK-04, EK-16, EK-07 | 溯源不变量：标签与源头解耦即无意义 |
| KO-03 | L3 Pattern | R1 机制簇 | EK-03, EK-04, EK-05, EK-06 | 三层校正模式跨两类判定面（phases/subagents）出现 |
| KO-04 | L3 Pattern | R4 主题簇 | EK-08, EK-05, EK-07, EK-11 | "自洽≠正确"主题：多个互补机制共同指向 |
| KO-05 | L4 Cognitive Model | R3 不变量簇 | EK-07, EK-08, EK-13, EK-11 | 可信度治理由显式策略常量承载（不变量=策略与执行分离） |

---

## KO-01 判定依赖链必须内建校正闭环（L4，R2 因果链簇）

**一句话**：任何"LLM 生成标签→被下游消费"的管道，可信度不是靠一个更好的 judge，而是靠 `digest 压缩 → 判定 → 投票共识 → 二轮复核 → 叙述不改判 → 可靠性度量` 的依赖链闭环；链条缺环 = 标签不可信且不可审计。

**簇内 EK**：EK-02 → EK-03 → EK-04 → EK-05 → EK-06 → EK-07 → EK-08（causal 边全链）

**论证**：Transect 把 judge 当作**不可信组件**（README："judged labels cannot be taken at face value either"），每个环节都是对上一环可信度的补偿——digest 决定判什么、投票收敛分歧、verifier 对高风险单元改写、narrator 不改判只叙述、reliability 显式度量并 flag。链条任一环缺失，报告仍会出，但 provenance/可靠性信息消失。

**反例压力**（counterexample-hunter 定向攻击）：
- C-1：OpenClaw 源锚点可为 None（无 event uuid）→ 溯源链在此断裂，报告仍渲染 → 链条是"尽量内建"而非"绝对保证"
- C-2：narrator 失败仅 fallback 计数（n_fallback），报告继续 → 叙述层是可降级环节
- 结论：模型表述为"校正闭环应作为依赖链设计"，不升为绝对定律

**状态**：Cognitive Model，`Cross-project validation pending`（跨项目验证：与 AgentJudgeBench 的 judge 侧审计形成对照候选，见 05 Candidates C-06）

---

## KO-02 判定可溯源性是不变量（L4，R3 不变量簇）

**一句话**：被引用的判定必须可回溯到产生它的源头（输入 digest 与产生回合）；溯源缺失 = 该标签只配作为 Hypothesis 展示。

**簇内 EK**：EK-02（digest 带 event_id）、EK-04（投票以 turn 为单元）、EK-16（anchor_event_id + judge-identity 列）、EK-07（LabelStats 按单元回溯）

**论证**：transcript 越长，人为重审越不可能；Transect 的选择是把"每条标签→源头回合"做成第一等数据结构（anchor_event_id + Scout deep link + frames 每行保留 transcript_id/turn），并用 judge-identity 列让 reliability 层能读到 regime。README "every label comes with its provenance"。

**反例压力**：
- C-1：solo regime 无 agreement/CI（None）→ 单判标签的"可靠性信息"是空的，报告仍标注 → 溯源与**可信度量**是两回事，本 KO 只管前者
- C-2：layer frame 不带 judge-identity 列 → detect_regime 抛 KeyError（拒绝而非静默）→ 违反即显式失败，印证其为不变量

**状态**：Cognitive Model，`Cross-project validation pending`

---

## KO-03 三层校正模式：投票收敛 → 复核改写 → 叙述不改判（L3，R1 机制簇）

**一句话**：对高风险判定面，通用做法是 `多数投票收敛分歧 → 对低置信/低一致/可疑单元做二轮复核（可改写，带 gate）→ 叙述层只描述不改判`——三层各自独立，避免"叙述者顺手改判"的隐性权威。

**簇内 EK**：EK-03（regime）、EK-04（投票）、EK-05（verifier）、EK-06（narrator）；mechanism 边：EK-04↔EK-05 共享"逐单元判定"机制

**论证**：同一模式在 phases（decision_phases 全链）与 subagents（subagent_classification 建于 cohort_llm_scanner，同一 reliability 契约）两个独立判定面重复出现；test_reliability `test_custom_judged_layer_conforms_to_the_reliability_contract` 证明自定义层也强制走同契约 → 跨实例性成立。

**反例压力**：
- C-1：cohort 时 verifier 强制 off（多数票即校正）→ 三层可裁剪为两层 → 模式是"可组合校正栈"而非固定三层
- C-2：solo 时 verifier 同模型二轮——"复核"与"首判"共享模型 → 校正独立性受限，报告中明示 caveat

**状态**：Pattern（跨两个内建判定面 + 契约化自定义层）

---

## KO-04 自洽性 ≠ 正确性（L3，R4 主题簇）

**一句话**：评测工具对"高度自洽"必须设防：k_roll 同模型重复自洽异常高，可能是采样/共享 prompt/cache 重放/标签名锚定，而非质量好；随机抽检出现推翻 = 直接红色警示。

**簇内 EK**：EK-08（flag 阈值含 KROLL_SUSPICIOUSLY_HIGH + spot-check overturn 必 red）、EK-05（random_sample 固定种子抽检）、EK-07（α/AC1 互不为 bounds）、EK-11（$0 结构与 cache 重放语义）

**论证**：`KROLL_SUSPICIOUSLY_HIGH` 是罕见的"对高分设防"阈值；cache 语义上 cached judge calls 不记 token（token_accounting 测试验证），意味着高一致可能含重放；same-model verifier caveat 明确"同模型二轮 ≠ 可重复性度量"。测试 `test_kroll_self_consistency_flags_low_and_suspicious_high` 双向覆盖。

**反例压力**：
- C-1：低自洽（<0.8）也只是 amber → 低自洽同样不自动否定（噪声样本本身可解释）
- C-2：α 高时无 flag（AGREEMENT_RELIABLE 以上不渲染）→ "高 judge 一致性"未设防（相关 judge 未必精确，见 Candidates C-03）

**状态**：Pattern（本项目强证据；跨项目验证 pending）

---

## KO-05 可信度治理由显式策略常量承载（L4，R3 不变量簇）

**一句话**：判定可信度的所有边界（什么算可信、什么必须复核、什么必然警示、置信低于多少不应用）应以集中常量 + 自包含 flag（metric/value/level/explanation/remediation）形式显式治理，而不是散落各处的 if。

**簇内 EK**：EK-07（_CI_MIN_N=8、wilson_interval）、EK-08（模块级阈值常量 + Flag 自包含）、EK-13（执行 vs 判定质量分层）、EK-11（cache 默认 True 成本策略）

**论证**：阈值（AGREEMENT_RELIABLE/AGREEMENT_TENTATIVE/KROLL_SUSPICIOUSLY_HIGH/LOWCONF/_SPOT_DEFAULT/_SAMPLE_MIN）全部模块级集中；flag 自带 explanation+remediation（"a flag must explain why crossing it matters"）；unjudged 必至少 amber、spot-check overturn 必 red 是无阈值硬规则。治理决策因此可被 code review / 测试直接攻击（test_reliability 逐带覆盖）。

**反例压力**：
- C-1：`_SPOT_DEFAULT=0.05` 与 `_SAMPLE_MIN=3` 是启发式 → 具体数值的"正确性"未经验证（Candidates C-02）
- C-2：verify_sample 可被用户关掉（verify=False）→ 治理可被显式降级（这是特性：成本-可信权衡开放给用户）

**状态**：Cognitive Model，`Cross-project validation pending`

---

## 计数

- KO：5（目标 7~12 按项目规模缩放：中等包 5 个合理）
- 聚合规则覆盖：5/5（R1×1, R2×1, R3×2, R4×1）；无"同子系统=聚合理由"假聚合
- 全部 KO 可回溯簇内 EK → EK 可回溯 Evidence（file:symbol）
- 无 Hypothesis 冒充 Fact：所有 L4 均标 `Cross-project validation pending`

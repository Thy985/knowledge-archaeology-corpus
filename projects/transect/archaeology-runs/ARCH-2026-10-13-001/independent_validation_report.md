# Independent Validation Report — Transect（ARCH-2026-10-13-001）

> **方法**：独立 Auditor 盲重建——不把考古包当事实来源，独立重读仓库（本轮盲重建抽查 10 项高风险声明，覆盖阈值常量/投票实现/spec 契约/抽样/依赖/报告结构），先建立 Independent Findings 再与考古包对比。**禁止修改原考古产物**（修正走阶段⑥ Reconciliation）。审计对象 commit：`75a7948257657f1c48b2fc1d2105fc2f6ab4074c`。

## 判定统计

| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 7 | V-1, V-2, V-3, V-4, V-5, V-6, V-10 |
| PARTIALLY_CONFIRMED | 1 | V-9 |
| CONTRADICTED | 1 | V-8 |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 1（并入 V-9 记录） | V-9b, V-9c |
| NEEDS_HUMAN_REVIEW | 0 | — |
| **合计** | **10** | |

## 3 个成功确认（Blind Findings 与包一致）

### V-1 CONFIRMED — 投票规则（modal → tie mean confidence → tie order）
独立实现：`cohort.py:vote_ballots()`（L422）`sides` 按 label 分组，`max(sides.items(), key=_rank_side)`，`_rank_side` 排序 → 模态胜出；平局按高 mean confidence；再平局按最早成员（ballot order 即 member order）。`agreement = len(side)/len(ballots)`，`confidence` 为模态侧 stated 值均值（单值保真），`confidence_pm = ci_half_width(stated)`。→ 考古包 EK-04 / KO-03 描述一致。

### V-2 CONFIRMED — spec 契约（extra="forbid" + plain 归一 + ops 保留）
独立实现：`spec.py` L13/30/45 三个模型 `ConfigDict(extra="forbid")`；L62 `_plain_labels_ok` 把 `"recon"` 归一为 `{"label": "recon"}`；L22 `ops: bool = False`（未声明则追加保留 "ops"）；L56 `extra: dict` 自由命名空间。→ 考古包 EK-12 一致。

### V-3 CONFIRMED — verifier 触发与抽样机制
独立实现：`phases_verify.py` `select_for_verify()`：4 触发（low_confidence=min_confidence<LOWCONF 0.60 / low_agreement<0.60（k_roll only）/ wedge（≤2-turn 夹层）/ random_sample）；`n_sample = max(_SAMPLE_MIN=3, round(_SPOT_DEFAULT=0.05 * len(phases)))`，`rng = random.Random(_SAMPLE_SEED=20260616)` 按 label 分层从未触发相中抽取；`VerifierAudit` 含 n_weak_relabel 等 9 计数。→ 考古包 EK-05 / 04-3 一致（"默认 max(5%,3)"语义一致）。

## 3 个错误/问题（Blind Findings 与包冲突或包遗漏）

### V-8 CONTRADICTED — cohort α 阈值带数值错误（考古包事实错误）
- **Blind Finding**：`report/reliability.py` L40-42 阈值常量 = `AGREEMENT_RELIABLE=0.80` / `AGREEMENT_TENTATIVE=0.66` / `KROLL_SUSPICIOUSLY_HIGH=0.95`；flag 逻辑 L174-188：`α < 0.66` → **red**；`0.66 ≤ α < 0.80` → amber；`α ≥ 0.80` → 无 flag。
- **考古包原述**：06_validation §6.4/03 KO-04 引"α <0.5 red / 0.5-0.8 amber"（该数值来自对测试名 `test_cohort_alpha_flag_bands` 的推断，未实读常量，错误）。
- **修正**：α < **0.66** red；0.66~0.80 amber（阶段⑥写入 corpus 修正版）。

### V-9 PARTIALLY_CONFIRMED + MISSING — flag 策略只部分入包
- **V-9 CONFIRMED 部分**："spot-check overturn 必 red"、"unjudged 至少 amber" 两条 thresholdless 规则与包一致（build_flags docstring）。
- **V-9b MISSING**：`RELABEL_RATE_HIGH = 0.20`（verifier 改写率 >1/5 examined → 高亮 flag）未入包。
- **V-9c MISSING**：`UNJUDGED_SHARE_SERIOUS = 0.25`（unjudged 占比 ≥1/4 → red，非仅 amber）未入包。
- **修正**：EK-08 与 06_validation 补两条阈值（阶段⑥）。

### V-10 CONFIRMED（补充修正）— snapshot 依赖版本缺一项
- **Blind Finding**：pyproject L13 `inspect-viz>=0.4.1`（考古包 snapshot 列了 inspect-ai/scout/pandas/pydantic 未列 inspect-viz 版本）。
- **修正**：snapshot 依赖表补 `inspect-viz>=0.4.1`（阶段⑥）。

## 遗漏（MISSING 汇总）

| # | 遗漏项 | 影响 |
|---|---|---|
| 1 | RELABEL_RATE_HIGH=0.20 阈值 | EK-08 flag 策略不完整 |
| 2 | UNJUDGED_SHARE_SERIOUS=0.25 阈值（unjudged ≥1/4 → red） | "unjudged 至少 amber" 描述不完整（存在 red 分支） |
| 3 | inspect-viz>=0.4.1 依赖版本 | snapshot 依赖表缺一项 |

## Benchmark Case（跨项目对照：AgentJudgeBench，ARCH-2026-10-12-001）

| 维度 | AgentJudgeBench（昨日） | Transect（本轮） |
|---|---|---|
| 可信度位置 | judge 侧：判官对齐/偏差审计 | 评测者侧：标签可观测/可审计 |
| 核心机制 | 判官行为审计协议 | 判定三层校正 + provenance + reliability audit |
| 互补性 | 回答"评测者是否公正" | 回答"评测结论能否被信任与复查" |
| 合并假说 | C-06：成熟 agent 评测管道应同时具备 judge 审计层 + 结果可观测层（评测可信度双环） | |

Benchmark 学习价值：两轮考古合起来验证了"评测工具的可信度设计"是 Agent-AI Engineering 的高价值主题簇；Transect 的"对高自洽度设防"（KROLL_SUSPICIOUSLY_HIGH=0.95）与 AgentJudgeBench 的"判官自相关"发现指向同一认知：**一致性指标单独不足以支撑可信度**——跨项目一致信号，值得在 corpus 层标注为 Cross-project Candidate（C-06）。

## Auditor 结论

- 核心机制层（投票/verifier/regime/reliability 数据模型/报告结构/选择规则）**全部 CONFIRMED**，无过度泛化、无 Epistemic 混淆（Archaeology Package 的 Fact/Observation/Hypothesis 标注抽查通过）。
- 1 处事实错误（α 阈值带数值）+ 2 处遗漏（flag 阈值常量）+ 1 处补充（依赖版本）→ 全部可在 Reconciliation 阶段修正 Corpus Artifact；**不构成对生产 Skill 的系统性缺陷**（SIP = none）。
- 原考古产物未做任何修改（本报告独立成文）。

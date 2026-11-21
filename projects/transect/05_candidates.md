# 05 Candidates — Transect（ARCH-2026-10-13-001）

> 不能确认的内容留在此处，严格标记 Hypothesis / 暂定模式 / 跨项目假说。不冒充已验证知识。

## C-01 标签名锚定假说（label-name anchoring）
- **状态**：Hypothesis
- **内容**：k_roll 高自洽可能含"标签名锚定"——judge 因标签名而非行为证据趋同（flag remediation 建议做 label-name sensitivity check，但 Transect 未内置该实验）。
- **当前证据**：`report/reliability.py` KROLL_SUSPICIOUSLY_HIGH remediation 文本；spec label 描述进 judge prompt。
- **缺失证据**：仓库无改名对照实验；需在真实数据集上 A/B 标签名才能验证。
- **验证路径**：对同一 digest 集用同义标签名重复 k_roll，比较自洽率与标签分布偏移。

## C-02 verifier 阈值 0.6 的通用性
- **状态**：暂定模式
- **内容**：verifier overturn gate（confidence≥0.6 才应用改写）与 random_sample 默认 5% 是启发式常量；对长时程评测（论文 1300 万 token 案例）的有效性未在仓库内基准化。
- **当前证据**：`cohort.py:LOWCONF=0.60`、`_SPOT_DEFAULT=0.05`、`_SAMPLE_MIN=3`；无阈值敏感性测试。
- **缺失证据**：不同评测族（代码/研究/客服）下 0.6 gate 的误改/漏改率。
- **验证路径**：构造带 ground-truth 的评测日志集，扫描 gate 阈值。

## C-03 高 judge 一致性未设防（相关 judge ≠ 精确）
- **状态**：Hypothesis（跨项目）
- **内容**：cohort α ≥ AGREEMENT_RELIABLE 时不渲染 flag——高一致性可能来自相关 judge（同系模型/同 prompt 家族）而非高精度，但 Transect 只在 remediation 文本提示 "More correlated judges need not improve precision"。
- **当前证据**：`report/reliability.py` cohort 分支 α≥0.8 无 flag；remediation 文案。
- **缺失证据**：需外部真值比较（judge 一致 vs 判定准确）才能把该提示升级为 flag。
- **验证路径**：跨项目（含 AgentJudgeBench 的 judge 对齐结果）考察一致性-精度相关性。

## C-04 OpenClaw 语义桥接差异
- **状态**：Observation（边界）
- **内容**：OpenClaw 导入的 span 结束是**合成**的（ends_recorded=False），与 Inspect 原生结束语义不同；scaffold 消息靠精确字符串匹配跳过（脆弱性未知）。
- **当前证据**：`scanners/base.py:token_timeline()` ends_recorded 分支；`_OPENCLAW_SCAFFOLD_USER_MESSAGES` 常量。
- **缺失证据**：真实 OpenClaw 长程会话中 scaffold 消息变体覆盖度；合成 span 对 phase 边界的偏差。
- **验证路径**：用多供应商 scaffold 模板集跑导入回归。

## C-05 报告"phase 正确性"未知（narration 状态 ≠ 事实正确）
- **状态**：Observation
- **内容**：`narration_group_status` 只描述坐标完整性（complete/invalid_partition/empty_groups/no_narrative/not_run），报告从未声称 phase 分割"正确"——这是诚实边界，但也意味着 Transect 不验证自己标签的 ground truth。
- **当前证据**：`phases.py` narration 字段文档；phases_narrate.py。
- **缺失证据**：与人工标注或 ground-truth 的 phase 分割对照。
- **验证路径**：基准评测集人工标注 phase 边界，对比自动分割（跨项目候选）。

## C-06 跨项目假说：判定可靠性度量应成为评测管道标准层（与 AgentJudgeBench 对照）
- **状态**：Cross-project Hypothesis
- **内容**：昨日（ARCH-2026-10-12-001）AgentJudgeBench 在 judge 侧解决"如何审计评测者"（判官对齐/偏差），Transect 在评测者侧解决"如何让评测结论可观测可审计"（标签溯源+可靠性）。合并假说：成熟的 agent 评测管道应同时具备 judge 审计层 + 结果可观测层，二者是"评测可信度双环"。
- **当前证据**：Transect reliability/report 层；AgentJudgeBench 考古包（corpus projects/agentjudge-bench/）。
- **缺失证据**：一个同时部署两层并展示端到端收益的真实评测项目。
- **验证路径**：未来对某开源评测工具（含 LLM judge 的）同时做 judge 对齐审计与 Transect 式可观测化，比较可信度收益。

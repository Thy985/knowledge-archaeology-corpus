# 05 · Candidates（未验证/跨项目假说）— ReviewBench（ARCH-2026-10-17-001）

> Hypothesis 不进知识层，留在候选。每条标注：类型 / 当前证据 / 缺失证据 / 验证路径。

## C-01 跨项目：grounded/augmented 双指标可迁移到自建评估集（L3 模式假设）
- 当前证据：ReviewBench 单项目完整实现（EK-01/02）；引擎 evaluation 层"程序化断言优先、LLM-judge 次级信号"方法论对齐。
- 缺失证据：第二个评估系统采用双指标并产生有效结果。
- 验证路径：在引擎 eval 层或新评估集上落地双指标并对比单一 F1 的偏差可见性。
- **Epistemic**: Hypothesis（Cross-project validation pending）

## C-02 离线/在线收敛性未验证（L2 观察→L3 假设）
- 当前证据：§8 明确"Whether offline and online benchmarks converge is an empirical question"；离线分数=classifier 对人类反应的代理。
- 缺失证据：真实 PR 上 agent review 被人类采纳/点赞/解决率 vs 离线 TP 标签的一致性与偏差方向。
- 验证路径：对 matched sample 采集 comment reactions/resolution rates，与 offline labels 对比并回喂（§8 建议）。
- **Epistemic**: Hypothesis

## C-03 producer-candidate 同源加分未被量化（L2 观察）
- 当前证据：§5.5 "A reviewer… may benefit from overlap with its own earlier output"——披露但未量化。
- 缺失证据：leaderboard 上 candidate 与 golden producer 同模型的系统性加分幅度。
- 验证路径：per-producer ablation（§10 建议）——去掉某 producer 重建 golden，重跑同模型 candidate 对比分数差。
- **Epistemic**: Hypothesis

## C-04 classifier 文化依赖的跨团队差异（L2 观察）
- 当前证据：§8 "another team running this methodology could configure the classifier differently and reach different scores for the same agents"。
- 缺失证据：不同团队配置下同 agent 分数差异的实际分布。
- 验证路径：多团队复跑同一 agent（§9 artifacts 固定下），比较分数散布。
- **Epistemic**: Hypothesis

## C-05 chunking 边界效应（L1 观察）
- 当前证据：§6.2.1 chunking "does not affect the matching semantics"；但跨 chunk 的 finding 对（跨文件/跨 chunk 同 issue）匹配行为未测。
- 缺失证据：chunk 边界处漏配率。
- 验证路径：对 chunk 边界样本做人工匹配复核，统计漏配。
- **Epistemic**: Observation

## C-06 augmented 指标的裁判边界（L2 观察）
- 当前证据：classifier 判 unmatched TP 的边界与 golden 的边界相同（同 classifier），但 unmatched 无 human audit 兜底（§6.3 vs §5.4 的 47 修正）。
- 缺失证据：unmatched 的 classifier TP 中"若经 human audit 会被改判"的比例。
- 验证路径：对 novel TPs 抽样 human audit，报告修正率并评估 augmented 指标偏差。
- **Epistemic**: Hypothesis

## C-07 25 PR 测试集的统计功效（L2 观察）
- 当前证据：test set 25 PRs 代表性设计（EK-13）；但 25 PR × 3 runs 的差异是否足以支撑严格统计推断未论证。
- 缺失证据：run-to-run 方差与 PR 间方差的功效分析。
- 验证路径：用现有 leaderboard 数据计算效应量/功效，评估排序稳定性。
- **Epistemic**: Hypothesis

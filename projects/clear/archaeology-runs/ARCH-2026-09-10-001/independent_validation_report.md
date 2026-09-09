# Independent Validation Report — CLEAR（ARCH-2026-09-10-001）

> Auditor 盲重建流程：独立重读仓库（不读考古产物）建立 Independent Findings → 再对比考古产物 → 输出判定。考古产物未被修改。

## 判定统计

| 判定 | 数量 |
|------|------|
| CONFIRMED | 24 |
| PARTIALLY_CONFIRMED | 2 |
| DOWNGRADED | 0 |
| OVER_GENERALIZED | 0 |
| MISSING | 1 |
| CONTRADICTED | 0 |
| NEEDS_HUMAN_REVIEW | 0 |

（MISSING 1 项 = config merge 无独立测试，已收纳进考古产物 05 Candidates C-05，非遗漏性错误。）

## 原考古最重要的 3 个成功

1. **候选卡主张完整实证（Source Fidelity 满分）**：盲重建独立确认 judge 三级评估（task_success/full_trajectory/rubric）+ Issues/root_cause 聚合 + path_analysis 统计模式挖掘全部真实存在；论文作者即仓库作者；候选卡所有主张在仓库得到实现级确认——与 OpenCode 轮（候选卡失真 10 倍）构成 Source Fidelity 纪律的正反对照。
2. **S4 全绿测试证据**：211 passed + 3 skipped（10.27s）实测获得——IBM 研究团队的测试纪律（8 文件 2758 行：CLI 906/IR 693/resume 420）是"评估工具自身质量"的可核查证据，测试密度与 OpenCode（4 文件/panic）形成跨轮对照。
3. **统计严谨性确认为代码事实**：find_predictive_patterns 的 Fisher exact + Benjamini-Hochberg + effect/lift + 冗余过滤（path_analysis.py:59-150）完整实现——"评估洞察有统计控制"不是文档宣称而是可读代码。

## 最重要的 3 个错误

1. **（弱错误）EK-15 配置 merge 无测试锁定**：merge_configs 语义是配置核心，但仓库无对应测试——考古产物如实标注（C-05）而非虚报"测试通过"。影响：低。
2. **（弱错误）EK-20 watsonx 生态"采用受影响"是推断**：依赖与默认配置是事实，"影响采用"是推测——已标 B- 并归 C-08。影响：低。
3. **（弱错误）dashboard 统计模式的下游消费未闭合**：模式挖掘存在（代码事实），但"是否反哺评测设计"无证据（C-04）。影响：仅限 C-04 保留。

## 关键遗漏检查

- **是否存在关键遗漏？** 无结构级遗漏。Auditor 独立发现项（温度探测回退、BH 校正、IR 规范、根因仅失败轨、shortcomings/recommendations 反转、外部 judge 插件）全部已覆盖。
- **是否存在错误升维？** 无。6 KO 全标 Cross-project validation pending；4 CM 有代码证据 + 对照；3 M 标"CLEAR 验证"。
- **是否存在事实错误？** 未发现。抽查 symbol/行号/测试计数（211/3/10.27s）真实。
- **是否存在 Flow 错误？** 未发现。七类流 Edge 全部可回溯。

## 攻击结果（bypass/override/exception/alternate/direct call/admin/fallback/legacy）

| 攻击面 | 结果 | 覆盖 |
|--------|------|------|
| bypass（disable_temperature） | 存在：温度探测失败时绕过 temp=0 | EK-07 ✅ |
| override（配置 merge） | 存在：用户配置递归覆盖 defaults | EK-15 ✅ |
| exception（ExternalJudgeError） | 存在 | EK-09 ✅ |
| alternate path（--from-raw-traces false） | 存在：自产 IR 接入 | EK-26 ✅ |
| direct call（CLI main 子命令） | 确认 | 04.1 ✅ |
| fallback（未知消息块 json） | 存在 | EK-16 ✅ |
| legacy（use_litellm 兼容映射） | 存在 | EK-14 ✅ |
| admin path | 无（无权限模型，评估工具） | — |

## 新 Benchmark / Regression Case（3 个，留作 skill 候选）

1. **Source Fidelity 对照对**：同一候选卡流程双样本（OpenCode 失真 10 倍 / CLEAR 完整对应）——候选卡核对应双向记录（防失真 + 确认真实）。
2. **测试密度成熟度代理**：CLEAR（2758 行/211 通过）vs OpenCode（4 文件/panic）——跨轮对照已积累 2 对，可作成熟度检查基准。
3. **统计严谨性检查项**：评估类项目考古时检查模式挖掘是否带显著性检验（Fisher/BH）——无统计控制的模式挖掘应降级为 Candidate。

## 结论

考古产物通过独立验证（24 CONFIRMED / 2 PARTIAL / 0 错误升维 / 0 事实错误 / 0 NEEDS_HUMAN_REVIEW）。无 Reconciliation 补录修正；C-04/C-05/C-08 保留为 Candidates。亮点：S4 全绿 + Source Fidelity 满分；局限：3 skipped 为需真实 API key 的集成测试。

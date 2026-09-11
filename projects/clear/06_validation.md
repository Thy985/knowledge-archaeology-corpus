# 06 · Validation & Evidence — CLEAR

> 独立 Auditor 盲重建：先独立重读仓库（不读考古产物）建立 Independent Findings，再对比。本文件记录验证结论与质量指标。

## 06.0 验证方法说明

- **运行环境**：Python 3.12.11 / pip 25.0.1；`pip install -e .` 成功（依赖：langchain/langgraph/ibm_watsonx_ai/pandas/pyarrow/streamlit 等）。
- **测试运行（S4 级，本 run 实测）**：`python3 -m pytest tests/ -q` → **211 passed + 3 skipped in 10.27s**。
- **盲重建**：Auditor 独立重读仓库（README/pyproject/结构/核心模块/测试/文档五份），实测安装与测试，再与考古产物比对。

## 06.1 判定统计（独立 Auditor）

| 判定 | 数量 | 对象 |
|------|------|------|
| CONFIRMED | 24 | EK-01~14, 16~21, 23~27（代码/结构/测试直接支持） |
| PARTIALLY_CONFIRMED | 2 | EK-15（配置 merge 实现确认，但无独立测试——见 C-05）、EK-22（自定义标准入口确认，full_trace 自定义维度仅 docstring 支持） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 1 | config merge 无测试（已收纳 C-05） |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 06.2 独立 Auditor 的成功发现（Top 3）

1. **候选卡主张完整实证**：Agentic CLEAR 卡声称"judge 三级 + 动态失败模式挖掘"——盲重建独立确认 task_success/full_trajectory/rubric 三级评估 + Issues/root_cause 聚合 + path_analysis 统计模式挖掘全部真实存在，且论文作者即仓库作者。Source Fidelity 满分（对照 OpenCode 轮失真 10 倍）。
2. **S4 全绿证据**：211 测试实测全过（10.27s）——本项目有真实且健康的测试体系；测试覆盖 CLI 参数（906 行）、IR 序列化（693 行）、断点续跑（420 行）、聚合输出、统计 dashboard。测试密度与 OpenCode（4 文件/panic）形成强烈对照。
3. **统计严谨性独立确认**：Auditor 独立读 path_analysis.py 确认 Fisher exact + Benjamini-Hochberg + effect/lift + 冗余过滤完整实现——不是文档宣称而是代码事实（59-150 行）。

## 06.3 关键错误 / 遗漏 / 攻击结果

### 考古产物 3 个最弱处（Auditor 视角）
1. **EK-15 配置 merge 无测试锁定**：merge_configs 语义（递归覆盖）是配置核心，但仓库无对应测试——考古产物如实标注（C-05），未虚报为"测试通过"。
2. **EK-20 watsonx 默认生态的影响是推断**：dependencies 与 default_config 是事实，但"采用受影响"是推断——已标 B- 并归 C-08。
3. **dashboard 统计模式的消费方式未闭合**：find_predictive_patterns 存在（代码事实），但"模式如何被反哺评测设计"无证据（C-04）。

### 攻击项
- **单案例→Pattern**：KO-01~06 全部标 Cross-project validation pending ✅
- **Pattern→L4**：4 CM 均有代码证据 + 跨项目对照，未写成 Law ✅
- **项目经验→通用 Principle**：M-01~03 标"CLEAR 验证" ✅
- **ADR→实现事实**：仓库无 ADR；README/文档自述（S2）未当实现行为断言 ✅
- **Flow Edge 真实性**：抽查 run_trajectory_evaluation_pipeline.py:526、full_pipeline.py:177/263、llm_client.py:97、config_loader.py:63、task_success_evaluator.py:118 全部真实 ✅
- **bypass/override/exception/alternate/direct call**：
  - bypass：temperature=0 被 disable_temperature 绕过（探测失败路径）✅ EK-07 覆盖
  - override：用户配置递归覆盖 defaults ✅ EK-15 覆盖
  - exception：ExternalJudgeError（external_judge.py:20）✅ EK-09 覆盖
  - alternate path：--from-raw-traces false（自产 IR 接入）✅ EK-26 覆盖
  - direct call：CLI main → 各子命令 ✅ 04.1 覆盖
  - fallback：未知消息块 json 序列化 fallback（trace_utils）✅ EK-16 覆盖
  - legacy：use_litellm 向后兼容映射 ✅ EK-14 覆盖
- **Epistemic 状态**：Fact↔EK↔KO↔CM↔M 分层无混淆；Hypothesis 全部进 05 ✅

## 06.4 关键遗漏检查

- **是否遗漏重要 Engineering Knowledge？** 无结构级遗漏。Auditor 独立发现项（温度探测、BH 校正、IR 规范、根因仅失败轨、shortcomings/recommendations 反转）全部已覆盖。
- **是否错误升维？** 无。
- **是否存在事实错误？** 未发现。抽查 symbol/行号/测试计数（211/3）真实。
- **是否存在 Flow 错误？** 未发现。
- **是否发现新的 Benchmark / Regression Case？** 3 个：
  1. **Source Fidelity 对照对**：同一候选卡流程，OpenCode 失真 10 倍 vs CLEAR 完全对应——"候选卡 → 仓库核对"基准应双向记录（既防失真也确认真实）。
  2. **测试密度作为成熟度代理**：CLEAR（2758 测试行/211 通过）vs OpenCode（4 文件/panic）——跨轮对照样本已积累 2 对。
  3. **统计严谨性检查项**：评估类项目考古时检查模式挖掘是否带显著性检验（Fisher/BH）——无统计控制的模式挖掘应降级为 Candidate。

## 06.5 质量指标

| 指标 | 值 |
|------|-----|
| EK 总数 | 27 |
| EK 有 links | 27（100%） |
| 游离 EK | 0（<20% ✅） |
| 平均出边 | ~2.0（≥1 ✅） |
| KO 聚合规则覆盖率 | 100%（6/6） |
| KO 平均簇规模 | 4.5（3~12 ✅） |
| 假聚合（同子系统=理由） | 0 |
| KO/CM/M | 6/4/3 |
| Candidates | 8 |
| 证据等级分布 | S3×21 + S4×6（211 测试全过） |
| 测试运行 | **211 passed + 3 skipped（10.27s）** |
| 交叉校验（Flow→KO） | 6/6 通过 |
| 单案例→Pattern 越级 | 0 |
| Hypothesis 冒充 Fact | 0 |

## 06.6 Reconciliation 记录

- Auditor 独立发现与考古产物一致，无补录修正。
- 保留项：C-04（dashboard 模式消费）、C-05（config merge 无测试）、C-08（IBM 生态影响）→ 待人工审查/后续验证。

## 06.7 验证结论

**考古产物通过验证**（Truth / Coverage / Causality / Flow / Abstraction / Counterexample / Epistemic 七项均通过；0 CONTRADICTED / 0 OVER_GENERALIZED）。**亮点**：S4 全绿（211 passed）+ Source Fidelity 满分（候选卡主张完整实证）——连续两轮验证方法论对照：OpenCode（失真→修正）与 CLEAR（真实→确认）都是 Source Fidelity 纪律的价值证明。**局限**：3 skipped 为需真实 API key 的集成测试（温度探测端到端）；dashboard 模式消费、config merge 测试待后续。

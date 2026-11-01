# ReviewBench — Knowledge Archaeology Index

## 一句话定位
GitHub 官方（Copilot 团队，2026-10-05 发布）的 **AI 代码审查离线评测基准**：219 个真实 PR（25 个代表性测试集）的 golden findings 为 ground truth，以 grounded/augmented 双指标衡量候选审查 agent 的 precision & recall。

## 考古结论（核心）
- **评测可信度 = 标签来源分层**：grounded（人类验证 golden 标签）vs augmented（classifier 补充判定）双指标，显式承认 golden set 非穷尽（KO-01）
- **反偏差是基准可信度大前提**：多源 producer（human/ccr/LLM/tool）+ 分布抽样 + Threats to Validity 显式披露（KO-02）
- **评测者与参赛者权威分离**：fresh container per PR、镜像 digest pin、offline 结果不可提交 leaderboard、3-runs 均值（KO-03）
- **LLM-judge 必须校准**：classifier hill-climb human dev set + 96.6% TP/FP agreement + 47 findings 人工修正（KO-04）

## 产物清单
| 目录 | 内容 |
|---|---|
| 00-knowledge-inventory/ | Overview（三条主线 + 质量快照） |
| 01-project-layer/ | 项目地图（架构/模块/数据结构/生命周期/双 adapter） |
| 02-engineering-knowledge/ | EK Graph（30 条 EK，六类边） |
| 03-knowledge-layer/ | Generalized KO（6 个，R1-R4 聚合规则） |
| 04-flow-atlas/ | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05-candidates/ | 7 个未验证假说（含跨项目） |
| 06-validation/ | Validation 报告 + 独立审计 + Reconciliation |
| archaeology-runs/ARCH-2026-10-17-001/ | run 级产物（manifest/snapshot/metadata） |

## 考古日期/版本
- 考古日期：2026-10-17
- 目标 commit：`6e37127defb2da675d4b59da61cf38d12ded1ad8`（main, 2026-10-16）
- Skill 版本：knowledge-archaeology v3.2
- Run ID：ARCH-2026-10-17-001

## 状态
- initial 考古（projects/ 首次收录 reviewbench）
- 独立验证：CONFIRMED 20 / PARTIALLY_CONFIRMED 2 / MISSING 3 / NEEDS_HUMAN_REVIEW 1 / CONTRADICTED 0
- 未产生 Skill Improvement Proposal（无系统性 skill 缺陷）

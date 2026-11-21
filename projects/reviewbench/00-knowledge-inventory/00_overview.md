# 00 · Overview — ReviewBench（ARCH-2026-10-17-001）

## 项目一句话
ReviewBench（GitHub 官方 2026-10-05 发布）是一个**可复现的 AI 代码审查离线评测基准**：以 219 个真实 PR（25 个代表性测试集）为基础，人类审核过的 golden findings 为 ground truth，用 grounded/augmented 双指标衡量候选审查 agent 发现有用问题与避免误报的能力。

## 三条主线
1. **评测可信度的"标签来源分离"**——grounded 指标只用人类验证过的 golden 标签，augmented 指标纳入 classifier 对 unmatched findings 的独立判定；"golden set 非穷尽"被显式承认并以 augmented 指标补偿。
2. **全链路防自我验证纪律**——fresh container per PR、镜像 digest pin、agent 不知自己在 benchmark 中、offline 结果不可提交 leaderboard、三次重复运行取均值；评测者（organizer/classifier）与参赛者（agent）权威分离。
3. **可复现 = 全链路 artifact 固定**——§9 八项 reproducibility artifacts + classifier/matcher prompt 公开 + checkpoint fingerprint + 仓库镜像防上游删除；Threats to Validity 六类限制显式披露而非弱化。

## 与已有 Corpus 的连接
- **pullfrog**（corpus 已考古，code-review 工具）→ ReviewBench 是其"基准+工具"配对的评测标尺
- **Validation 编译器 / 引擎 evaluation 层** → judge 校准约束（程序化断言优先、LLM-judge 次级信号）、可回放测试、证据链数据模型直接对应
- **deepseek-harness**（validated）→ 运行时隔离（sandbox）vs 本基准的 eval 层隔离（fresh container per PR）对照
- **TestJack/AgentJudgeBench/Transect**（10-16 三环验证，维持观察）→ ReviewBench 是"golden set + 统一 rubric + 独立验证"三环的工程化实现

## 质量快照
EK 30（六类边）/ KO 6 / Candidates 7 / Flow Edge 24 / 反例覆盖 6/6 / 盲重建独立执行 / 判定统计见 06。

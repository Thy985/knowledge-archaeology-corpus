# Job Manifest · ARCH-2026-10-14-001

- **date**: 2026-10-14（Evaluation 周 Day3）
- **job_id**: ARCH-2026-10-14-001
- **mode**: initial

## 候选排序（不按 star，按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 可考古性）

| 序 | 候选 | 状态 | 理由 | 决策 |
|---|---|---|---|---|
| P0 | **ReviewBench**（GitHub 官方 AI 代码审查离线基准） | NEW（10-14 新卡，A 级） | ①评测可信度方法论三环的**第三环**：数据分布建模→多源 golden set→统一 rubric→独立验证→校准 grader→版本化评估——"自建基准方法论"缺口的高质量范例（能力树 E 层）；②与 10-12 AgentJudgeBench（judge 审计层）、10-13 Transect（结果可观测层）互补成**完整评测可信度栈**；③代码审查 Agent = Agent-AI Engineering 直接应用场景；④仓库可得（review-bench/ReviewBench，TypeScript/MIT/50★，10-13 仍活跃 push）；⑤10-05 发布窗口内，增长信号明确 | **选中（initial）** |
| P1 | 求索 4.0（全产业链评测基准，国标体系） | NEW（10-14 新卡，A 级） | 知识价值高（国家级标准基准体系方法论），但**无公开代码仓库**（候选卡自述"待定位仓库"），不可做代码考古 | 不选，维持观察（若轻量版工具开源再做） |
| P2 | Transect | COMPLETED（10-13 已考古） | 已考古（ARCH-2026-10-13-001，PR #42） | 跳过 |
| P3 | AgentJudgeBench | COMPLETED（10-12 已考古） | 已考古（ARCH-2026-10-12-001，PR #41） | 跳过 |
| P4 | promptfoo | VALIDATED 锚点 | 已在本机实测验证（索引 validated） | 跳过 |

## 选中项目

- **project**: ReviewBench
- **repository**: https://github.com/review-bench/ReviewBench.git（默认分支 main，TypeScript，MIT）
- **source_entry**: `03_expansion_queue/candidates/[cand]reviewbench-2026.md`（2026-10-14 建卡）+ `05_scan_logs/2026-10/2026-10-14_radar-watch_scan.md`（第 42 次扫描）+ GitHub Blog 2026-10-05 官方文章
- **priority**: P0（high）
- **reason**（同排序表）:
  1. 基准构造方法论的可考古范例——103.9M PR 分布建模、219 PR/19 语言、多源 golden set（人类+LLM+静态分析）、语义去重、统一 rubric（Claude Sonnet 5 校准 grader）、96.6% 独立复标一致性、版本化评估（dataset/judge/matcher 同版本）
  2. 评测可信度三环闭环：judge 审计（AgentJudgeBench）→ 结果可观测（Transect）→ **基准本身构造可信**（ReviewBench）
  3. 代码审查 Agent 评测 = 与 Agent-AI Engineering 最直接相关的评测领域
  4. 仓库可得性实证：GitHub API 命中 `review-bench/ReviewBench`，created 2026-09-04、pushed 2026-10-13（前一日仍有提交）、5 open issues、8 forks、MIT、homepage review-bench.ai

## 验证计划（阶段③④）

1. 浅克隆 `https://github.com/review-bench/ReviewBench.git` 到 `archaeology-jobs/ARCH-2026-10-14-001/repo`
2. 记录 commit SHA / 分支 / 时间戳 + 项目基础地图
3. 完整考古（v3.2 六阶段：Discovery → EK Graph → Knowledge Layer → Flow Atlas → Candidates → Validation）
4. 独立 Auditor 盲重建 → Reconciliation → corpus `projects/reviewbench/` → PR

# Job Manifest — 2026-10-17（Daily Knowledge Archaeology Runtime）

## 候选排序（按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 未考古 / Benchmark 学习价值；不按 star）
| # | 项目 | 状态 | 排序理由 |
|---|---|---|---|
| P0 | **ReviewBench**（review-bench/ReviewBench） | NEW | GitHub 官方 10-05 发布的 AI 代码审查离线基准；公开仓库活跃（2026-10-16 push）；方法论（1 亿+ 真实 PR 分布建模 → 多源 golden set → 统一 rubric → 资深工程师独立验证）直接命中能力树 E 层"自建基准方法论"缺口；与 Validation 编译器（rubric 一致性）、pullfrog（code-review 工具，corpus 已考古）形成"基准+工具"配对；★59（小 star 不构成排除——按学习价值而非 star 排序）|
| P1 | 求索 4.0（国标评测体系） | NEW | 国家级标准评测体系（央视 10-13 发布）；**排除**：GitHub 公开仓库 0 命中（轻量版代码工具未以独立开源仓库形式存在） |
| P2 | HAAF（Holographic Agent Assessment） | NEW | 评测方法论 A 级；**排除**：仅 1★ 实验 pilot 仓库（TonyQJH/haaf-pilot），非生产项目 |
| P3 | TestJack / Transect / AgentJudgeBench | QUEUED | 10-16 已核验：无官方公开仓库，维持观察（三环验证完成） |
| P4 | ToFu / MXC / LiveMACE / PI-Hunter | QUEUED | 无官方公开仓库；LiveMACE 留 Memory 周，MCP 安全类留 MCP 安全/治理周 |
| P5 | dogwood / opencode / langfuse / pullfrog / skillfortify 等 | COMPLETED | corpus 已有对应考古结果，非 refresh 窗口 |

## 选中项目
- **job_id**: ARCH-2026-10-17-001
- **project**: ReviewBench
- **repository**: https://github.com/review-bench/ReviewBench.git
- **mode**: initial
- **priority**: P0（A 级）
- **reason**: 当前 Evaluation 周（10-12~10-18）主题直接命中——能力树 E 层"自建基准方法论"缺口的高质量范例（分布建模 + 多源 golden set + 独立验证三步法）；官方发布 + 公开仓库活跃；未考古；与已有 corpus（pullfrog/validation 编译器族）形成直接连接；Benchmark 学习价值：评测集构建方法论可迁移到引擎 evaluation 层
- **source_entry**: `[cand]reviewbench-2026.md`（03_expansion_queue/candidates/，2026-10-14 深耕 Day3 新增）

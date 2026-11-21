# Job Manifest · ARCH-2026-10-15-001

- **run_id**: ARCH-2026-10-15-001
- **date**: 2026-10-15（Evaluation/Observability 周 Day4，KnowlegeMap 第 43 次扫描）
- **skill_version**: knowledge-archaeology v3.2
- **source_map_commit**: KnowlegeMap f3a18ba（2026-10-15 radar watch）

## 候选排序（非 star 排名，按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 是否已考古 / Benchmark 价值）

| 排序 | 项目 | 状态 | 优先级 | 仓库可用性 | 理由 |
|---|---|---|---|---|---|
| P0 | **RAMPART**（microsoft/RAMPART） | REFRESH_REQUIRED | **S** | ✅ 有公开仓库（★421, Python, 2026-03-24） | pytest-native Agent 安全测试框架，PyRIT 生态之上（corpus 10-04 已考古 PyRIT，直接生态对照）；Agent-AI Engineering 安全 CI 直接场景；2026-05-20 MS AI Red Team 官方发布（FACT）；dsh-pentest/silver-shield/E2E-CLI 连接 |
| P1 | TestJack（arXiv 2610.10619） | NEW | A | ❌ 无公开代码仓库（GitHub API 多角度搜索 0 命中；候选卡验证计划"检查是否有配套代码仓库"结论=无） | 评测可信度三环第四环（补丁/测试侧审计），方法论文本价值高但不可浅克隆考古；维持观察，等待 code release |
| P2 | LiveMACE（arXiv 2610.09872） | NEW | B+ | ❌ 无公开工具/数据集（候选卡自述"未见配套工具"） | 动态演化测试床 + outcome-capability gap 方法论增量；论文级证据，待工具释出 |
| P3 | PI-Hunter（arXiv 2606.12737） | QUEUED | A | ❌ 论文族（未确认工具仓库） | 注入摄入路径审计范式；维持观察 |
| P4 | OTel GenAI 可观测 | QUEUED | A | 标准/平台格局卡（无单一仓库；Langfuse 等已在 corpus） | 标准卡不可单一考古，维持观察 |

**已排除**：求索 4.0（无公开代码仓库）；TestJack/LiveMACE/PI-Hunter（无仓库不满足阶段③浅克隆前提）；agentevals/opencode/transect/reviewbench/agentjudge-bench/promptfoo（已在 corpus 或 validated）。

## 选中项目

- **project**: RAMPART
- **repository**: https://github.com/microsoft/RAMPART.git
- **mode**: refresh（Reconciliation 修正：corpus 已有 ARCH-2026-09-04-001 @ commit 125595cb，本轮 HEAD 2c9067df 前进约 40 天，含 RAMPARTner onboarding skill #195；按 REFRESH_REQUIRED 处理，产物入新 run 子目录，不覆盖 9-04 结果）
- **source_entry**: `[cand]rampart-clarity-2026.md` @ KnowlegeMap 03_expansion_queue/candidates/（2026-09-03 雷达 #2，等级 S）
- **priority**: S（最高——Microsoft AI Red Team 官方、MIT、可落地工具、PyRIT 生态对照、命中 G1 最高优先缺口）
- **reason**: ① Agent 安全 CI/CD pytest 测试框架——"把红队技术变成可嵌入 CI 的测试"，Agent-AI Engineering 安全测试直接相关；② 与 corpus 已考古的 PyRIT（ARCH-2026-10-04-001）构成"生态之上"对照（RAMPART 在 PyRIT 生态之上，架构/职责边界差异值得考古）；③ S 级候选卡为当前雷达最高等级；④ 有公开代码仓库满足考古前提；⑤ 安全测试框架的 pytest-native 设计（插件体系/断言/CI 集成）是 Benchmark 学习对象

## 验证状态（KnowlegeMap 候选卡）
- [x] 维持观察 → 本轮晋升为考古对象（有仓库 + S 级 + 生态连接）

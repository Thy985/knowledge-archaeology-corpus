# Job Manifest — ARCH-2026-09-19-001

job_id: ARCH-2026-09-19-001
project: hermes-agent
repository: https://github.com/NousResearch/hermes-agent.git
mode: initial
priority: A
source_entry: 03_expansion_queue/candidates/[cand]hermes-agent-2026.md（B 级，09-06 增量后"升 A 判断保持"）+ 05_scan_logs/2026-09/2026-09-19_radar-watch_scan.md §5（OpenClaw 生态聚焦）
corpus_slug: hermes-agent（corpus 无冲突，检查通过）

## 候选排序（Top 5，不按 star 排名）
| 排名 | 项目 | 关键维度 | 状态 |
|---|---|---|---|
| 1 | **hermes-agent**（NousResearch） | Knowledge Value 极高（246K★ 增长最猛框架）；Novelty 高（2026 增长最猛）；Growth 极强（pushed 2026-09-18，4 个月 9 大版本 v0.11→v0.20）；近期发布 ✅；与 Corpus 连接极强（letta-code 记忆路线同源：文件化 MEMORY.md 注入扫描 / Skills 自改进；框架格局：openclaw/opencode/omnigent）；Benchmark 价值高（ClawBench 一等公民）；未考古 | NEW |
| 2 | Aigis（gaebalai/aigis-kr） | 运行时防火墙品类代表；但与 AgentGuard 等 10+ 项目同质，GitHub pushed 2026-05-18（5 个月停更），Growth 弱 | NEW（延后） |
| 3 | hermes 备选：AgentGuard（0xrem） | 0★，pushed 2026-03-18，停更 6 个月 | 拒绝（不活跃） |
| 4 | ClawKeeper（智源） | 面向 OpenClaw 安全框架，多源报道但非独立开源仓库可考古（CSDN 报道） | 延后（无一手仓库） |
| 5 | MS Agent Governance Toolkit | 2026-04 发布，重量级但为 monorepo 工具族；与 rampart 部分重叠 | 延后（下轮候选） |

## 选中理由
1. **连续路线递进**：09-18 已考古 letta-code（git-backed MemFS 文件化记忆，治理/沙箱强）；hermes-agent 是同一"文件化记忆"路线的另一代表性实现（MEMORY.md/USER.md + 注入/外泄扫描 + 自我改进循环 + 自主 Curator）——两天考古形成**跨项目对照**，直接服务 Skill 的 Cross-project Candidate 与 K3 模式提取。
2. **最强活跃信号**：246K★（2026-02 发布→09 即 246K），pushed 2026-09-18（昨日），v0.11→v0.20（4 个月 9 个大版本）——远超其他候选（Aigis/AgentGuard 均已停更数月）。
3. **Knowledge 缺口**：Corpus 现有 13 项目无"自我改进/成长型 agent harness"实现样本；hermes 的 self-improving skills + 注入/外泄扫描是 letta-code 未覆盖的维度（letta 有 pre-commit 门禁，hermes 有注入扫描——防御机制的两种实现对照）。
4. **Benchmark 价值**：ClawBench 一等公民 + 自我改进循环可测——为 Validation/benchmark 学习提供高价值样本。

## 连接
- KnowlegeMap [cand]hermes-agent-2026：B 级（09-06 后维持"升 A 评估"）；今日考古完成可作为升 A 证据
- Corpus 对照：letta-code（ARCH-2026-09-18-001）、openclaw、opencode、omnigent

# Transect — Knowledge Archaeology

**一句话定位**：长时程 Agent 评测的"可观测性保留器"——零成本结构扫描 + judged 标签三层校正（投票/verifier/narrator）+ 每个标签携带 provenance 与可靠性信息。

**来源**：UK AISI 官方仓库 [AI-Safety-Institute/transect](https://github.com/AI-Safety-Institute/transect)（main `75a7948`，v0.2.4）+ 论文 [arXiv 2610.08364](https://arxiv.org/abs/2610.08364)（10-08 v2）+ [AISI 博客](https://www.aisi.gov.uk/blog/transect-making-large-scale-agentic-evaluations-easier-to-understand)。

**考古日期**：2026-10-13（run `ARCH-2026-10-13-001`，initial，skill v3.2）

## 产物清单

| 文件 | 内容 |
|---|---|
| `metadata.yaml` | 项目元数据 + 质量指标 + 关键发现/争议 |
| `00_overview.md` | 项目定位 + 认知价值 |
| `01_project-layer.md` | 项目地图（架构分层/模块/生命周期/regime/帧家族/报告结构/可靠性审计/配置/依赖/边界） |
| `02_engineering-knowledge.md` | EK Graph 16 条（六类边，全连接） |
| `03_knowledge-layer.md` | KO 5 个（R1-R4 聚合，含反例压力与升维裁决） |
| `04_flow-atlas.md` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05_candidates.md` | 6 个候选（Hypothesis/暂定/跨项目） |
| `06_validation.md` | 六类审计 + 质量指标 + Reconciliation |
| `archaeology-runs/ARCH-2026-10-13-001/` | 原始 run 包（11 文件，未覆盖） |

## 核心认知（详见 00/03）

1. **judge 标签不可信是第一性约束** —— "Pointing an LLM judge at the transcript only moves the problem"；校正闭环：digest 压缩 → 判定 → 投票共识 → 二轮 verifier（0.6 gate）→ narrator 不改判 → 可靠性度量与 flag。
2. **可溯源性是不变量** —— 每个标签带 anchor_event_id + judge-identity 列，深链回源头回合；违反（layer 无 judge-identity）即显式失败。
3. **自洽性 ≠ 正确性** —— k_roll 高自洽（>0.95）反而 amber（cache 重放/共享 prompt/名称锚定）；同模型 verifier 二轮 ≠ 可重复性度量。
4. **成本-可信治理显式化** —— $0 结构先行、cache 默认、verify_sample 5%、阈值常量集中 + 自包含 flag（explanation/remediation）。

## 跨项目连接

- **AgentJudgeBench**（ARCH-2026-10-12-001，PR #41）：评测可信度双环（judge 审计层 + 结果可观测层），见 Candidates C-06。
- **deepseek-harness / promptfoo / PyRIT** 等既有 corpus：判定可靠性度量作为评测管道标准层的候选证据库。

# 00 Overview — Transect（ARCH-2026-10-13-001）

## 项目定位

**Transect = 长时程 Agent 评测的"可观测性保留器"**：一个基于 Inspect Scout 的 transcript 分析管道，把一次长达数百页（论文演示 1300 万 token）的 agent 运行，压缩成**一条可导航的回合时间线**——结构扫描器以零模型成本提取事件/令牌/上下文压缩/人工干预/子代理活动，judged 扫描器用你自己的行为词汇表给行为打标签，HTML 报告把两者并排呈现，且**每个标签都携带 provenance 与可靠性信息**（如何产生的、judge 哪里分歧）。

核心设计命题（README 原话）："Pointing an LLM judge at the transcript only moves the problem: judged labels cannot be taken at face value either. Transect is built around that fact."

## 规模

- 仓库：5.5MB / 155 文件 / 99 .py / 279 tests（pytest）/ pyproject v0.2.4 / Python ≥3.12
- 代码：UK AISI 官方仓库（AI-Safety-Institute/transect），main `75a7948`（2026-10-07），论文 arXiv 2610.08364（10-06 v1 / 10-08 v2）
- 输入源：Inspect `.eval` log 或 OpenClaw `.jsonl` telemetry；输出：HTML 报告（7 内建 section + 可靠性 audit）+ pandas 帧家族（20 个）+ Scout viewer 深链

## 认知价值（这个项目让我们认识到什么）

1. **"judge 标签不可信"被当作第一性设计约束**——不是事后加 disclaimer，而是从数据结构（basis/label_source/confidence/agreement）、流程（投票→verifier→narrator 三层）、渲染（reliability audit flags + remediation）全部内建
2. **评测可观测性有"包络收缩"问题**——transcript 越长，评测者可靠推断的包络越窄；工具的价值不是生成更多标签，而是**保留标签→源头回合的可追溯性**
3. **判定权威与执行叙述分离**——verifier 可以改写标签（0.6 gate），narrator 永远不改标签（只叙述）；可靠性 audit 是独立第四层
4. **自洽性 ≠ 正确性**——k_roll 自洽高到异常反而 amber 预警（cache 重放/共享 prompt/名称锚定风险），这是少见的"对高指标设防"的评测工具

## Package 索引

| 文件 | 内容 |
|---|---|
| 01_project-layer | 项目地图（架构/模块/生命周期/依赖/入口） |
| 02_engineering-knowledge | EK Graph（15 条，六类边） |
| 03_knowledge-layer | KO（5 个，R1-R4 聚合） |
| 04_flow-atlas | 七类流 |
| 05_candidates | 6 个候选 |
| 06_validation | 六类审计 + 质量指标 + Reconciliation |
| run_metadata.yaml | run 元数据 |

## 核心结论一句话

**长时程 agent 评测的可信度 = 结构零成本锚定 + 判定三层校正（投票/复检/叙述）+ 每个标签可追溯回源头 + 可靠性度量显式化（含"高自洽度设防"）。**

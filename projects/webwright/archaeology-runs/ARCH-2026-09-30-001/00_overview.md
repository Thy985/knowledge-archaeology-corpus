# 00 Overview — microsoft/Webwright（ARCH-2026-09-30-001）

## 一句话定位
**Terminal-Native Web Agent**：把"写代码控制浏览器"作为 agent 的动作面（code-as-action），把本地工作区（脚本+截图+日志）作为状态（workspace-as-state），浏览器只是可丢弃环境。核心主张：**你的 web agent 浏览历史是一个可重跑的 Python 文件**。

## 它让我们认识到了什么（核心认知增量）
1. **动作空间≠预测单步 DOM 操作**：主流 computer-use 让模型每步预测一个点击/输入；Webwright 让模型写 Playwright 脚本、跑、看截图、修。模型变强后，前者是 harness 瓶颈，后者是 harness 的顺势升级（README Motivation）。
2. **验证门而非自裁**：`require_self_reflection_success=true` 时，agent 的 `done=true` 被硬阻塞，直到外部图片判定器（self_reflection 两阶段：per-image Score + final Status verdict）对最终脚本的运行截图给出 `predicted_label==1`。**证明与执行分离**——agent 不能单方面宣称完成。
3. **零 token 技能复用（Skill Factory）**：solve 留下的脚本经 gate（gold/self_verify）准入 → learn/update 蒸馏为参数化 skill → route 时可直接运行（无模型，~40s）或作为 prior 注入 prompt。WebArena held-out 55%→70%（+15pp）。技能不是"模型读的上下文"，而是"可运行的程序"。
4. **双状态模型一个 loop**：workspace 模式（无状态、产物在磁盘、外部验证）与 live browser 模式（有状态会话、ARIA 观察、模型自判）共用同一个 DefaultAgent loop，差异全部收敛在 config 与 action_field。

## 性能锚点（README 声明，非本考古实测）
- Online-Mind2Web 86.7% / Odysseys 60.1%（GPT-5.4 基线 33.5%）
- WebArena：10 retrieve 模板、3 自托管站点、gpt-5.4，reuse 提升 held-out 55%→70%
- 核心 footprint ~1.5k LoC（skill_factory 合并前）；skill 重放 ~40s / zero tokens

## 与已有 Corpus 的连接（Benchmark 对照）
| Corpus 项目 | 关系 |
|---|---|
| browser-use | 同一问题域（web agent harness）反范式对照：browser-use = observe→predict 单步动作 + DOM/AX 快照；Webwright = 写脚本→执行→修复 代码循环。README 对比表明确列出 |
| deepseek-harness | 通用内部 harness（多 agent、会话遥测）↔ Webwright 单 agent 极简终端范式；"自愈/可编程/轻框架"理念同源（候选卡连接） |
| opencode / a2a / mcp | 同属 2026 agent 工程主线；Webwright 的插件化（Claude Code/Codex/OpenClaw/Hermes 四宿主）与 MCP/Agent 协议生态互补 |

## 产物清单
- 01 Project Layer：`package/01_project-layer/project-layer.md`
- 02 Engineering Knowledge（EK Graph）：`package/02_engineering-knowledge/ek-graph.md`
- 03 Knowledge Layer（KO，R1-R4 聚合）：`package/03_knowledge-layer/knowledge-layer.md`
- 04 Flow Atlas（七类流）：`package/04_flow-atlas/flow-atlas.md`
- 05 Candidates：`package/05_candidates/candidates.md`
- 06 Validation & Evidence：`package/06_validation/validation.md`
- run_metadata：`package/run_metadata.yaml`
- Snapshot：`package/snapshot/snapshot_artifact.md`

## 质量指标摘要
facts 31 → EK 44（EK Graph 全覆盖、游离 0）→ KO 8（聚合规则 100%）→ Candidates 6；Independent Audit 判定统计见 06。

## 证据边界声明
- 基准数字（86.7%/60.1%/55%→70%）来自 README 声明，**未在本考古实测复现**（无模型 key / 站点），按 Observation 级证据处理，不可升维为 Fact。
- 全部架构/机制/流程结论直接来自源码（agents/environments/models/skill_factory/tools/config）与测试，符号可回溯。

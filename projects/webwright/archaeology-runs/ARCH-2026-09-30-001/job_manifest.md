# Job Manifest — ARCH-2026-09-30-001

## 候选排序（按 Knowledge Value / Agent-AI Engineering 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 是否已考古 / Benchmark 学习价值，非 star 排名）

| 排序 | 候选 | 状态 | 关键判定 |
|---|---|---|---|
| 1 | **microsoft/Webwright**（Terminal-Native Web Agent） | NEW | Novelty 最高（terminal-native 反主流 computer-use 范式）；MS Research 2026-04 开源；SOTA 可量化（Online-Mind2Web 86.7% vs GPT-5.4 33.5%）；与 corpus browser-use/deepseek-harness 形成"浏览器会话↔终端脚本↔通用 harness"三形态对照；用户 E2E-CLI 直接受益 |
| 2 | langfuse/langfuse（OTel GenAI 可观测） | QUEUED | 昨日活跃（pushed 2026-09-29）+ ★35K 生产大仓；可观测线 corpus 空白；但 Novelty 低于 Webwright（2023 老项目），本期不入 |
| 3 | 0xrem/agentguard（Agent 防火墙） | 淘汰 | ★0、2026-03 后停滞、624KB 小仓；品类已饱和（corpus 已考古 aigis/guardian/ai-protector/agent-governance-toolkit） |
| 4 | 阿里 OpenCodeReview / Pullfrog（Agentic CI） | QUEUED | corpus CI 线空白；但 A7 为 B 级卡，本期信号不足 |
| 5 | agentic-inference-infra（AgentKV/ActKV 等） | QUEUED | 雷达三线之一今日大增量，但 9/10 为 arXiv 论文无可考古仓库；月度回顾时再评估 |

## 选中项目
- **project**: microsoft/Webwright
- **repository**: https://github.com/microsoft/Webwright.git
- **mode**: initial
- **priority**: P1
- **run_id**: ARCH-2026-09-30-001

## 理由
1. **范式 Novelty**：terminal-native web agent（agent 用 Playwright 写代码控制浏览器 + bash + 日志迭代，把浏览器会话当可启动/检查/丢弃的进程，持久产物=可复用脚本）——与主流 vision-based computer-use 反范式；MSR 论文《Webwright: A terminal is all you need for web agents》佐证。
2. **Benchmark 学习价值**：Online-Mind2Web 86.7% / Odysseys 60.1%（GPT-5.4 基线 33.5%）接近翻倍——agent loop（Runner→model→shell command→Environment 反馈）与 skill_factory（code-native 可复用 skill）机制值得深度挖掘。
3. **Corpus 三形态对照**：browser-use（浏览器原生 harness，已考古）↔ Webwright（终端原生 harness）↔ deepseek-harness（通用内部 harness，已考古）——同一"web agent harness"问题域的三种架构策略，Benchmark 价值高。
4. **用户连接**：E2E-CLI（E2E 测试与修复）——"脚本而非会话"设计直接可迁移；campus_order（OpenClaw web 操作）同线。
5. **活跃度核验**：★6020、MIT、Python 主仓、default main、51 open issues（2026-09-30 GitHub API 实测）。

## source_entry
- [cand]webwright-2026.md（2026-09-02 雷达发现，FACT 四重核验：官方 GitHub + microsoft.github.io + MSR writeup + arXiv 论文）
- 关联卡：[cand]computer-browser-use-2026.md（S5 级，同线）

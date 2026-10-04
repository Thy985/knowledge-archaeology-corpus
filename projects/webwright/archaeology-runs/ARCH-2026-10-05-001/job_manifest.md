# Job Manifest — ARCH-2026-10-05-001（Daily Archaeology Job Selection）

## 输入状态（KnowlegeMap @ bd722e8，10-05 雷达第 33 次运行）
- 今日聚焦三线：agentic-attack（S 事件线，重大事件日 9：DIVD 首个全自主 AI 攻击 + 澳洲第二起 + Transluce 侦察）/ computer-browser-use（Copilot desktop CU + Perplexity Comet + a11y 路线）/ agent-harness（NVIDIA open shell + Tokenomics + worker perms）
- 无新卡（3 条既有卡增量），candidates 维持 38 张
- next-step 验证优先级原文：silver-shield 威胁模型加入对抗性自主 agent ＞ dsh-pentest 对照 DIVD 链 ＞ E2E-CLI 借鉴 Copilot computer use

## 候选排序（SOP 维度：Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 已考古 / refresh / Benchmark 价值；不按 star）
| # | 项目 | 等级 | 状态 | 决策 |
|---|---|---|---|---|
| 1 | **microsoft/Webwright**（Terminal-Native Web Agent） | B→A | NEW | **选中**（见下） |
| 2 | agent-firewall 品类（ClawKeeper/AgentGuard/Pipelock） | A | QUEUED | 卡内 aigis/guardian/ai-protector/owasp 已考古，品类边际递减；延后 |
| 3 | trace-based-agent-eval（agentevals） | A | REFRESH_REQUIRED | agentevals 09-12 已考古；卡仍 A，待 refresh 轮 |
| 4 | agent-harness-control-plane | S | QUEUED | 代表 Omnigent 09-07 已考古；MS 产品/arXiv 综述无单一仓库；延后 |
| 5 | agentic-attack（事件线） | S | IN_PROGRESS(雷达线) | 事件无仓库；攻击侧已由 pyrit(10-03) 覆盖；继续雷达跟踪 |
| 6 | computer-browser-use | A | QUEUED | 同域刚考古（browser-use 09-22、browser-harness 10-04）；Copilot/Comet 为闭源产品 |
| 7 | local-edge-llm | A | QUEUED | 雷达标注月度轮 refresh；主题偏推理性能，Agent-AI 相关性中等 |
| 8 | open-source-agentic-ci | B | COMPLETED | pullfrog 10-02 已考古；OpenCodeReview 同域降优先级 |

## 选中项目
- **project**: Webwright
- **repository**: https://github.com/microsoft/Webwright.git
- **mode**: initial
- **priority**: A（卡标 B，按 SOP 维度综合上调）
- **run_id**: ARCH-2026-10-05-001
- **source_entry**: KnowlegeMap `[cand]webwright-2026.md`（B 级，一手四重核验通过：官方 GitHub + microsoft.github.io/Webwright + MSR writeup + arXiv 论文《Webwright: A terminal is all you need for web agents》）

## 选中理由
1. **Novelty + 路线对照**：浏览器 agent 领域第三条独立实现路线——browser-use=LLM 决策层（09-22 已考古）、browser-harness=CDP 连接层（10-04 已考古）、Webwright=**终端/脚本层**（agent 产出可复用 Playwright 脚本而非一次性会话）。三足鼎立补齐 Browser 域认知，跨项目对照价值极高。
2. **Knowledge Value**：skill_factory / Web Skill Factory（code-native 可复用可验证技能）与用户 Skills 体系同构；"浏览器会话=可启动/检查/丢弃的进程"设计隐喻；脚本而非会话的工程纪律（可审查/可回放/可复现）。
3. **Benchmark 学习价值**：Online-Mind2Web 86.7% / Odysseys 60.1%（基线 GPT-5.4 33.5%）——接近翻倍，量化证据充分。
4. **与用户项目连接**：E2E-CLI（脚本化浏览器 = E2E 测试自然升级）；campus_order(OpenClaw web 操作)。
5. **与已有 Corpus 连接**：browser-use / browser-harness（同域对照）+ deepseek-harness（harness 主题）+ opencode（agent harness 家族）。
6. **规模适中**：MSR 研究型仓库，适合单日深度考古。

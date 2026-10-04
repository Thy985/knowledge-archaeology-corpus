# 05 Candidates — 未验证假设与跨项目假说（Webwright）

> 全部为 Hypothesis / 暂定模式 / 不确定结论。**不得升格为已验证 Principle**。
> 每条标注：类型 / 当前证据 / 缺失证据 / 验证路径。

## C-01 code-as-action 的 token 效率优势是否跨模型成立（Hypothesis）
- 内容：Webwright 在 Harness Local 上 424,026 tokens vs Codex Skill 3,291,183 tokens（README compare_trajectory 图）——code-as-action 决策面更紧凑。
- 当前证据：单仓库单对比图（README），gpt-5.4 上有效。
- 缺失证据：跨模型（小模型/开源）、跨任务类型（视觉主导 vs 文本主导站点）、跨 harness 控制的系统对比。
- 验证路径：在同一任务集上用 Opus/Qwen 各跑 Webwright vs 坐标法基线，控变量比较 token/成功率。

## C-02 技能库 LLM retrieval 的可扩展性（Hypothesis）
- 内容：retrieve 默认把 catalog 全量注入 LLM prompt 选 top-k（k=3）；库增长后 token/噪声上升，需换 embeddings。
- 当前证据：retrieve.py 代码注释（"interface is stable so retrieval can be swapped for embeddings"）。
- 缺失证据：>50 技能的检索质量退化曲线；embeddings 变体未实现。
- 验证路径：合成库规模实验，测 recall@k 与耗时。

## C-03 self_verify gate 的自我分级盲区（Hypothesis）
- 内容：self_verify（非空+shape+agent 自报 SUCCESS）会 admit agent 误信的错答案——self-grading 不可作为正确性判据。
- 当前证据：gate.py 显式注释局限 + 升级路径（WebJudge/跨源一致性）。
- 缺失证据：实际误收率统计（哪些任务类型最易自我欺骗）。
- 验证路径：对 admit 技能做 held-out 真值抽查，统计假阳性率。

## C-04 脚本化技能 vs 宿主原生技能（Cross-project Hypothesis）
- 内容：learned skill 以 standalone 脚本形式存在（0 token、可跨宿主运行）vs skills/webwright 插件版把验证交给宿主原生能力（无 API key、不可移植到无视觉宿主）——两者孰优取决于宿主能力面。
- 当前证据：仓库内两套并存（skill_factory/examples + skills/webwright），SKILL.md 明示替换关系。
- 缺失证据：两套在同一任务集上的质量对比；跨 Claude Code/Codex/OpenClaw/Hermes 的行为一致性。
- 验证路径：同任务双跑，比较完成率/证据可审计性/维护成本。

## C-05 Firefox vs Chromium 的指纹规避是否普遍（Hypothesis）
- 内容：插件版 SKILL.md 声明 Firefox 因 TLS/H2 指纹导致部分站点 ERR_HTTP2_PROTOCOL_ERROR 而取代 Chromium；harness 默认仍 Chromium。
- 当前证据：SKILL.md 文本（单宿主实践）。
- 缺失证据：受影响站点清单；指纹规避的稳定性；对 Chrome-only 站点（需特殊 WebGL/扩展）的损失。
- 验证路径：站点矩阵双浏览器跑测，记录失败指纹与完成率。

## C-06 "No hidden orchestration" 与 Skill Factory 管线的张力（暂定观察）
- 内容：README 宣称"no multi-agent system, no graph engine, no plugin layer, no hidden orchestration"，但 skill_factory 是 14 模块三层流水线（learn/build/update/route/recommend）——声明针对的是**运行时 agent**（harness 内部无编排），离线蒸馏管线不属于该声明范围。
- 当前证据：README 原文 + skill_factory 目录结构。
- 缺失证据：作者对"orchestration"边界的明确定义（README 未展开）。
- 验证路径：查 MSR writeup/论文对该声明的界定。

## C-07 Benchmark 数字的第三方复现性（暂定观察）
- 内容：86.7% OM2W / 60.1% Odysseys / WebArena 55→70% 均为 MSR 自报（README），评价体系（AutoEval、OM2W 官方 judge）口径未在仓库内完全复现。
- 当前证据：README §Benchmarks；self_reflection 即仓内 judge 代理。
- 缺失证据：独立第三方复现记录；hard split 稳定性（N=100）。
- 验证路径：复跑 OM2W 300 子集并对比官方 judge 得分。

---
## 跨项目连接（与既有 Corpus）
- browser-use（Corpus 09-22）：LLM 决策层 + CDP 工具调用——与 KO-01（动作空间=代码 vs 工具调用）直接对照。
- browser-harness（Corpus 10-04）：CDP 连接层 harness——与 KO-02（连接 vs 执行环境）衔接。
- deepseek-harness（Corpus 09-03）：SWE harness 范式——Webwright 是其网页域对应物。
- opencode（Corpus 09-09）：agent 编码器——code-as-action 的另一端（写代码 vs 写脚本动作）。
- omnigent（Corpus 09-07）：agent 编排框架——对照"no orchestration"声明。
- E2E-CLI / campus_order（用户自述项目，KnowledgeMap 连接）：脚本化复用与任务分解可借鉴 skill_factory。

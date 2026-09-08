# 05 · Candidates — OpenCode（未验证假说与跨项目候选）

> 每条标注：假设状态 / 当前证据 / 缺失证据 / 验证路径。Hypothesis 不冒充 Fact。

## C-01（Hypothesis · 跨项目候选）"工具副作用统一授权门"是 harness 通用模式
- **内容**：OpenCode 所有副作用工具走 permission.Service；猜想任何生产级 agent harness 都需要统一授权接口（非 per-tool 各自判断）。
- **当前证据**：OpenCode（S3）+ omnigent（策略执行点）+ Dogwood（调用前判定）——**三项目弱印证**，样本 n=3。
- **缺失证据**：DeepSeek Harness（已考古）的授权机制是否同构。
- **验证路径**：对照 corpus projects/deepseek-harness 考古包。

## C-02（Hypothesis）同步阻塞审批（无超时）导致挂起失败模式
- **内容**：`resp := <-respCh` 无超时——UI 失联时工具调用永久挂起；猜想这是早期 harness 的常见失败模式（与 omnigent ASK 86400s 超时对照）。
- **当前证据**：OpenCode permission.go:97-104（S3）。
- **缺失证据**：实际挂起案例（issue/文档未记录）。
- **验证路径**：查 Crush 仓库是否改为超时/异步审批。

## C-03（Tentative Pattern）"增长最快"与"工程成熟度"系统性解耦
- **内容**：OpenCode 13.7k★ 但 4 个测试文件 + panic bug + 随即归档——生态信号与工程质量正交；候选卡甚至把数据夸大到 147k★（失真 10 倍）。
- **当前证据**：OpenCode（S3/S4F）。
- **缺失证据**：第二/三实例的量化对照。
- **验证路径**：抽查其它高 star 早期项目（claw-code/grok build）测试密度。

## C-04（Unresolved Contradiction）候选卡 headless HTTP 描述与仓库实际的差异来源
- **内容**：候选卡说"client/server 解耦 headless Hono + Vercel AI SDK"——本仓库完全不存在；可能混淆 charmbracelet/crush（活跃继任者）或另一项目，也可能描述的是未公开分支。
- **当前证据**：仓库结构（无 TS/Hono/server）+ GitHub API（S3）。
- **缺失证据**：crush 仓库的实际架构（需考古 crush 确认 headless 描述是否对应它）。
- **验证路径**：浅克隆 charmbracelet/crush 核对架构；如属实，KnowlegeMap 候选卡应指向 crush 或修正描述（但 KnowlegeMap 只读，仅记录建议）。

## C-05（Scope-uncertain）copilot provider 的"合作"程度
- **内容**：代码含 copilot provider（bearer token + OpenAI SDK 认证直连）——但"GitHub Copilot 官方合作"声明在仓库无证据；可能只是社区逆向接入。
- **当前证据**：copilot.go（S3）。
- **缺失证据**：官方合作公告/文档引用。
- **验证路径**：查 GitHub 官方博客/opencode 历史 README。

## C-06（Hypothesis）LSP 集成是单体 TUI harness 相对无头 harness 的结构性优势
- **内容**：文件级感知（watcher）+ 诊断注入 + diff 可视化——猜想 headless harness（DeepSeek Harness 等）缺少此能力，这是路线差异而非实现差异。
- **当前证据**：OpenCode LSP 深度（S3）。
- **缺失证据**：DeepSeek Harness 是否含 LSP 能力（已考古包可查）。
- **验证路径**：对照 corpus projects/deepseek-harness。

## C-07（Observation）归档后"迁移对象"crush 是演化终态
- **内容**：Charm 团队把 OpenCode 演进为 Crush（原班人马）——单体 TUI 路线的终态是另一个单体 TUI 还是 headless？决定候选卡描述正确性。
- **当前证据**：README 归档声明（S3）。
- **缺失证据**：crush 架构。
- **验证路径**：克隆 charmbracelet/crush 结构核对（后续轮次候选）。

## C-08（Hypothesis）测试密度可作为 harness 成熟度代理指标
- **内容**：42k 行 / 4 测试文件 → 测试密度（tests-per-kLOC）可预测项目稳定性/归档风险；猜想 <0.1 tests/kLOC 的 agent 项目高归档风险。
- **当前证据**：OpenCode（0.095 tests/kLOC）（S4 实测）。
- **缺失证据**：对照样本集。
- **验证路径**：对已考古 8 项目计算测试密度并对比归档/活跃状态。

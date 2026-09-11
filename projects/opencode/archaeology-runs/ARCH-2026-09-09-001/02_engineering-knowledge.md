# 02 · Engineering Knowledge — OpenCode（EK Graph，宽底座）

> v3.1 规范：每条 EK 声明 `links`（六类边）。证据等级：S3=implemented（代码存在）、S4=测试运行验证（本 run 实测）、S4F=测试运行揭示失败。

## 图统计
- EK 总数：27；有 links：27（100%）；平均出边 ≥2；游离 0
- 证据等级：S3 = 20，S4（编译+测试通过）= 3，S4F（测试运行揭示失败）= 2，S2（设计注释/README 声明）= 2

---

## A. 工具执行与权限

### EK-01 副作用工具统一走权限门
- **内容**：所有可能改变状态的工具（bash/edit/write/patch/fetch）Run 前调用 `permissions.Request(...)`；拒绝返回 `permission.ErrorPermissionDenied`——工具执行与授权解耦，工具自身不判断权限。
- **证据**：bash.go:270-283（Request + ErrorPermissionDenied）；tools.go:28-38（Bash/Edit/Fetch/Patch/Write 均注入 permission）。
- **links**：mechanism→EK-02（审批模型）、subsystem→EK-03（工具集）、causal→EK-04（agent 循环）。
- **等级**：S3。

### EK-02 审批模型：Request→UI 批准/拒绝，同参缓存 + 会话级自动批准
- **内容**：permissionService.Request 顺序：①autoApproveSessions 命中→直接放行；②sessionPermissions 缓存命中（同 tool+action+session+path）→放行；③否则发布 pendingRequests 事件等待 UI 响应（同步阻塞）。AutoApproveSession 实现会话级信任提升。
- **证据**：permission.go（Request:74-107 / AutoApproveSession:110-112）。
- **links**：causal→EK-01、contrast→EK-05（无超时阻塞 vs 超时审批）、constraint→EK-04。
- **等级**：S3。

### EK-03 内置工具集 + MCP 工具统一命名空间
- **内容**：CoderAgentTools = MCP 工具（GetMcpTools，前缀 `{mcpName}_{toolName}` 避免冲突）+ 12 内置工具（bash/edit/fetch/glob/grep/ls/patch/sourcegraph/view/write/agent + diagnostics）；TaskAgentTools 为简化子集——不同 agent 类型不同工具面。
- **证据**：tools.go:14-46（CoderAgentTools/TaskAgentTools）；mcp-tools.go:41-46（Info 前缀）。
- **links**：subsystem→EK-01、mechanism→EK-06（MCP 接入）。
- **等级**：S3。

### EK-04 agent 循环：生成→工具调用→分发→执行→回喂（含取消/汇总）
- **内容**：agent.Run → processGeneration → streamAndHandleEvents：for 循环每次迭代前 select 检查取消；工具调用循环（toolCalls）逐个分发；支持会话级取消（Cancel）与摘要（summarize）请求。
- **证据**：agent.go（Run:198 / processGeneration:233 / for 循环:276-277 cancellation select / tool 循环:352-357）。
- **links**：causal→EK-01→EK-02（权限门在循环内）、subsystem→EK-10（MCP 工具循环内调用）。
- **等级**：S3。

### EK-05 同步阻塞审批（无超时）——设计与对比点
- **内容**：permission.Request 等待 UI 响应 `resp := <-respCh`，**无超时/无取消路径**——工具执行被无限阻塞直到 UI 应答；与 omnigent ASK 审批（默认 86400s 超时 + 静默拒绝被认定 bug）形成审批设计对照。
- **证据**：permission.go:97-104（resp := <-respCh，无 select 超时分支）。
- **links**：contrast→EK-02、constraint→EK-01。
- **等级**：S3。

### EK-06 MCP 服务器接入（mcp-go client）
- **内容**：internal/llm/agent/mcp-tools.go 用 mark3labs/mcp-go 包装 MCP 服务器工具：Initialize → ListTools → CallTool；工具名 `{mcpName}_{toolName}` 前缀命名避免跨服务器冲突；MCP 配置在 config.MCPServer。
- **证据**：mcp-tools.go（mcpTool struct:18-23 / MCPClient 接口:25-33 / Info:35-46 / runTool:48-52）。
- **links**：mechanism→EK-03、subsystem→EK-04、contrast→EK-17（Copilot provider）。
- **等级**：S3。

## B. 模型接入

### EK-07 多 provider 抽象（统一工具/消息接口）
- **内容**：llm/provider 提供 OpenAI/Anthropic/Gemini/Azure/Groq/Bedrock/OpenRouter/Copilot 多实现；统一 models/message 抽象层；config.setProviderDefaults 按 provider 注入 API key 默认值（viper）。
- **证据**：provider 目录（copilot.go/gemini.go/anthropic.go）；config.go:253-268。
- **links**：subsystem→EK-08、mechanism→EK-04。
- **等级**：S3。

### EK-08 Copilot provider：OpenAI SDK + bearer token 认证直连
- **内容**：copilot.go 用 openai-go SDK + bearerToken + extraHeaders 实现 Copilot 认证直连（copilotOptions.reasoningEffort/bearerToken）；这是候选卡"GitHub Copilot 合作"在代码层的唯一形态（认证接入，非产品合作证据）。
- **证据**：copilot.go（copilotOptions:29-40 / OpenAI SDK import）。
- **links**：contrast→EK-07、contrast→EK-17。
- **等级**：S3。

### EK-09 工具重复调用修复：Monkey patch for Copilot Sonnet-4
- **内容**：agent.go:373 注释 "Monkey patch for Copilot Sonnet-4 tool repetition obfuscation"——特定模型（Copilot Sonnet-4）重复生成工具调用的规避补丁；工具循环在 patch 前后对 toolCalls 做特殊处理。
- **证据**：agent.go:352-373（tool 循环 + monkey patch 注释）。
- **links**：mechanism→EK-04（循环内修复）、causal→EK-13（模型行为差异驱动补丁）。
- **等级**：S3。

## C. 编辑器智能（LSP）

### EK-10 LSP 深度集成：protocol 生成代码 + client + watcher
- **内容**：lsp/ 含 protocol/tsprotocol.go（6907 行生成代码）+ tsjson.go（3072）+ client.go（785）+ watcher.go（987 文件变更跟踪）+ methods.go（554）——从 TS LSP 协议自动生成 Go 绑定；watcher 实时跟踪文件变更；diagnostics 工具把类型错误注入上下文。
- **证据**：lsp/ 文件行数；tools/diagnostics.go；tools.go:24（NewDiagnosticsTool(lspClients)）。
- **links**：mechanism→EK-11（文件变更跟踪）、causal→EK-04（diagnostics 进 agent 循环）。
- **等级**：S3。

### EK-11 LSP watcher：文件变更事件驱动
- **内容**：lsp/watcher/watcher.go（987 行）跟踪工作区文件变更（fsnotify），驱动诊断刷新与 diff 可视化——编辑器级文件状态感知是单体 TUI harness 的特色（vs 无头 harness 无此能力）。
- **证据**：watcher.go + fsnotify 依赖（go.mod）。
- **links**：causal→EK-10、subsystem→EK-12。
- **等级**：S3。

## D. 持久化与文件修改

### EK-12 SQLite + SQLC 持久化（会话/消息/文件快照）
- **内容**：db/ 用 sqlc.yaml 生成查询（files/messages/sessions sql），goose migrations + embed（内嵌迁移），ncruces/go-sqlite3 纯 Go 驱动——会话/消息/文件快照全部持久化，支持会话恢复。
- **证据**：db/ 目录（connect.go/db.go/embed.go/sqlc.yaml） + go.mod（goose/go-sqlite3）。
- **links**：subsystem→EK-01、causal→EK-04（会话恢复）。
- **等级**：S3。

### EK-13 diff/patch：文件修改 diff 生成与应用
- **内容**：internal/diff（873 行：diff 生成/可视化）+ internal/patch（740 行：patch 应用）——agent 编辑以 diff 形式呈现/应用，编辑可跟踪可回显（go-udiff 依赖）。
- **证据**：diff/diff.go + diff/patch.go + go.mod（go-udiff）。
- **links**：mechanism→EK-11（变更可视化）、subsystem→EK-01（edit/write/patch 工具）。
- **等级**：S3。

### EK-14 WorkingDirectory panic：无保护全局状态
- **内容**：`config.WorkingDirectory()`（config.go:874）在未初始化时 **panic**；ls 工具（ls.go:99）调用它；测试 ls_test.go:139 触发 panic 导致 FAIL——全局可变工作目录状态 + 无错误返回路径的失败模式。
- **证据**：测试实测（go test tools FAIL，panic trace：config.go:874 ← ls.go:99 ← ls_test.go:139）；S4F。
- **links**：contrast→EK-16（错误处理）、constraint→EK-01（工具执行依赖全局状态）。
- **等级**：S4F。

### EK-15 测试覆盖极低（4 文件 vs 42k 行）
- **内容**：仓库仅 4 个 *_test.go（prompt/ls/theme/custom_commands）；全量 go test 实测：prompt/theme/dialog 3 包 ok，tools 包 FAIL（panic）——早期项目"增长 vs 工程成熟度"解耦的证据。
- **证据**：find *_test.go + go test 实测（S4）。
- **links**：mechanism→EK-14（测试揭示 bug）、constraint→EK-18（归档）。
- **等级**：S4。

## E. 配置/状态/演化

### EK-16 viper 配置默认值注入（API key 自动填充）
- **内容**：config.go:253-268 setProviderDefaults：检测到 anthropic/openai/gemini/groq API key 环境变量后 viper.SetDefault 对应 provider 配置——零配置起步设计。
- **证据**：config.go:253-268。
- **links**：mechanism→EK-07、subsystem→EK-06。
- **等级**：S3。

### EK-17 无 headless HTTP 服务（候选卡失真点）
- **内容**：本仓库**不存在**任何 HTTP 服务/headless API/Hono/Vercel AI SDK 代码（无 TS 文件、无 server 包）——单体 TUI 应用，进程内直接调用 agent 服务；候选卡"headless 服务化"描述与本仓库不符（或对应 Crush 时代）。
- **证据**：仓库结构（无 TS/Hono/server）+ README（"terminal-based"）。
- **links**：contrast→EK-03、constraint→EK-18（此差异导致候选卡失真）。
- **等级**：S3（结构事实）。

### EK-18 归档决策：迁移 charmbracelet/crush
- **内容**：2025-09-17 README 归档声明（"no longer maintained... continued under the name Crush, developed by the original author and the Charm team"）；GitHub API archived: True——单体 TUI 路线在 Charm 团队内部被 Crush 取代的演化节点。
- **证据**：README 顶部 + GitHub API（pushed_at 2025-09-18T02:54:28Z）。
- **links**：causal→EK-17、constraint→EK-15。
- **等级**：S3。

### EK-19 候选卡数据失真（147k★/headless vs 实际 13.7k★/TUI）
- **内容**：KnowlegeMap 雷达 #8 候选卡声称 147k★/650 万月活/headless HTTP/Vercel AI SDK；GitHub API 实测 13,727★、Go 单体 TUI、已归档——**卡数据与仓库实际严重不符**（可能混淆 charmbracelet/crush 或另一项目），构成 Discovery Layer 数据质量案例。
- **证据**：GitHub API（archived: True / stars: 13727）+ 仓库结构。
- **links**：mechanism→EK-18、constraint→EK-20。
- **等级**：S3（API 实测）。

## F. 测试运行证据

### EK-20 测试运行画像（本 run 实测）
- **内容**：GOTOOLCHAIN=auto（下载 go1.24.0）后 `go build ./...` 通过；`go test` 实测：internal/llm/prompt ok（0.125s）、internal/tui/theme ok（0.016s）、internal/tui/components/dialog ok（0.103s）、internal/llm/tools FAIL（ls_test panic，0.053s）。
- **证据**：本 run bash 执行记录（S4/S4F）。
- **links**：causal→EK-14、constraint→EK-15。
- **等级**：S4。

### EK-21 pubsub 事件总线（工具/权限/会话事件）
- **内容**：internal/pubsub 提供事件发布订阅（CreatedEvent 等）；permission 发布权限请求事件（s.Publish(pubsub.CreatedEvent, permission)）；TUI 订阅响应——UI 与后端解耦的事件驱动。
- **证据**：permission.go:95-96（Publish）+ pubsub 包存在。
- **links**：mechanism→EK-02、mechanism→EK-04。
- **等级**：S3。

### EK-22 历史（history）：变更历史追踪
- **内容**：internal/history 记录文件变更历史（edit/write/patch 工具注入 history）；diff 可视化依赖历史——"agent 改了什么"可回看。
- **证据**：tools.go:29-37（Edit/Patch/Write 均传 history）。
- **links**：mechanism→EK-13、subsystem→EK-11。
- **等级**：S3。

## G. 关键实现细节

### EK-23 agent-tool：agent 可调用 agent（递归工具）
- **内容**：NewAgentTool 把 agent 服务本身暴露为工具（agent-tool.go）——agent 可发起子会话/任务（与 TaskAgentTools 区分）；Coder 与 Task 两种 agent 类型。
- **证据**：agent-tool.go（agentTool.Info/Run + NewAgentTool:99）；tools.go:43（TaskAgentTools）。
- **links**：mechanism→EK-04、contrast→EK-03。
- **等级**：S3。

### EK-24 附件与消息模型
- **内容**：message 包定义 ContentPart/Attachment；agent.Run 接收 attachments 并转 message parts（agent.go:216-231）——图片/文件进上下文。
- **证据**：agent.go:216-231 + message 包。
- **links**：subsystem→EK-04、subsystem→EK-12。
- **等级**：S3。

### EK-25 标题生成（仅 coder agent）
- **内容**：agent.go:84 注释 "Only generate titles for the coder agent"——generateTitle 异步生成会话标题（agent.go:154）——会话管理 UX 自动化。
- **证据**：agent.go:84/154。
- **links**：subsystem→EK-04。
- **等级**：S3。

### EK-26 Sourcegraph 工具：代码搜索集成
- **内容**：内置 sourcegraph 工具（tools/sourcegraph.go）——代码搜索直接作为 agent 工具（外部代码库查询）。
- **证据**：tools/sourcegraph.go + tools.go:34（NewSourcegraphTool）。
- **links**：mechanism→EK-03、contrast→EK-10（本地 vs 远程代码智能）。
- **等级**：S3。

### EK-27 Early Development 声明 + 生产边界
- **内容**：README 顶部 "⚠️ Early Development Notice: This project is in early development and is not yet ready for production use. Features may change, break, or be incomplete."——与归档声明并列，参考实现诚实自我声明。
- **证据**：README.md 顶部。
- **links**：mechanism→EK-18、constraint→EK-15。
- **等级**：S2（README 自述）。

---

## EK Graph 边密度

| 指标 | 值 |
|------|-----|
| 平均出边 | ~2.0 |
| 游离 EK | 0 |
| mechanism 边 | EK-01/02/03/04/06/07/09/10/11/13/16/21/22/23/26 |
| causal 边 | EK-01→02→04；EK-09→13；EK-10→11→04；EK-18→17 |
| constraint 边 | EK-02→04；EK-05→01；EK-14→01；EK-15→18；EK-17→18/19 |
| contrast 边 | EK-02↔05；EK-06↔08；EK-10↔26；EK-14↔16 |
| subsystem 边 | 工具族（01/02/03/05）；LSP 族（10/11/22）；持久化族（12/13）；provider 族（07/08）；演化族（17/18/19） |

# 01 · Project Layer — OpenCode（项目地图）

> 本层为 L0 工程事实，全部可追溯到仓库实际内容（commit 73ee493，归档快照）。

## 1.1 项目定位

Go 终端 AI coding agent：Bubble Tea TUI + 多 LLM provider（OpenAI/Anthropic/Gemini/AWS Bedrock/Groq/Azure/OpenRouter/Copilot）+ 工具执行（bash/文件编辑/patch）+ SQLite 持久化 + LSP 集成。README 明确 "Early Development Notice: not yet ready for production"。**2025-09-17 归档，项目迁移 charmbracelet/crush**。

## 1.2 架构（包级）

```
cmd/（cobra CLI 入口：root.go + schema）
internal/
├── app/            # app.go + lsp.go（应用装配/LSP 生命周期）
├── tui/            # Bubble Tea TUI（tui.go 965 + components/chat + dialog/permission 522）
├── llm/
│   ├── agent/      # agent 循环（agent.go 758 + agent-tool.go + mcp-tools.go + tools.go 路由）
│   ├── tools/      # 12+ 内置工具（bash/edit/fetch/glob/grep/ls/patch/shell/view/write/sourcegraph/diagnostics）
│   ├── provider/   # 多 provider（copilot.go 671 / gemini.go 554 / anthropic.go 472 / azure/...）
│   ├── models/ prompt/ message/
├── permission/     # permission.go（Service：Request/Grant/Deny/AutoApproveSession）
├── db/             # SQLite + SQLC（connect/db/embed/files/messages/sessions/migrations + goose）
├── lsp/            # protocol（生成代码 6907+3072）/ client.go（785）/ watcher（987）/ methods
├── config/         # config.go（980：providers/LSP/shell/agents，viper 默认值）
├── diff/ patch/    # diff.go（873）+ patch.go（740）：文件修改应用
├── session/ message/ history/ pubsub/ logging/ fileutil/ format/ completions/ version/
```

## 1.3 核心模块职责

| 模块 | 职责 | 证据 |
|------|------|------|
| llm/agent | agent 循环：生成→工具调用→权限→执行→回喂；取消/汇总；monkey patch | agent.go（Run:198 / streamAndHandleEvents:322 / tool 循环:352 / monkey patch:373） |
| llm/tools | 内置工具集；副作用工具统一持 permission.Service | tools.go 路由（CoderAgentTools:14） |
| permission | 工具权限门：Request→UI 批准/拒绝；会话级自动批准；同参缓存 | permission.go（Request:74 / AutoApproveSession:110） |
| llm/provider | 多 LLM 接入；copilot 认证直连 | copilot.go（bearerToken + OpenAI SDK） |
| lsp | 语言智能：protocol/client/watcher/diagnostics | lsp/protocol（tsprotocol.go 6907 行） |
| db | SQLite + SQLC 持久化（会话/消息/文件快照） | db/（migrations + goose + embed） |
| diff/patch | 文件修改 diff 生成与应用 | diff/patch.go |
| config | 配置（providers/LSP/shell/agents/MCP）+ viper 默认注入 | config.go |

## 1.4 生命周期

- **运行**：CLI（cobra）→ TUI（Bubble Tea）→ app 装配 → session 创建/恢复 → agent.Run → 事件流回 TUI
- **会话**：SQLite 持久化（messages/sessions），支持恢复历史会话
- **工具调用**：agent 生成 tool_call → 分发（agent-tool.go）→ 权限门（permission）→ 执行 → 结果回喂模型
- **LSP**：watcher 跟踪文件变更 → diagnostics 进上下文 → 编辑器智能（类型错误）

## 1.5 主要状态

- 会话状态：SQLite（db/sessions.sql.go + messages.sql.go）
- 权限状态：autoApproveSessions（切片）+ sessionPermissions（缓存）+ pendingRequests（map 等待 UI）
- 文件状态：LSP watcher 跟踪（watcher.go）+ diff 历史（history）

## 1.6 配置与入口

| 项 | 值 | 证据 |
|----|-----|------|
| 入口 | `opencode` CLI（cobra root） | cmd/root.go |
| 配置 | config（providers/LSP/shell/agents）；viper 自动注入 API key 默认值 | config.go:253-268 |
| MCP | config.MCPServer + mcp-go client | mcp-tools.go |
| SQLite | 内嵌（go-sqlite3 + goose migrations） | db/embed.go |

## 1.7 权限 / policy / governance 机制

- **工具权限门**：所有副作用工具（bash/edit/write/patch/fetch）Run 前 `permissions.Request(...)`；拒绝返回 `permission.ErrorPermissionDenied`（bash.go:270-283）
- **信任提升**：AutoApproveSession（会话级自动批准，无逐次确认）
- **同参缓存**：sessionPermissions 匹配 tool+action+session+path 即放行
- **同步阻塞审批**：pendingRequests → pubsub 发布 → `resp := <-respCh`（无超时，UI 响应前阻塞工具执行）
- 归档治理：CONTRIBUTING/LICENSE（MIT）；README Early Development 声明

## 1.8 外部依赖

- charmbracelet 全家（bubbletea/bubbles/glamour/lipgloss）、spf13（cobra/viper）、sqlc + ncruces/go-sqlite3 + goose、mark3labs/mcp-go、OpenAI/Anthropic SDK、pest 无（Go）——go.mod 全量

## 1.9 设计证据入口

- agent.go:373 monkey patch 注释（Copilot Sonnet-4 工具重复调用）
- permission.go（审批模型）
- config.go:253-268（viper API key 注入）
- lsp/protocol（生成代码说明编辑器智能深度）
- README 归档声明 + Early Development 声明

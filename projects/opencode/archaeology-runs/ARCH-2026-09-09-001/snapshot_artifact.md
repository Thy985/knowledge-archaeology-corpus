# Repository Snapshot — OpenCode（ARCH-2026-09-09-001）

```yaml
repository: https://github.com/opencode-ai/opencode
commit_sha: 73ee493265acf15fcd8caab2bc8cd3bd375b63cb
branch: main
tag: (无，归档快照；README 历史建议 VERSION=0.1.0)
repository_version: (无 release tag)
archived: true（2025-09-17，README 归档声明；GitHub API archived: True，pushed_at 2025-09-18T02:54:28Z）
stars: 13727（GitHub API 2026-09-09 实测）
analysis_timestamp: 2026-09-09T02:00:00+08:00
knowledge_archaeology_skill_version: v3.2
```

## 项目基础地图

```
opencode/
├── README.md                  # 归档声明（迁移 charmbracelet/crush）+ Early Development 声明
├── go.mod / go.sum            # Go 1.24.0；charmbracelet 全家 + mcp-go + OpenAI/Anthropic SDK
├── main.go / cmd/             # cobra CLI（root.go + schema）
├── internal/
│   ├── app/                   # app.go + lsp.go（装配/LSP 生命周期）
│   ├── tui/                   # Bubble Tea（tui.go 965 + components/chat + dialog/permission 522）
│   ├── llm/
│   │   ├── agent/             # agent.go 758（Run/循环/取消/汇总/monkey patch）+ agent-tool.go + mcp-tools.go + tools.go（路由）
│   │   ├── tools/             # bash/edit/fetch/glob/grep/ls/patch/sourcegraph/view/write/diagnostics + shell/
│   │   ├── provider/          # copilot.go 671 / gemini.go 554 / anthropic.go 472 / azure / groq / bedrock / openrouter
│   │   ├── models/ prompt/ message/
│   ├── permission/            # permission.go（Service：Request/Grant/Deny/AutoApproveSession）
│   ├── db/                    # SQLite + SQLC（connect/db/embed/files/messages/sessions + migrations + goose）
│   ├── lsp/                   # protocol（tsprotocol.go 6907 + tsjson.go 3072 生成代码）/ client.go 785 / watcher.go 987 / methods.go 554
│   ├── config/                # config.go 980（providers/LSP/shell/agents/MCP；viper 默认注入）
│   ├── diff/ patch/           # diff.go 873 + patch.go 740（文件修改应用）
│   ├── session/ message/ history/ pubsub/ logging/ fileutil/ format/ completions/ version/
├── sqlc.yaml                  # SQLC 查询生成配置
├── install/ scripts/          # 安装脚本（curl | bash / brew / AUR）
└── opencode-schema.json
```

## 识别项

| 项 | 值 | 证据 |
|----|-----|------|
| 主要语言 | Go（module go 1.24.0） | go.mod |
| 主要运行入口 | `opencode` CLI（cobra root）→ TUI | cmd/root.go + main.go |
| 核心模块 | llm/agent（循环）、llm/tools（工具集）、permission（权限门）、llm/provider（多模型）、lsp（编辑器智能）、db（SQLC/SQLite） | internal/ 结构 |
| 核心数据结构 | ToolCall/ToolResponse、PermissionRequest（ID/Path/SessionID/ToolName/Action/Params）、AgentEvent、message.ContentPart、Session/Message（SQLC 模型） | tools.go/permission.go/agent.go/db/models.go |
| 核心状态 | autoApproveSessions + sessionPermissions + pendingRequests（权限）；SQLite 会话/消息；LSP watcher 文件状态 | permission.go/db/ |
| 主要测试体系 | 4 个 *_test.go：prompt_test / ls_test / custom_commands_test / theme_test；ls_test 实测 FAIL（panic） | find + go test |
| 主要配置 | config（providers/LSP/shell/agents/MCP）；viper 默认注入 API key；sqlc.yaml | config.go:253-268 |
| 权限/policy/governance 机制 | permission.Service 门控副作用工具（ErrorPermissionDenied）；AutoApproveSession 信任提升；同参缓存；同步阻塞审批（无超时） | permission.go/bash.go:270 |
| 主要外部依赖 | charmbracelet（bubbletea/bubbles/glamour/lipgloss）、spf13（cobra/viper）、mark3labs/mcp-go、ncruces/go-sqlite3 + goose + sqlc、openai-go/anthropic-sdk-go、fsnotify、go-udiff | go.mod |

## 快照说明

- 浅克隆（--depth 1）HEAD 73ee493，main 分支（归档 final commit，2025-09-17）。
- 规模：140 .go 文件 / 42,162 行（find + wc 实测）。
- 测试运行：GOTOOLCHAIN=auto `go build ./...` 通过；`go test` 实测 prompt/theme/dialog 3 包 ok、tools 包 FAIL（ls_test panic config.WorkingDirectory config.go:874）。
- 本快照事实全部可回溯到仓库实际内容（文件路径/行号见各层引用）。

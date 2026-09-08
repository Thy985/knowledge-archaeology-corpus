# 04 · Flow Atlas — OpenCode（七类流）

> 从真实代码/文档导出；每条 Edge 标注可回溯 symbol / file。

## 04.1 Control Flow（控制流）

```
用户输入 → CLI（cobra root，cmd/root.go）
  → TUI（Bubble Tea，internal/tui/tui.go）
  → app 装配（internal/app/app.go）
  → agent.Run(ctx, sessionID, content, attachments)（agent.go:198）
      → processGeneration（agent.go:233）→ createUserMessage
      → streamAndHandleEvents（agent.go:322）
          → for 循环（agent.go:276）：每次迭代 select 检查取消（277-278）
          → 流式事件（eventChan）→ 工具调用（toolCalls 循环，352-357）
              → 工具分发（agent-tool.go Run:43）
                  → 权限门 permission.Request（bash.go:270）
                      → autoApproveSessions 命中？→ 放行
                      → sessionPermissions 缓存命中？→ 放行
                      → pendingRequests 发布 → UI 应答（resp := <-respCh，无超时）
                  → 工具执行 → 结果回喂（tool call result 消息）
          → 完成 → AgentEvent 流回 TUI
```
关键 symbol：`agent.Run`、`streamAndHandleEvents`、`permission.Request`、`AgentEvent`。

## 04.2 State Flow（状态流）

```
会话状态：SQLite（db/sessions.sql.go + messages.sql.go）
  → 创建/恢复（session 包）→ 消息追加 → 持久化
权限状态（permission.go）：
  autoApproveSessions（[]string，会话级信任提升，AutoApproveSession:110）
  sessionPermissions（[]PermissionRequest 缓存，同 tool+action+session+path 命中）
  pendingRequests（sync.Map：ID→respCh，等待 UI）
文件状态：LSP watcher（watcher.go 987 行，fsnotify）→ 变更事件 → diagnostics 刷新 + diff 历史
```
关键 symbol：`AutoApproveSession`、`sessionPermissions`、`pendingRequests`、`watcher`。

## 04.3 Data Flow（数据流）

```
消息：user content + attachments（agent.go:216-231）→ message.ContentPart
  → 模型请求（provider 统一接口）→ 流式响应 → tool_calls
工具参数：ToolCall → 工具 Info/Run（tools.BaseTool 接口）
  → bash（BashParams）/ edit / write / patch / fetch / glob / grep / ls / sourcegraph / agent
MCP：mcp.InitializeRequest → ListTools → CallTool（mcp-tools.go:48-52）
  → 工具名 {mcpName}_{toolName} 前缀（Info:41-46）
持久化：SQLC 生成查询（sqlc.yaml）→ SQLite（messages/files/sessions）
```
关键 symbol：`ToolCall`、`ToolResponse`、`mcpTool.Info`、`sqlc.yaml`。

## 04.4 Evidence Flow（证据流）

```
运行证据：日志（internal/logging，InfoPersist 持久日志）
权限证据：permission 请求/批准事件（pubsub CreatedEvent → UI → 结果）
变更证据：diff（internal/diff）→ 历史（internal/history）→ patch 应用
诊断证据：LSP diagnostics（diagnostics 工具）→ 上下文
会话证据：SQLite 会话/消息持久化（可恢复/可回放）
测试证据：go test（本 run 实测：3 包 ok / tools FAIL panic）
审计缺口：无显式审计日志模块（依赖 logging 持久化）——与 Dogwood 同族"审计自补"模式
```
关键 symbol：`logging.InfoPersist`、`pubsub.CreatedEvent`、`history`、`sqlc`。

## 04.5 Authority Flow（权威流）

```
配置权威：config（providers/LSP/shell/agents，config.go）→ viper 默认注入 API key（253-268）
工具权威：permission.Service（permission.go）→ 门控所有副作用工具
  ├─ autoApproveSessions：会话级信任提升（"这个会话我信任"）
  ├─ sessionPermissions：同参缓存（"这个动作批准过"）
  └─ pendingRequests：UI 逐次审批（默认路径）
模型权威：provider 认证（copilot bearerToken / anthropic / openai API key）
MCP 权威：config.MCPServer 配置的服务器（工具来源）
```
关键 symbol：`permissionService.Request/Grant/Deny`、`AutoApproveSession`、`copilotOptions.bearerToken`。

## 04.6 Memory Flow（记忆流）

```
长期记忆：SQLite（会话/消息/文件快照）——harness 结构化持久层
  → 会话恢复（session 包）→ 历史上下文重建
短期记忆：模型上下文（消息历史 msgHistory，agent.go:257）
文件记忆：LSP watcher 文件状态 + history 变更历史 + diff
工具记忆：sessionPermissions 缓存（批准历史）
UX 记忆：标题生成（generateTitle，仅 coder agent，agent.go:84）
```
关键 symbol：`db/sessions.sql.go`、`msgHistory`、`history`、`generateTitle`。

## 04.7 Policy Flow（治理闭环：Decision→Approval→Policy→Enforcement→Future Decision）

```
Decision（设计）：
  README Early Development 声明（生产边界）
  Go 单体 TUI 路线（charmbracelet 全家）
Approval（治理门槛）：
  CONTRIBUTING/LICENSE（MIT）
  归档决策（2025-09-17，README 声明 → charmbracelet/crush）
Policy（策略固化）：
  config（providers/LSP/shell/agents）+ MCP 服务器配置
  权限模型（permission.Service：Request/Grant/Deny/AutoApprove）
Enforcement（执行）：
  工具权限门（bash.go:270-283 ErrorPermissionDenied）
  LSP 诊断门（diagnostics 注入）
Future Decision（反馈）：
  monkey patch（Copilot Sonnet-4 重复调用修复，agent.go:373）
  测试失败揭示 WorkingDirectory panic（ls_test.go:139）→ 归档而非修复
  迁移 Crush（架构路线变更）
```
关键 symbol：`ErrorPermissionDenied`、`monkey patch`、`WorkingDirectory panic`、归档声明。

## Flow→KO 交叉校验

| KO | 依赖 Flow 段 | 一致性 |
|----|-------------|--------|
| KO-01 权限门 | 04.5（工具权威）+ 04.1（分发） | ✅ |
| KO-02 循环内治理 | 04.1（循环）+ 04.5 | ✅ |
| KO-03 LSP 集成 | 04.2（文件状态）+ 04.4（诊断证据） | ✅ |
| KO-04 harness 记忆 | 04.6（记忆流）+ 04.3（持久化） | ✅ |
| KO-05 可插拔生态 | 04.3（MCP/工具）+ 04.5（provider 权威） | ✅ |
| KO-06 增长≠成熟 | 04.7（归档/补丁）+ 04.4（测试证据） | ✅ |

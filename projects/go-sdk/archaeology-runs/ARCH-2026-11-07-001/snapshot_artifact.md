# Repository Snapshot · modelcontextprotocol/go-sdk

- **run_id**: ARCH-2026-11-07-001
- **repository**: https://github.com/modelcontextprotocol/go-sdk.git
- **branch**: `main`
- **commit SHA**: `df0f29525bc6c590b6b68d9aae581ec0f21ebb3e`
- **commit time**: 2026-11-06 23:38:14 +0800
- **commit message**: `mcp: make the ioConn frame cap independent of read boundaries (#1510)`
- **version**: v1.8.0 为最新 Release（2026-09-14T08:05:46Z）；HEAD 在 v1.8.0 之后（浅克隆无 tag 对象，版本以 Release + README 兼容表为准）
- **license**: LICENSE 声明 MIT → Apache-2.0 的**过渡状态**；文档 CC-BY-4.0；GitHub API 显示 `NOASSERTION`

## 1. 规模与组成

| 指标 | 值 |
|---|---|
| 文件（不含 .git） | 236 |
| Go 文件 | 155 |
| Go 行数 | 69,302 |
| 测试文件 `*_test.go` | 68 |
| 顶级目录 | `mcp/`、`auth/`、`oauthex/`、`jsonrpc/`、`internal/`、`conformance/`、`examples/`、`docs/`、`design/`、`scripts/` |
| 文档行数 | docs/ 3,247 行（server.md 1,237 / protocol.md 886 / mcpgodebug.md 229） |
| CI workflow | 8 个（test / conformance / nightly / codeql / scorecard / docs-check / publish-docs / require-linked-issue） |
| 模块 | `github.com/modelcontextprotocol/go-sdk`，`go 1.25.0` |

### 包规模（含测试）

| 包 | 行数 | 职责 |
|---|---|---|
| `mcp/streamable.go` | 3,117 | Streamable HTTP 服务端 + 客户端传输，无状态/有状态双路径，resumability |
| `mcp/server.go` | 2,506 | Server、ServerSession、能力/工具/资源/提示注册、discover |
| `mcp/protocol.go` | 2,510 | 协议类型（97 个 type、18 个 method 常量）、JSON schema |
| `mcp/client.go` | 1,737 | Client、ClientSession、connect/握手/回退 |
| `mcp/shared.go` | 1,073 | 协议版本列表与协商、MethodHandler、Session 接口 |
| `mcp/transport.go` | 958 | Transport/Connection 抽象、stdio/in-memory、取消通知 |
| `mcp/sse.go` | 610 | 旧版 SSE 传输（兼容保留） |
| `mcp/event.go` | 497 | EventStore 接口 + MemoryEventStore |
| `auth/authorization_code.go` | 807 | OAuth authorization code 流程 |
| `auth/extauth/*` | 312/272/269 | client credentials / OIDC login / enterprise handler |
| `oauthex/*` | 366/352/216/213 | resource metadata、DCR、token exchange、auth meta |
| `internal/jsonrpc2/conn.go` | 870 | JSON-RPC 2.0 连接层（并发、取消、drain） |

## 2. 核心入口与抽象

库形态（无 `cmd/` 二进制）：

- **服务端**：`NewServer` → `AddTool/AddResource/AddPrompt` → `Server.Connect(transport)`；HTTP 入口是 `NewStreamableHTTPHandler(getServer, opts)` 返回 `http.Handler`。
- **客户端**：`NewClient` → `Client.Connect(ctx, transport, opts)` → `ClientSession` → `CallTool/ReadResource/GetPrompt`。
- **传输**：`Transport` 接口（`Connect(ctx) (Connection, error)`），内置 `StdioTransport`、`InMemoryTransport`、`StreamableClientTransport`、`StreamableServerTransport`、旧 `SSEServerTransport`。
- **会话**：`ClientSession` / `ServerSession`；状态由 `clientSessionState` / `ServerSessionState` 持有。
- **授权**：`auth.RequireBearerToken(verifier)` 中间件 + `auth.TokenInfoFromContext`；`oauthex` 提供 Protected Resource Metadata / DCR / token exchange。
- **事件**：`EventStore`（`Open` / `Append` / `After` / `SessionClosed`），`MemoryEventStore` 为参考实现。

## 3. 核心数据结构

| 结构 | 位置 | 关键字段 |
|---|---|---|
| `Server` | `mcp/server.go:63` | options、tools/prompts/resources 注册表、sessions、protocolVersions |
| `ServerSession` | `mcp/server.go:1680` | state、supportedVersions、listenIDs、onClose |
| `ServerSessionState` | `mcp/server.go` | `InitializeParams`、`InitializedParams`、`NegotiatedProtocolVersion`、`LogLevel` |
| `StreamableHTTPHandler` | `mcp/streamable.go:47` | `sessions map[sessionID]*sessionInfo`、opts、mu |
| `sessionInfo` | `mcp/streamable.go:57` | session、transport、userID、timeout/timer、inflight POST |
| `StreamableHTTPOptions` | `mcp/streamable.go:147` | `Stateless`、`JSONResponse`、`SessionTimeout`、`EventStore`、`StreamKeepAlive` |
| `streamableServerConn` | `mcp/streamable.go:980` | streams map、eventStore、done、sessionID |
| `stream` | `mcp/streamable.go:1052` | id、w、event index、keepalive |
| `Client` / `ClientSession` | `mcp/client.go:30/473` | impl、opts、state、listenCancel |
| `TokenInfo` | `auth/auth.go:21` | UserID、Scopes、Expiration |
| `InitializeParams` | `mcp/protocol.go:1129` | ProtocolVersion、ClientInfo、Capabilities |

## 4. 状态与生命周期

### 协议版本（`mcp/shared.go:56-82`）

```
2026-07-28（latest，默认）
2025-11-25
2025-06-18
2025-03-26
2024-11-05
```

- `negotiatedVersion`：仅用于 legacy `initialize`，**上限为 2025-11-25**；对 2026-07-28 不再协商。
- `negotiateMutuallySupportedVersion`：客户端在收到 `CodeUnsupportedProtocolVersion` 时从中挑选双方支持且 ≥ 2026-07-28 的版本重试。

### 会话生命周期

| 阶段 | 新协议（≥2026-07-28） | 旧协议（≤2025-11-25） |
|---|---|---|
| 建立 | 客户端先发 `server/discover`（无状态 RPC） | `initialize` → `notifications/initialized` |
| 请求 | 每次 POST 建 **ephemeral session**，`defer session.Close()` | 首次 POST 建 session，返回 `Mcp-Session-Id` |
| 续连 | 可选 `subscriptions/listen` 长流 | GET 独立 SSE 流（可选） |
| 终止 | 请求结束即关闭；无 DELETE 语义 | `DELETE` + `Mcp-Session-Id` 显式关闭 |
| 断点重放 | 需 `EventStore` + `Last-Event-ID` | 同左 |

### 服务端 HTTP 分派（`streamable.go:385` `ServeHTTP`）

- `Stateless=true` → `serveStateless`：只允许 POST；DELETE 仅在 `MCPGODEBUG=allowsessionsinstateless=1` 时返回 204 no-op。
- `Stateless=false` → `serveStateful`：GET（SSE）/ POST / DELETE。
- 无 sessionID 的 stateful POST 会创建新 session；若 `GetSessionID()` 返回 `""` 则退化为 ephemeral。

## 5. 权限与治理机制

- **Bearer token 中间件**：`auth.RequireBearerToken` 校验后把 `TokenInfo` 注入 context；`TokenInfo.UserID` 被 `lookupSession` 用于**会话劫持防护**（user 不匹配返回 403）。
- **OAuth**：authorization code（`auth/authorization_code.go`）、client credentials / OIDC login / enterprise managed authorization（`auth/extauth/`）、Protected Resource Metadata / DCR / token exchange（`oauthex/`）。
- **新协议禁止 server-initiated 请求**：`assertServerInitiatedRequestAllowed`（`server.go:1758`）在 ≥2026-07-28 时拒绝 sampling/roots/elicitation 的服务器主动请求，要求改用 `InputRequests`（SEP-2322 多轮往返）。
- **弃用窗口**：roots/sampling/logging 由 SEP-2577 在 2026-07-28 弃用，SDK 继续支持至少 12 个月（README）。
- **开发治理**：`AGENTS.md` 规定 gofmt、版权头、测试命令、文档生成（README 由 weave 自动生成，`DO NOT EDIT`）。

## 6. 测试体系

- 单元/集成测试：**68 个 `*_test.go`**，`go test ./...`。
- 官方一致性测试：`scripts/client-conformance.sh` / `scripts/server-conformance.sh`，套件从 npm 下载；`conformance/everything-client`、`conformance/everything-server`。
- `conformance/baseline.yml` **显式列出未通过/未实现的用例**：`auth/client-credentials-jwt`、`auth/enterprise-managed-authorization`、`auth/dpop`、`auth/dpop-nonce`、`auth/wif-jwt-bearer`。
- 专项测试文件量大：`streamable_test.go` 5,279 行、`mcp_test.go` 4,904、`server_test.go` 3,051、`streamable_client_test.go` 2,433、`mrtr_test.go` 1,787。
- CI：`test.yml` / `conformance.yml` / `nightly.yml` / `codeql.yml` / `scorecard.yml`。

## 7. 外部依赖（`go.mod`）

`golang-jwt/jwt/v5`、`google/jsonschema-go`、`segmentio/encoding`、`yosida95/uritemplate/v3`、`golang.org/x/oauth2`、`golang.org/x/time`、`golang.org/x/tools`；间接 `segmentio/asm`、`x/sync`、`x/sys`。

## 8. 已知"粗糙边缘"（官方自述，`docs/rough_edges.md`）

- `EventStore.Open` 不必要（早期版本遗留）。
- `Event` 不应导出，`Name` 应为 `event`。
- 无效工具名无法在 v1 panic，只能 error log（SEP-986 晚于 v1）。
- 命名不一致（`ResourceUpdatedNotificationsParams`、`ProgressNotificationParams`）。
- `AudioContent.MarshalJSON` 应为指针接收者。
- `ClientCapabilities.Roots` 应为独立 struct 指针（提供 `RootsV2` 过渡）。
- 默认能力非空（server 默认 logging、client 默认 roots listChanged），`nil` 不等于"无能力"。
- `CreateMessageResult.Content` 单数 vs 规范允许数组（用 `...WithTools` 变体过渡）。

## 9. 风险与边界（供考古阶段核对）

1. **协议双栈**：新协议路径与 legacy 路径并存，任一行为需先判定请求版本，不能按单一实现推断。
2. **无状态 ≠ 无会话**：无状态模式下仍存在**请求级 ephemeral session**，且 `initialize` 语义被 SDK 合成（`ephemeralConnectOpts`）。
3. **断点重放依赖外部 EventStore**：无 EventStore 时不能重放；`Close` 用 `context.TODO()` 调 `SessionClosed`，事件存储关闭可能无超时。
4. **初始化门禁被注释**：`checkInitialized` 当前恒返回 nil（TODO 明确），旧协议下"未 initialized 就发请求"不会被拒。
5. **一致性基线不覆盖全部 OAuth 机制**：DPoP / WIF / client-credentials-jwt / enterprise-managed-authorization 未实现。
6. **许可证文件头未统一**：120 个 Go 文件写 "MIT-style license"，0 个写 "Apache License"，与 LICENSE 的过渡声明存在表述差异。

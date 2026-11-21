# 01.3 Data Model · 核心类型与状态

## 1. 协议类型（`mcp/protocol.go`，97 个 type、18 个 method 常量）

| 类别 | 代表类型 |
|---|---|
| 初始化/发现 | `InitializeParams`、`InitializeResult`、`DiscoverParams`、`DiscoverResult`、`UnsupportedProtocolVersionData` |
| 会话 | `ServerSession`、`ClientSession`、`ServerSessionState`、`clientSessionState` |
| 能力 | `ServerCapabilities`、`ClientCapabilities`、`ToolCapabilities`、`ResourceCapabilities`、`PromptCapabilities`、`SamplingCapabilities`、`ElicitationCapabilities` |
| 工具 | `Tool`、`CallToolParams`、`CallToolParamsRaw`、`CallToolResult`、`InputRequestMap` |
| 资源 | `Resource`、`ResourceTemplate`、`ReadResourceParams/Result`、`SubscribeRequest` |
| 提示 | `Prompt`、`GetPromptParams/Result`、`PromptMessage` |
| 内容 | `TextContent`、`ImageContent`、`AudioContent`、`EmbeddedResource`、`ResourceLink` |
| 结果扩展 | `resultType`（`complete` / `input_required`）、`Cacheable` |
| 进度/通知 | `ProgressNotificationParams`、`ResourceUpdatedNotificationParams`、`LoggingMessageNotificationParams` |

## 2. 传输层类型

| 类型 | 位置 | 作用 |
|---|---|---|
| `Transport` | `transport.go:45` | `Connect(ctx) (Connection, error)`，一个 transport 只能连接一次 |
| `Connection` | `transport.go:70` | `Read/Write/Close`；Close 必须能解除 Read 阻塞 |
| `ProtocolVersionSupporter` | `transport.go:58` | 可选能力：声明该 transport 能服务的协议版本 |
| `StreamableHTTPHandler` | `streamable.go:47` | HTTP 入口，持有 `sessions` map |
| `StreamableHTTPOptions` | `streamable.go:147` | 无状态/JSON/EventStore/SessionTimeout/安全开关 |
| `StreamableServerTransport` | `streamable.go:878` | 服务端传输，含 `SessionID`、`Stateless` |
| `streamableServerConn` | `streamable.go:980` | 流集合、事件重放、写队列 |
| `stream` | `streamable.go:1052` | 单条 SSE 流：id、writer、事件索引、keepalive |
| `StreamableClientTransport` | `streamable.go:2212` | 客户端 HTTP 传输 |

## 3. 会话状态字段

`ServerSessionState`：

| 字段 | 语义 | 何时设置 |
|---|---|---|
| `InitializeParams` | 初始化参数；**nil = 尚未 initialize** | 真 `initialize` 或 `server/discover` 版本匹配时 |
| `InitializedParams` | `notifications/initialized`；**nil = 尚未 initialized** | `initialized` 通知 |
| `NegotiatedProtocolVersion` | 实际服务版本 | initialize/discover 或旧协议合成 |

`sessionInfo`（有状态 HTTP）：

| 字段 | 语义 |
|---|---|
| `session *ServerSession` | 会话对象 |
| `transport *StreamableServerTransport` | 传输 |
| `userID string` | 从 `TokenInfo.UserID` 复制，用于会话劫持防护 |
| `timeout / timer` | 空闲超时与 `AfterFunc` |
| 并发计数 | `startPOST/endPOST/stopTimer` |

## 4. 结果类型与"需要输入"

`CallToolResult` / `InputRequiredResult`：

- `resultType == "input_required"` → `NeedsInput() == true`。
- `InputRequests` 是 `map[server-assigned ID] → input request`。
- **`InputRequests` 为空但 `NeedsInput()==true` = load-shedding**（`protocol.go:439`）。
- 客户端必须填充输入后重试（多轮往返，MRTR）。

## 5. 事件存储契约

`EventStore`（`event.go:196`）：

| 方法 | 契约 |
|---|---|
| `Open` | 新 stream 建立时调用（官方称"不必要"，可 no-op） |
| `Append` | 追加事件数据 |
| `After` | 返回从 index 之后的迭代器；**一旦数据被丢弃必须立即报错，不得返回部分结果** |
| `SessionClosed` | 会话结束通知；**不能依赖它做清理**，需自带超时回收 |

`MemoryEventStore` 为参考实现；`Event.Name` 是导出字段（官方认为不应导出）。

## 6. 授权数据

| 类型 | 字段 |
|---|---|
| `auth.TokenInfo` | `UserID`、`Scopes`、`Expiration` |
| `auth.TokenVerifier` | `func(ctx, token, *http.Request) (*TokenInfo, error)` |
| `auth.RequireBearerTokenOptions` | `Scopes`、`ClockSkew`、`AllowMissingExpiration` |
| `oauthex.ProtectedResourceMetadata` | 资源、授权服务器、scope 等（RFC 9728） |

## 7. 数据模型判断

1. **协议版本是字符串**（`"2026-07-28"`），比较依赖 ISO 日期字典序；这使版本比较简单，但要求版本号格式严格。
2. **会话状态是"可合成的"**：旧协议请求没有握手也能被塞入伪造的 `InitializeParams`，因此"有状态字段"不等于"发生过握手"。
3. **"需要输入"与"负载卸载"共用同一结果类型**，区别只在 `InputRequests` 是否为空——语义密度高，但容易误读。

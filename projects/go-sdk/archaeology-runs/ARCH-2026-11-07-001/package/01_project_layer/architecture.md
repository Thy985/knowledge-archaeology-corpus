# 01.1 Architecture · modelcontextprotocol/go-sdk

## 1. 模块图

```
                  ┌──────────────────────────┐
   HTTP 客户端 ───▶│ StreamableHTTPHandler    │  (mcp/streamable.go)
                  │  ├─ serveStateless       │
                  │  └─ serveStateful        │
                  └───────────┬──────────────┘
                              │ connectStreamable
                              ▼
                  ┌──────────────────────────┐
                  │ Server / ServerSession    │  (mcp/server.go)
                  │  tools / prompts /        │
                  │  resources / subscriptions│
                  └───────────┬──────────────┘
                              │ jsonrpc2.Connection
                              ▼
                  ┌──────────────────────────┐
                  │ internal/jsonrpc2         │  帧/并发/取消/drain
                  └───────────┬──────────────┘
                              │ Connection
                              ▼
                  ┌──────────────────────────┐
                  │ Transport                 │  (mcp/transport.go)
                  │  stdio / in-memory /      │
                  │  StreamableClient/Server  │
                  └──────────────────────────┘
```

## 2. 包责任

| 包 | 责任 | 对外可见 |
|---|---|---|
| `mcp` | 核心 API：Client、Server、Session、Tool/Resource/Prompt、Streamable HTTP、SSE、EventStore | 是 |
| `jsonrpc` | 公共 JSON-RPC 2.0 传输接口（自定义 transport 用） | 是 |
| `auth` | OAuth 原语：TokenInfo、TokenVerifier、Bearer 中间件、authorization code | 是 |
| `oauthex` | OAuth 扩展：Protected Resource Metadata、DCR、token exchange、auth metadata | 是 |
| `internal/jsonrpc2` | 连接层实现（Read/Write 循环、取消、drain） | 否 |
| `internal/{json,authutil,util,xcontext,mcpgodebug}` | 编解码、工具、调试开关 | 否 |
| `conformance/{everything-client,everything-server}` | 官方一致性套件的参考实现 | 否（测试用） |

## 3. 入口

- 服务端（HTTP）：`mcp.NewStreamableHTTPHandler(func(r *http.Request) *mcp.Server, *mcp.StreamableHTTPOptions) http.Handler`
- 服务端（stdio）：`mcp.NewServer(...)` + `mcp.NewStdioTransport()` + `server.Connect(ctx, transport)`
- 客户端：`mcp.NewClient(...)` + `client.Connect(ctx, *mcp.StreamableClientTransport, opts)`
- 自定义传输：实现 `mcp.Transport`（`Connect(ctx) (Connection, error)`），底层可复用 `jsonrpc` 包

## 4. 依赖方向（无环）

```
examples / conformance  →  mcp  →  internal/jsonrpc2  →  internal/{json,util}
                              ↘  auth  →  oauthex  →  internal/authutil
```

- `mcp` 不依赖 `auth` 的内部实现，只消费 `TokenInfo` / `TokenInfoFromContext`。
- `auth` 与 `oauthex` 互相独立，`auth` 依赖 `oauthex` 的 metadata 类型。
- `examples/server/distributed` 展示跨进程 EventStore/session 用法。

## 5. 架构判断

1. **库不是服务**：没有 `cmd/`，运行时由使用方提供；因此所有生命周期/超时/安全默认值都是"库默认值"，可被 `StreamableHTTPOptions` / `ServerOptions` 覆盖。
2. **传输是唯一分叉点**：`serveStateless` 与 `serveStateful` 是同一个 handler 的两条 HTTP 路径；两条路径共用 `streamableServerConn` 的流/事件机制。
3. **协议版本不是全局变量**：`Server.protocolVersions` 是服务端配置，`ServerSession.supportedVersions` 是"transport 实际能服务"的子集，`DiscoverResult` 每次返回该子集。

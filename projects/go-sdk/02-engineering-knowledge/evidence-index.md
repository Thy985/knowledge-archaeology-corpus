# 02.1 Evidence Index（50 Facts）

> 证据等级：S2=设计/文档声明；S3=代码已实现；S4=有测试覆盖（本包仅记"存在对应测试文件"，未逐条断言）；S5=运行时验证（本轮未运行二进制）。
> 所有行号基于 commit `df0f29525bc6c590b6b68d9aae581ec0f21ebb3e`。

| ID | Fact | 证据（file:line / 命令） | 等级 | Flow |
|---|---|---|---|---|
| E-01 | HEAD=`df0f2952`，2026-11-06 23:38 +0800，commit #1510 | `git log -1` | S3 | Data |
| E-02 | 236 文件 / 155 Go 文件 / 69,302 行 Go / 68 测试文件 | `find`+`wc` | S3 | Data |
| E-03 | module `github.com/modelcontextprotocol/go-sdk`，Go 1.25.0 | `go.mod:1-3` | S3 | Data |
| E-04 | 直接依赖 jwt/v5、jsonschema-go、segmentio/encoding、uritemplate、oauth2、x/time、x/tools | `go.mod:5-13` | S3 | Data |
| E-05 | 支持 5 个协议版本：2026-07-28 / 2025-11-25 / 2025-06-18 / 2025-03-26 / 2024-11-05 | `mcp/shared.go:56-82` | S3 | Policy |
| E-06 | latest = `protocolVersion20260728` = "2026-07-28" | `mcp/shared.go:63-66` | S3 | Policy |
| E-07 | `negotiatedVersion` 仅服务 legacy initialize，上限 2025-11-25；无可用版本返回 "" | `mcp/shared.go:88-109` | S3 | Policy |
| E-08 | `negotiateMutuallySupportedVersion` 按新→旧取交集 | `mcp/shared.go:111-121` | S3 | Policy |
| E-09 | `StreamableHTTPOptions.Stateless`：不读/不设 session header、GET/DELETE 405、每请求临时会话 | `mcp/streamable.go:147-186` | S2+S3 | State |
| E-10 | `serveStateless` 只允许 POST；DELETE 仅在 `allowsessionsinstateless=1` 时处理 | `mcp/streamable.go:443-455` | S3 | Control |
| E-11 | 无状态 legacy DELETE 要求 `Mcp-Session-Id` 但直接 204（no-op） | `mcp/streamable.go:516-523` | S3 | Control |
| E-12 | `Stateless` 模式强制 `Content-Type: application/json`，Accept 必须同时含 json + event-stream | `mcp/streamable.go:456-475` | S3 | Control |
| E-13 | 每请求 `connectStreamable` 新建 session，`defer session.Close()` | `mcp/streamable.go:494-514` | S3 | State |
| E-14 | `serveStateful` 分派 GET/POST/DELETE，其它 405 + Allow | `mcp/streamable.go:623-640` | S3 | Control |
| E-15 | `lookupSession` 缺失 404；`userID` 不匹配返回 403 `session user mismatch` | `mcp/streamable.go:642-660` | S3 | Authority |
| E-16 | 有状态 POST 无 sessionID 时创建新 session；`GetSessionID()==""` 退化为 ephemeral | `mcp/streamable.go:710-790` | S3 | State |
| E-17 | 会话注册 `h.sessions[transport.SessionID]=sessInfo`，`SessionTimeout` 用 `AfterFunc` | `mcp/streamable.go:790-820` | S3 | State |
| E-18 | 初始化失败（`InitializeParams()==nil`）在 defer 中 `session.Close()`（#578） | `mcp/streamable.go:818-826` | S3 | State |
| E-19 | `ephemeralConnectOpts` 读 body，判定 initialize/initialized/subscriptions/listen | `mcp/streamable.go:538-575` | S3 | Control |
| E-20 | 旧协议请求被合成 `InitializeParams`/`InitializedParams`/`LogLevel="info"` | `mcp/streamable.go:579-597` | S3 | State |
| E-21 | `usesNewProtocol` 判定：任一新版请求（>=2026-07-28）即 true | `mcp/streamable.go:565-575` | S3 | Policy |
| E-22 | 客户端 >=2026-07-28 先 `server/discover`，最多 2 次；失败回退 legacy | `mcp/client.go:309-410` | S3 | Control |
| E-23 | 回退时版本强制 `protocolVersion20251125` | `mcp/client.go:404-410` | S3 | Control |
| E-24 | `CodeUnsupportedProtocolVersion` + Supported 列表 → 协商后重试 | `mcp/client.go:352-379` | S3 | Policy |
| E-25 | initialize 返回版本不在 supported 列表 → 关闭会话并报错 | `mcp/client.go:393-402` | S3 | Policy |
| E-26 | `Server.discover` 返回 transport 可服务的版本、capabilities、instructions | `mcp/server.go:999-1035` | S3 | Evidence |
| E-27 | discover 仅当版本在列表内才记录 `InitializeParams`/`NegotiatedProtocolVersion` | `mcp/server.go:1010-1024` | S3 | State |
| E-28 | `ServerSession.supportedVersions` = transport 过滤后的子集（`filterSupportedVersions`） | `mcp/server.go:1590-1600` | S3 | Policy |
| E-29 | `ProtocolVersionSupporter` 可选接口；未实现则视为支持全部版本 | `mcp/transport.go:58-68` | S3 | Policy |
| E-30 | `checkInitialized` 校验逻辑整段注释，恒返回 nil（TODO） | `mcp/server.go:1730-1743` | S3 | Policy |
| E-31 | `initialized` 处理：无 initialize 报错；重复 initialized 报错 | `mcp/server.go:1611-1635` | S3 | Control |
| E-32 | ≥2026-07-28 禁止 server-initiated sampling/roots/elicitation | `mcp/server.go:1758-1766` | S3 | Policy |
| E-33 | MRTR：`resultType=input_required` + `InputRequests`，客户端补齐后重试 | `mcp/protocol.go:25-28/376-392` | S3 | Control |
| E-34 | `NeedsInput()==true` 且 `InputRequests` 为空 = load-shedding | `mcp/protocol.go:437-440` | S3 | Policy |
| E-35 | `EventStore` 契约：Open/Append/After/SessionClosed；After 不得返回部分结果 | `mcp/event.go:196-227` | S3 | Data |
| E-36 | `SessionClosed` 调用不保证；存储需自带超时回收 | `mcp/event.go:215-220` | S2+S3 | Data |
| E-37 | 断点重放：`acquireStream(streamID,lastIdx)` + `EventStore.After` | `mcp/streamable.go:1505-1560` | S3 | Data |
| E-38 | 并发 replay 同一 stream 返回 409 `stream ID conflicts` | `mcp/streamable.go:1535-1548` | S3 | Data |
| E-39 | `Close` 调 `EventStore.SessionClosed(context.TODO())`，无超时 | `mcp/streamable.go:2194-2205` | S3 | Data |
| E-40 | 旧版 SSE 传输仍在 `mcp/sse.go`（610 行） | `mcp/sse.go` | S3 | Control |
| E-41 | DNS rebinding：loopback 服务拒绝非 loopback Host（403） | `mcp/streamable.go:385-397` | S3 | Authority |
| E-42 | `MaxBytesReader` 默认 4 MiB，超限 413 | `mcp/streamable.go:409-412/265` | S3 | Control |
| E-43 | 默认 `StreamKeepAlive=30s`，以 SSE 注释行保活 | `mcp/streamable.go:278-289` | S3 | Control |
| E-44 | `PropagateRequestCancellation` 仅新版协议生效 | `mcp/streamable.go:226-244` | S3 | Control |
| E-45 | `validateProtocolVersion` 只拒绝 `<2026-07-28` 的不支持 header；新版不按 header 校验 | `mcp/streamable.go:425-441` | S3 | Policy |
| E-46 | 新协议每请求注入 `_meta`（protocolVersion/clientInfo/clientCapabilities） | `mcp/client.go:541-568` | S3 | Data |
| E-47 | `RequireBearerToken` 中间件 + `TokenInfoFromContext` | `auth/auth.go:21/83/97` | S3 | Authority |
| E-48 | token 检查顺序：缺失→401；invalid→401；expiration（默认必须存在）→401；scope→403 | `auth/auth.go:133-196` | S3 | Authority |
| E-49 | `conformance/baseline.yml` 列出 5 个未实现 OAuth 用例 | `conformance/baseline.yml:1-13` | S2 | Evidence |
| E-50 | `docs/rough_edges.md` 自述 8 类 API 遗留；`docs/mcpgodebug.md` 已有 1.9.0 段落（HEAD 面向 1.9.0） | `docs/rough_edges.md`、`docs/mcpgodebug.md` | S2 | Evidence |
| E-51 | LICENSE：MIT→Apache-2.0 过渡；文档 CC-BY-4.0 | `LICENSE` | S2 | Policy |
| E-52 | 120 个 Go 文件头写 "MIT-style license"，0 个写 "Apache License" | `grep -rl` 实测 | S3 | Policy |
| E-53 | 8 个 CI workflow（test/conformance/nightly/codeql/scorecard/docs-check/publish-docs/require-linked-issue） | `.github/workflows/` | S3 | Evidence |

> 计数：53 条 Fact。E-50 合并两条官方文档事实，编号连续。

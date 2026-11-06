# 01.2 Lifecycle · 会话、请求、流

## 1. 服务端 HTTP 请求生命周期

```
ServeHTTP (streamable.go:385)
 ├─ DNS rebinding 检查（localhost 时 Host 必须 loopback）
 ├─ CrossOriginProtection（可选）
 ├─ MaxBytesReader（默认上限）
 ├─ 读 MCP-Protocol-Version header → 注入 context
 ├─ Stateless ? serveStateless : serveStateful
 │
 ├─ serveStateless (443)
 │   ├─ DELETE 仅在 allowsessionsinstateless=1 时 → 204 no-op
 │   ├─ 非 POST → 405 + Allow: POST
 │   ├─ Content-Type 必须 application/json
 │   ├─ Accept 必须同时含 json + event-stream
 │   ├─ validateProtocolVersion（仅 <2026-07-28 的 header 会被拒）
 │   ├─ ephemeralConnectOpts（读 body 判定版本/初始化）
 │   ├─ newStreamableServerTransport(sessionID, stateless=true)
 │   ├─ connectStreamable → ServerSession
 │   ├─ defer session.Close()
 │   └─ transport.ServeHTTP → JSON 或 SSE 响应
 │
 └─ serveStateful (623)
     ├─ GET  → serveStatefulGET（需要 sessionID、Accept: event-stream）
     ├─ POST → serveStatefulPOST
     └─ DELETE → serveStatefulDELETE（session.Close() + 204）
```

## 2. 有状态会话生命周期

| 事件 | 代码 | 行为 |
|---|---|---|
| 首次 POST（无 sessionID） | `serveStatefulPOST:710` | 新建 `ServerSession`，`SessionID=GetSessionID()` |
| 注册会话 | 同上 | `h.sessions[transport.SessionID] = sessInfo`；记录 `TokenInfo.UserID` |
| 空闲超时 | `SessionTimeout>0` | `time.AfterFunc` 到点 `session.Close()`；`endPOST` 重置计时器 |
| 并发 POST | `sessionInfo.startPOST/endPOST` | 计 refcount，最后一个 POST 结束才启动 timeout |
| 后续请求 | `lookupSession:642` | 查 map；`userID != ""` 时比对 `TokenInfo.UserID`，不匹配 403 |
| 显式关闭 | `serveStatefulDELETE` | `session.Close()`，`onClose` 删除 map 条目并 `stopTimer` |
| 初始化失败清理 | `serveStatefulPOST` defer | `InitializeParams()==nil` → `session.Close()`（issue #578） |

## 3. 无状态请求生命周期

- **每个 POST 一个 ephemeral session**：`connectStreamable` → `defer session.Close()`。
- `Stateless=true` 时**不读也不设** `Mcp-Session-Id`（`StreamableHTTPOptions.Stateless` 文档）。
- 服务端主动请求（server→client request）**立即拒绝**，因为没有可回应的通道；通知可在请求上下文内送达。
- GET/DELETE 返回 405；兼容开关 `allowsessionsinstateless=1` 可恢复旧行为（DELETE no-op + 读 sessionID）。
- `GetSessionID` 不被咨询。

## 4. 客户端连接生命周期

```
Client.Connect (client.go:309)
 ├─ connect（建立 jsonrpc2 连接）
 ├─ protocolVersion = opts.ProtocolVersion || latest(2026-07-28)
 ├─ if >= 2026-07-28:
 │    ├─ server/discover（最多 2 次）
 │    │    ├─ 成功 → 保存 InitializeResult；按 capabilities 订阅 list_changed
 │    │    └─ CodeUnsupportedProtocolVersion 且带 Supported → 协商后重试
 │    └─ 失败 → 回退 legacy（版本强制 2025-11-25）
 ├─ initialize → 校验返回版本在 supportedProtocolVersions 内
 ├─ notifications/initialized
 └─ KeepAlive>0 → startKeepalive
```

## 5. 流与断点重放生命周期

1. 请求产生 event → `EventStore.Append(sessionID, streamID, data)`。
2. 客户端断线重连带 `Last-Event-ID`（`formatEventID(sid, idx)`）→ `acquireStream(streamID, lastIdx)`。
3. `acquireStream` 优先复用在途 stream；若 stream 已结束则建临时条目**独占 replay**；并发 replay 返回 409。
4. `After` 逐条重放；EventStore 丢弃数据时必须**立即报错**，不得返回部分结果。
5. `session.Close()` → `EventStore.SessionClosed(context.TODO(), sessionID)`；文档明确"不能依赖该方法清理，需自带超时/回收"。

## 6. 状态机（协议版本视角）

```
                    ┌────────────────────────┐
  新客户端 ──discover──▶ 2026-07-28 无状态路径 │
                    └────────────────────────┘
                              │ 失败（任何非现代错误）
                              ▼
                    ┌────────────────────────┐
  旧客户端 ──initialize──▶ 2025-11-25 有状态路径 │
                    └────────────────────────┘
                              │ initialized
                              ▼
                        正常请求（server-initiated 允许）
```

> 关键：**不是"服务器决定版本"，而是"客户端先用新协议探测，失败才降级"**；服务端只在旧协议路径上校验 `MCP-Protocol-Version` header。

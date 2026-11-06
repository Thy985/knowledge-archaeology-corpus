# 04 Flow Atlas · 七类流（27 Edge）

> 所有 Edge 均可回溯到符号/文件/条件；失败路径单列。行号基于 `df0f29525bc6`。

## 1. Control Flow（控制流，5）

| ID | From → To | Trigger | Symbol / File | Condition | Failure Path |
|---|---|---|---|---|---|
| F-C1 | `ServeHTTP` → `serveStateless` / `serveStateful` | HTTP 请求 | `streamable.go:385/443/623` | `h.opts.Stateless` | `getServer` 返回 nil → 400 |
| F-C2 | `Client.Connect` → `discover` → `initialize` | 建连 | `client.go:309/421` | 版本 ≥2026-07-28 | discover 失败 → 回退；仍失败 → 连接错误 |
| F-C3 | `serveStatefulPOST` → `connectStreamable` → `Server.Connect` | POST | `streamable.go:710` | 有/无 sessionID | `GetSessionID()==""` → ephemeral |
| F-C4 | handler → `InputRequiredResult` → client retry | 工具调用需输入 | `protocol.go:25-28` | `NeedsInput()` | 客户端不支持 MRTR → 调用不可完成 |
| F-C5 | `session.Close()` → `onClose` / `stopTimer` / `EventStore.SessionClosed` | 超时/显式 DELETE/请求结束 | `streamable.go:2194-2205` | 有 EventStore | `SessionClosed` 无超时，长关闭可能阻塞 |

## 2. State Flow（状态流，5）

| ID | From → To | Trigger | Symbol / File | Condition | Failure Path |
|---|---|---|---|---|---|
| F-S1 | 无 sessionID → 新 `ServerSession` | 首个 POST | `streamable.go:730-790` | stateful | 初始化失败 → `InitializeParams==nil` → Close |
| F-S2 | `sessions[ID]` 插入 → 查找 → 超时/删除 | 请求进出 | `streamable.go:790/642` | `SessionTimeout>0` | 超时关闭；DELETE 删除 |
| F-S3 | 请求开始 → ephemeral session → 请求结束关闭 | 每次 POST（stateless） | `streamable.go:494-514` | `Stateless=true` | 请求 panic/中断 → defer 仍关闭 |
| F-S4 | `InitializeParams=nil` → `session.Close()` | POST defer 检查 | `streamable.go:818-826` | 初始化未成功 | 关闭后客户端需重试 |
| F-S5 | `InitializeResult` → `usesNewProtocol()` | 客户端发送前 | `client.go:530-535` | 版本 ≥2026-07-28 | 版本为空 → false → 不发 `_meta` |

## 3. Data Flow（数据流，4）

| ID | From → To | Trigger | Symbol / File | Condition | Failure Path |
|---|---|---|---|---|---|
| F-D1 | HTTP body → `ephemeralConnectOpts` → 合成 init state | POST | `streamable.go:538-597` | 旧协议/无 session | body 超限 → 413 |
| F-D2 | 事件 → `EventStore.Append` → `After` 重放 | 流写入/重连 | `streamable.go:1505-1560` | 有 EventStore + Last-Event-ID | 数据被丢弃 → `After` 必须报错（不得部分重放） |
| F-D3 | Bearer token → `TokenInfo` → `sessionInfo.userID` | 会话创建 | `streamable.go:790-810`；`auth/auth.go:83` | 启用 auth | 无 token → 401；userID 空 → 不绑定 |
| F-D4 | 请求参数 → `injectRequestMeta` → wire `_meta` | 每次出站请求 | `client.go:541-568` | 新协议 | 已有 key 不覆盖（调用方优先） |

## 4. Evidence Flow（证据流，3）

| ID | From → To | Trigger | Symbol / File | Condition | Failure Path |
|---|---|---|---|---|---|
| F-E1 | `server/discover` → 版本列表/capabilities/instructions → 客户端 | 探测 | `server.go:999-1035` | transport 实现了版本声明则过滤 | 版本不匹配 → 不记录 init state → 会话被关闭 |
| F-E2 | conformance baseline → CI 判定"哪些用例不跑" | CI | `conformance/baseline.yml`；`.github/workflows/conformance.yml` | 固定用例清单 | 清单漂移 → 缺口被掩盖 |
| F-E3 | `rough_edges.md` → 使用者迁移决策 | 文档生成 | `docs/rough_edges.md`（weave 自动生成） | — | 生成源过期 → 文档落后于代码 |

## 5. Authority Flow（权限流，4）

| ID | From → To | Trigger | Symbol / File | Condition | Failure Path |
|---|---|---|---|---|---|
| F-A1 | `Authorization` header → verifier → `TokenInfo` → context | 每个受保护请求 | `auth/auth.go:97-132` | Bearer 格式 | 缺失 → 401；verifier 错误 → 401/400/500 |
| F-A2 | `TokenInfo.UserID` → `lookupSession` 比对 → 403 | 有状态会话请求 | `streamable.go:642-660` | `sessionInfo.userID != ""` | 不匹配 → 403 `session user mismatch` |
| F-A3 | `req.Host` vs loopback → 403 | 每个 HTTP 请求 | `streamable.go:385-397` | 未禁用 localhost 防护 | 非 loopback Host → 403 |
| F-A4 | `TokenInfo.Scopes` → scope 检查 → 403 | 授权后 | `auth/auth.go:180-195` | `opts.Scopes` 非空 | 缺失 scope → 403 `insufficient_scope` |

## 6. Memory Flow（记忆流，3）

| ID | From → To | Trigger | Symbol / File | Condition | Failure Path |
|---|---|---|---|---|---|
| F-M1 | `ServerSession` → `h.sessions` map | 有状态会话创建 | `streamable.go:790` | stateful | 进程重启 → map 丢失（需外部存储/粘性路由） |
| F-M2 | 事件 → `MemoryEventStore` / 外部 EventStore | Append | `event.go` | 配置 EventStore | 内存实现不能跨进程；外部实现需实现 `After` 错误契约 |
| F-M3 | `InitializeParams` / `InitializedParams` → `ServerSessionState` | 握手/通知/discover | `server.go:1680+` | 版本匹配 | 合成状态使"有字段"≠"真握手" |

## 7. Policy Flow（策略流，3）

| ID | From → To | Trigger | Symbol / File | Condition | Failure Path |
|---|---|---|---|---|---|
| F-P1 | `MCPGODEBUG` 开关 → 行为分支 | 进程启动 | `internal/mcpgodebug`；`docs/mcpgodebug.md` | 环境变量 =1 | 开关过期未退役 → 行为分叉累积 |
| F-P2 | `SupportedProtocolVersions` → discover 列表 / initialize 返回 | 配置 | `server.go:200-230` | 列表非空 | 只留新版 → 旧客户端收到 2025-11-25 后断开 |
| F-P3 | SEP-2577 弃用策略 → roots/sampling/logging 保留 ≥12 个月 | 版本 ≥2026-07-28 | `server.go` 注释；README | 弃用窗口内 | 窗口结束后未移除 → 永久双栈 |

## 8. Flow → KO 交叉校验

| KO | 依赖的 Flow Edge | 是否矛盾 |
|---|---|---|
| KO-01 | F-C2, F-E1, F-P2 | 否（discover/回退与版本列表一致） |
| KO-02 | F-S3, F-S1, F-D1, F-M3 | 否（ephemeral 与合成状态一致） |
| KO-03 | F-A2, F-D3 | 否（有状态才绑定，无状态退化） |
| KO-04 | F-E1, F-D4, F-S5 | 否（三平面均有落点） |
| KO-05 | F-C2, F-S4, F-C4 | 否（旧协议 fail-open、新协议用 MRTR 约束） |
| KO-06 | F-D2, F-C5, F-M2 | 否（契约与实现一致） |
| KO-07 | F-C4, F-D4 | 否（InputRequests 与 `_meta` 并存） |
| KO-08 | F-E2, F-E3 | 否（负向清单是证据源） |


## 9. Reconciliation 追加（2 Edge）

| ID | From → To | Trigger | Symbol / File | Condition | Failure Path |
|---|---|---|---|---|---|
| F-D5 | JSON-RPC method/name/params → `Mcp-Method`/`Mcp-Name`/`Mcp-Param-*` header | 客户端出站 / 服务端入站 | `streamable_headers.go:231-282/416-480` | 参数在 header 白名单内 | header 与 body 不一致 → 解码时报错；未在白名单 → 只能走 body |
| F-P4 | `SSEServerTransport.SupportsProtocolVersion` → discover 版本过滤 | discover | `sse.go:232-236` | 版本 ≥2026-07-28 | 旧 SSE 不宣告新版 → 客户端看不到该 transport 的新协议能力 |

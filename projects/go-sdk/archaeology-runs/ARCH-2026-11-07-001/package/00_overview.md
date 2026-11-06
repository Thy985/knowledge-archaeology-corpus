# 00 Overview · modelcontextprotocol/go-sdk

> **一句话定位**：MCP 协议的**官方 Go SDK**——把"客户端/服务端 + 工具/资源/提示 + 传输 + OAuth"封装成库；在 2026-07-28 协议从"有状态双向会话"转为"无状态 request/response"的转折点上，用**双协议栈 + 探测优先 + 无损回退**完成迁移，同时保留 12 个月弃用兼容窗口。

## 1. 这个项目让我们认识到了什么

不是"MCP 有一个 Go SDK"，而是：

**协议代际迁移的工程解法，不是"改实现"，而是"让两代协议在同一入口共存，并把版本判定下沉到每一次请求"。**

go-sdk 在同一个 `StreamableHTTPHandler` 里同时服务 2026-07-28（无状态）与 2025-11-25 及更早（有状态），客户端先 `server/discover` 探测、失败再回退 `initialize`。这条路径揭示了一个可迁移的结构：

```
传输层（Transport）
   ├─ 版本能力声明（ProtocolVersionSupporter）
   ├─ 请求级自描述（_meta: protocolVersion/clientInfo/clientCapabilities）
   └─ 会话策略（无状态=ephemeral / 有状态=sessionID+userID 绑定）
          ↓
协议层（method handler）
   ├─ 新协议：discover + MRTR（InputRequests）
   └─ 旧协议：initialize + server-initiated requests（deprecated 但保留）
          ↓
安全层（auth 中间件 + 传输保护）
   ├─ Bearer token → TokenInfo → session user 绑定
   └─ DNS rebinding / CORS / body cap / fail-closed token
```

## 2. 七个关键问答

| 问题 | 答案（详见对应章节） |
|---|---|
| 它是什么？ | 官方 Go SDK：`mcp`（核心）、`jsonrpc`（传输接口）、`auth`/`oauthex`（OAuth），无二进制 |
| 怎么运行？ | 库形态：`NewServer` + `Server.Connect` / `http.Handler`；`NewClient` + `Client.Connect` |
| 会话是什么？ | 新协议下是**请求级 ephemeral session**；旧协议下是 `Mcp-Session-Id` + `sessionInfo` |
| 版本怎么定？ | 三平面：HTTP header（旧协议）、请求 `_meta`（新协议）、`initialize`/`discover` 握手 |
| 状态放哪？ | 服务端 `sessions` map（有状态）或每次请求新建（无状态）；重放依赖外部 `EventStore` |
| 权限怎么做？ | `auth.RequireBearerToken` 中间件 + OAuth 流程；会话与 `TokenInfo.UserID` 绑定 |
| 边界在哪？ | 未实现 DPoP/WIF/client-credentials-jwt/enterprise-managed-auth；初始化门禁当前被注释 |

## 3. 八条关键发现

1. **双协议栈是显式的，不是隐式的**（`client.go:309-421`）：≥2026-07-28 先 `server/discover`；任何非现代错误即回退 legacy `initialize`，并把版本强制降到 2025-11-25。
2. **"无状态"仍创建会话**（`streamable.go:443-516`）：每个 POST 都 `connectStreamable` 建 `ServerSession`，`defer session.Close()`；它去掉的是**跨请求会话标识**，不是会话对象。
3. **旧协议兼容靠"合成状态"**（`streamable.go:538-598`）：`ephemeralConnectOpts` 为旧协议请求伪造 `InitializeParams`/`InitializedParams`，否则 init gate 会拒绝。
4. **初始化门禁被主动关闭**（`server.go:1730-1743`）：`checkInitialized` 的校验逻辑整段注释，只留 TODO；顺序校验只在 `initialized` 通知处理里生效。
5. **新协议禁止服务器主动请求**（`server.go:1758-1766`）：sampling/roots/elicitation 必须改为 `InputRequests`（SEP-2322 多轮往返）。
6. **过载语义藏在结果类型里**（`protocol.go:439`）：`NeedsInput()==true` 且 `InputRequests` 为空 = load-shedding。
7. **断点重放是外部契约**（`event.go:196-227`）：`EventStore` 的 `After` 必须"一旦丢弃就立即报错，不得返回部分结果"；`SessionClosed` 不能依赖一定被调用。
8. **官方自己维护"粗糙边缘"清单**（`docs/rough_edges.md`）与 conformance baseline（5 个未实现 OAuth 用例）——这是声明-实现差距的**现成证据索引**，也是本项目最值得学习的方法论资产。

## 4. 三层配比

| 层 | 数量 | 说明 |
|---|---|---|
| Facts（E） | 46 | 全部可回溯到 file:line 或官方文件 |
| Engineering Knowledge（EK） | 30 | EK-01~EK-30，六类边 |
| Knowledge Objects（KO） | 7 | KO-01~KO-07，R1-R4 |
| Candidates（C） | 6 | C-01~C-06 |
| Flow Edge | 26 | 七类流 |

> 项目规模 69,302 行 Go，按 skill 的"小项目按比例缩放"原则，不硬凑 100+ Facts / 40+ EK。

## 5. 认知状态声明

- **Fact**：仓库代码、官方文档、conformance baseline 可直接引用的陈述。
- **Observation**：由多处 Fact 归纳但未跨项目验证的观察。
- **Hypothesis**：进入 Candidates，不写成知识。
- **Validated Pattern**：本项目内多路径证据闭合的机制（如双协议栈、请求级会话）。
- **Principle / L4**：仅在解释范围能覆盖本项目多个子系统时提升；跨项目一律标 `Cross-project validation pending`。
- 本包**没有**把任何单项目经验写成"普遍定律"。

## 6. 目录结构

```
00_overview.md
01_project_layer/     architecture / lifecycle / data_model / config_and_testing / governance
02_engineering_knowledge/  evidence-index / ek-01-15 / ek-16-30 / ek-graph
03_knowledge_layer/   ko-01-04 / ko-05-07 / ko-aggregation-matrix
04_flow_atlas/        flow-atlas.md
05_candidates/        candidates.md
06_validation/        internal_validation.md / counterexamples.md
run_metadata.yaml
```

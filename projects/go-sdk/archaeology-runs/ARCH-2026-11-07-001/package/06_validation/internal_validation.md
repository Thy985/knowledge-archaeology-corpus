# 06.1 Internal Validation

> 验证原则：先盲重建、再对比；不因结论漂亮而放宽证据。本文件为**内部**自检记录，独立 Auditor 的结果见 `06-validation/independent-validation.md`（corpus 交付时）。

## 1. Truth Audit（这句话是真的吗）

抽查 12 条关键 Fact，逐条回源码核对：

| Fact | 核对结果 |
|---|---|
| E-05/E-06 版本列表与 latest | ✅ `shared.go:56-82` 常量与 `supportedProtocolVersions` 顺序一致 |
| E-07 `negotiatedVersion` 上限 | ✅ 返回前有 `min(version, protocolVersion20251125)` 逻辑与注释 |
| E-09 Stateless 语义 | ✅ 选项注释与 `serveStateless` 行为一致 |
| E-13 每请求 ephemeral | ✅ `connectStreamable` + `defer session.Close()` |
| E-15 会话 user 校验 | ✅ `lookupSession` 403 `session user mismatch` |
| E-20 合成初始化 | ✅ `ephemeralConnectOpts` 赋 `InitializeParams`/`InitializedParams` |
| E-22/E-23 discover + fallback | ✅ `Client.Connect` 分支与版本强制 |
| E-30 checkInitialized 恒 nil | ✅ 校验整段注释 |
| E-32 server-initiated 禁令 | ✅ `assertServerInitiatedRequestAllowed` |
| E-34 load-shedding | ✅ `protocol.go:439` 注释原文 |
| E-35 EventStore 契约 | ✅ `After` 注释"must not return partial results" |
| E-48 token 顺序 | ✅ expiration 在 scope 之前 |

**结果**: 12/12 CONFIRMED。

## 2. Coverage Audit（还有什么没发现）

| 区域 | 覆盖状态 |
|---|---|
| `mcp/shared.go` | 全读关键段 |
| `mcp/streamable.go` | 关键路径全读（ServeHTTP/serveStateless/serveStateful/lookupSession/serveStatefulPOST/acquireStream/Close） |
| `mcp/server.go` | 关键段（ServerOptions/Connect/discover/ServerSession/checkInitialized/assertServerInitiated） |
| `mcp/client.go` | Connect/usesNewProtocol/injectRequestMeta/capabilities |
| `mcp/protocol.go` | 类型与 MRTR 相关段 |
| `mcp/transport.go` / `event.go` / `sse.go` | 接口与关键注释 |
| `auth/auth.go` | 全读核心 |
| `auth/authorization_code.go`、`auth/extauth/*`、`oauthex/*` | **仅结构/文件级**，未逐行 |
| `internal/jsonrpc2/conn.go` | **未逐行**（870 行） |
| `docs/protocol.md` / `server.md` | 未全文（仅 rough_edges / mcpgodebug） |

**结论**: 核心传输/会话/协议/授权中间件覆盖；OAuth 具体流程与 JSON-RPC 连接层是缺口，记入 C-05。**不宣称"覆盖全仓库"**。

## 3. Flow Audit（Flow 是否真的对应代码）

抽查 10 条 Edge：

| Edge | 核对 |
|---|---|
| F-C1 | ✅ `ServeHTTP` 分派 |
| F-C2 | ✅ `Client.Connect` discover→initialize |
| F-S3 | ✅ ephemeral + defer Close |
| F-S4 | ✅ `InitializeParams()==nil` 清理 |
| F-D2 | ✅ `acquireStream` + `After` |
| F-D3 | ✅ token→userID |
| F-D4 | ✅ `injectRequestMeta` |
| F-A2 | ✅ 403 mismatch |
| F-A3 | ✅ Host 检查 |
| F-P2 | ✅ `SupportedProtocolVersions` 注释 |

**结果**: 10/10 CONFIRMED；1 条语义需标注：F-S2 的"超时/删除"由 `AfterFunc` 与 `onClose` 两处实现，非单一函数。

## 4. Abstraction Audit（是否过度升维）

| KO | 判定 | 理由 |
|---|---|---|
| KO-01 | L3 保留 | 三文件解释范围扩大 |
| KO-02 | L4 保留 + pending | 覆盖传输/状态/身份/配置；跨项目未验 |
| KO-03 | L3 保留 | 不变量跨会话与授权 |
| KO-04 | L3 保留 | 三平面解释 |
| KO-05 | L4 保留 + pending，Epistemic 降为 Observation | 动机不可证；反例存在 |
| KO-06 | L3 保留 | 契约解释 |
| KO-07 | L3 保留 | 机制簇 |
| KO-08 | L3 保留（方法论） | 不升 L4，因许可证反例说明"显性化"非全覆盖 |

**未升维项**: 没有把"官方 Go SDK 的做法"写成"所有 SDK 都应如此"。

## 5. Counterexample Audit

见 `counterexamples.md`：12 次定向攻击，5 次导致表述削弱。

## 6. Epistemic Audit

| 检查 | 结果 |
|---|---|
| Hypothesis 是否被写成 Fact | 否；C-01~C-07 全部在 Candidates |
| Observation 是否被写成 Principle | 否；KO-02/KO-05 标 `Cross-project validation pending` |
| 设计意图是否被写成实现事实 | 否；`rough_edges`/注释类内容标 S2 或"S2+S3" |
| 是否有"未发现=不存在"的推断 | 否；C-05 明确覆盖缺口 |

## 7. 质量指标

| 指标 | 值 |
|---|---|
| Facts | 53 |
| EK | 30 |
| KO | 8 |
| Candidates | 7 |
| Flow Edge | 27 |
| EK 平均度 | 2.47 |
| 孤立 EK | 0 |
| 聚合规则覆盖 | 8/8 = 100% |
| Truth 抽查通过 | 12/12 |
| Flow 抽查通过 | 10/10 |
| 反例攻击 | 12 |
| 因反例削弱 | 5 |

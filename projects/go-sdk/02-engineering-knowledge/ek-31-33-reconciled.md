# 02.4 Reconciliation 追加 · EK-31 ~ EK-33

> 来源：Independent Validation 的 M-01 / M-02 / M-03 遗漏项。原 Package 保留不动，本节为 corpus 交付版本的增补。

---

## EK-31 HTTP header 是第二套参数承载面（`Mcp-Method` / `Mcp-Name` / `Mcp-Param-*`）

- **Level**: L2
- **Statement**: `streamable_headers.go` 把 JSON-RPC 的 `method`、工具/资源名、以及白名单参数编码进 HTTP header；服务端可解码回参数，并校验 header 与 body 是否一致。
- **Why**: 让网关/中间件在不解析 JSON body 的前提下完成路由、限流、审计与观测；对无状态部署尤其重要（每个请求都要被独立路由）。
- **Evidence**: IF-01, IF-02（`streamable_headers.go:28-30/231-282/416-480`；`streamable_headers_test.go` 1,919 行）
- **Links**: `mechanism` ↔ EK-17（同为"请求自描述"）；`subsystem` ↔ EK-23（传输层加固）；`contrast` ↔ EK-45（header 版本校验）
- **Value**: A（此前被漏掉的核心机制）
- **Epistemic**: Fact

## EK-32 旧 SSE 传输不宣告 ≥2026-07-28

- **Level**: L2
- **Statement**: `SSEServerTransport.SupportsProtocolVersion(version)` 返回 `version < protocolVersion20260728`；因此旧 SSE 会在 discover 中被 transport 过滤掉新版。
- **Why**: 2026-07-28 只定义 stdio 与 Streamable HTTP 两种绑定；旧 HTTP+SSE 不能服务新协议。把限制写在 transport 上，比在 Server 里特判更准确。
- **Evidence**: IF-03（`mcp/sse.go:232-236`）
- **Links**: `dependency` ← EK-03（transport 过滤）；`subsystem` ↔ EK-29（旧能力保留）
- **Value**: B
- **Epistemic**: Fact

## EK-33 `CloseSSEStream` 在新协议下是 no-op

- **Level**: L2
- **Statement**: 服务端主动断流（触发客户端 Last-Event-ID 重连）的机制在 ≥2026-07-28 时直接 return；代码注释标记 deprecated。
- **Why**: 新协议没有"服务端主动断流 + 客户端重连同一 stream"的语义；继续执行会制造与无状态模型冲突的隐式状态。
- **Evidence**: IF-04（`mcp/streamable.go:1808-1818`）
- **Links**: `causal` ← EK-05（无状态模型）；`contrast` ↔ EK-19（重放仍保留）
- **Value**: B
- **Epistemic**: Fact

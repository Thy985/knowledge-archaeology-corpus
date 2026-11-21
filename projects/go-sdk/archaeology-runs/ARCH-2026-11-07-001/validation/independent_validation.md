# Independent Validation Report · go-sdk (ARCH-2026-11-07-001)

> **角色**：独立 Auditor。方法：不把 `package/` 当作事实来源，先独立重读仓库的 6 个区域（其中 4 个是 Package 覆盖较少或未覆盖的），形成 Independent Findings，再与 Package 逐条对比。
> **只读**：未修改任何 Package 文件；修正建议只写入本报告。

## 0. 独立重读范围

| 区域 | 为什么选它 |
|---|---|
| `mcp/streamable_headers.go`（604 行） | Package 完全未提及，但属于 `mcp` 核心 |
| `mcp/sse.go` 的 `SupportsProtocolVersion` | Package 只说"保留旧 SSE"，未验证它如何参与版本协商 |
| `mcp/streamable.go:1800-1830`（CloseSSEStream） | 新协议弃用路径，Package 未覆盖 |
| `examples/server/distributed/main.go` | 分布式部署的会话语义，Package 未引用 |
| `auth/authorization_code.go`（807 行） | Package 只写"authorization code 流程"，未验证 PKCE/DCR |
| `internal/jsonrpc2/conn.go`（870 行） | commit #1510 的主题（frame cap），Package 未验证 |

## 1. Independent Findings（独立得出的 9 条）

| IF | 发现 | 证据 |
|---|---|---|
| IF-01 | `streamable_headers.go` 把 JSON-RPC 的 method/name/params 编码进 HTTP header（`Mcp-Method`、`Mcp-Name`、`Mcp-Param-<name>`），使中间件无需解析 body 即可路由/限流/观测 | `streamable_headers.go:28-30/231-282` |
| IF-02 | 该 header 编码是双向的：客户端可把参数镜像到 header，服务端可解码回参数；并有专门测试（`streamable_headers_test.go` 1,919 行） | `streamable_headers.go:416-480` |
| IF-03 | `SSEServerTransport.SupportsProtocolVersion` 明确返回 `version < 2026-07-28`，即旧 SSE **不宣告**新协议 | `mcp/sse.go:232-236` |
| IF-04 | `CloseSSEStream`（服务端主动断流触发客户端 Last-Event-ID 重连）在 ≥2026-07-28 时直接 return（no-op），标记 deprecated | `streamable.go:1808-1818` |
| IF-05 | 分布式部署示例明确"必须无状态，因为后续请求不保证落到同一后端；但仍可有逻辑 session ID（verbose logging）" | `examples/server/distributed/main.go:9-16` |
| IF-06 | `auth/authorization_code.go` 实现 PKCE 支持检查、动态客户端注册（DCR）、token auth method 选择、protected resource metadata 解析 | `authorization_code.go:342/602/656/565` |
| IF-07 | 授权码流程把 `usedCodeVerifier` 记录在 handler 中，说明 PKCE verifier 是**每次授权请求**的状态，不是全局配置 | `authorization_code.go:678` |
| IF-08 | `internal/jsonrpc2/conn.go:672` 有一条注释明确"无法一般性地限制 handler 大小"，说明帧上限不在该层统一实现；commit #1510 的 frame cap 需在传输层定位（本轮未定位到具体符号） | `conn.go:672` |
| IF-09 | `SSEHandler.ServeHTTP` 也实现了 DNS rebinding 防护（不只 Streamable HTTP） | `sse.go:238+` |

## 2. 与 Package 的逐条对比

### 2.1 CONFIRMED（14）

| Package 断言 | 独立核对 |
|---|---|
| 双协议栈 discover→fallback | ✅ `client.go` 分支一致 |
| 无状态每请求 ephemeral session | ✅ `connectStreamable` + defer Close |
| 旧协议合成 InitializeParams | ✅ `ephemeralConnectOpts` |
| `lookupSession` 403 user mismatch | ✅ |
| `checkInitialized` 恒 nil | ✅ 注释整段 |
| `initialized` 顺序/重复校验 | ✅ |
| 新协议禁止 server-initiated | ✅ |
| InputRequests / load-shedding | ✅ |
| EventStore 契约 | ✅ |
| `SessionClosed(context.TODO())` | ✅ |
| DNS rebinding 防护 | ✅（且 IF-09 补充 SSE 也有） |
| token 检查顺序 | ✅ |
| conformance baseline 5 个缺口 | ✅ |
| rough_edges 8 类 | ✅ |

### 2.2 PARTIALLY_CONFIRMED（6）

| # | Package 表述 | 修正 |
|---|---|---|
| P-01 | KO-02："无状态去掉了跨请求会话标识" | 应区分**传输层 session ID**（确实去掉）与**逻辑 session ID**（分布式示例仍可有，仅用于日志/业务）。KO-02 的 Statement 过窄。 |
| P-02 | EK-29/旧 SSE："保留旧 SSE 传输" | 应补充：旧 SSE **不宣告** ≥2026-07-28（`SupportsProtocolVersion` 返回 false），因此在 discover 中会被 transport 过滤掉。这正是 EK-03 的实例，但 Package 没把它连起来。 |
| P-03 | EK-01："任何非现代错误都会回退" | `Client.Connect` 的回退条件是"discover 失败且不是可协商的 unsupported-version"；对 `CodeUnsupportedProtocolVersion` 且带 Supported 列表会先重试，不是立即回退。表述需要更精确。 |
| P-04 | Project Layer："旧版 SSE 传输仍在" | 应补 `CloseSSEStream` 在新协议下 no-op（IF-04），说明"旧能力保留但部分主动行为被禁用"。 |
| P-05 | "OAuth authorization code 流程" | 应补 PKCE / DCR / token auth method 选择（IF-06/07），否则低估了 auth 包复杂度。 |
| P-06 | Snapshot："HEAD 在 v1.8.0 之后" | 方向正确，但 `docs/mcpgodebug.md` 的 1.9.0 段说明 HEAD 面向 v1.9.0；应写成"HEAD 已包含 1.9.0 文档条目，Release 仍为 v1.8.0"。 |

### 2.3 DOWNGRADED（1）

| # | 原表述 | 降级 |
|---|---|---|
| D-01 | KO-05 的 L4 "门禁的有效性取决于执行点" | 本项目只能证明"该门禁失效"，不能证明"所有门禁都必须由执行点强制"这一普遍模型；应降为 **L3 Pattern + 反例**，或保留 L4 但明确 `Cross-project validation pending`（Package 已标 pending，故仅需在标题语义上收紧）。 |

### 2.4 OVER_GENERALIZED（1）

| # | 原表述 | 问题 |
|---|---|---|
| O-01 | KO-08 "把已知缺口写成负向清单，是治理资产" | 本项目同时存在**未被任何清单覆盖**的缺口（许可证文件头 120 vs LICENSE）。"是治理资产"成立，但"负向清单能覆盖治理缺口"不成立；Package 边界 2 已提到，但 KO 标题仍偏普遍。建议标题改为"负向清单降低误用，但不保证治理一致性"。 |

### 2.5 MISSING（3）

| # | 遗漏 | 影响 |
|---|---|---|
| M-01 | `streamable_headers.go` 的 `Mcp-Method` / `Mcp-Name` / `Mcp-Param-*` header 映射（IF-01/02） | 这是 `mcp` 包内 604 行的核心机制，直接影响中间件可观测性/路由/限流；Package 的架构图与 EK 都没有它 → 覆盖缺口。 |
| M-02 | `SSEServerTransport.SupportsProtocolVersion` 把旧 SSE 排除在新协议外（IF-03） | 影响"双协议栈"的完整性：并非所有 transport 都支持新协议。 |
| M-03 | `CloseSSEStream` 在新协议 no-op（IF-04） / 旧 SSE 的 DNS rebinding 防护（IF-09） | 影响"弃用策略"与"安全边界"的完整性。 |

### 2.6 CONTRADICTED（0）

无直接与代码矛盾的断言。

### 2.7 NEEDS_HUMAN_REVIEW（2）

| # | 问题 | 原因 |
|---|---|---|
| H-01 | `internal/jsonrpc2` 的帧上限具体实现位置（commit #1510） | 浅克隆无历史；`conn.go:672` 只说明"无法一般性限制 handler 大小"。需 Owner 确认 frame cap 的真实符号/常量。 |
| H-02 | `checkInitialized` 注释的动机与是否计划恢复 | 只有代码注释自述"测试不稳定"，无 issue/PR 证据。 |

## 3. 判定统计

| 判定 | 数量 |
|---|---|
| CONFIRMED | 14 |
| PARTIALLY_CONFIRMED | 6 |
| DOWNGRADED | 1 |
| OVER_GENERALIZED | 1 |
| MISSING | 3 |
| CONTRADICTED | 0 |
| NEEDS_HUMAN_REVIEW | 2 |
| **合计** | **27** |

## 4. 三个"成功"（独立验证确认 Package 判断正确）

1. **双协议栈的结构判断**：独立读 `Client.Connect` 与 `Server.discover` 后，确认"探测优先 + 回退"是显式设计，不是偶发分支。
2. **无状态 = 请求级会话**：独立读 `serveStateless` 与 `ephemeralConnectOpts`，确认每请求建 session 并合成初始化状态；Package 的 KO-02 方向正确。
3. **缺口显性化**：独立读 `conformance/baseline.yml` 与 `docs/rough_edges.md`，确认官方确实维护负向清单；Package 的判断成立。

## 5. 三个"错误/不准确"（需在 Reconciliation 修正）

1. **KO-02 的"去掉跨请求会话标识"** → 改为"去掉传输层 `Mcp-Session-Id`；逻辑 session ID 仍可由部署方保留"（P-01）。
2. **"旧 SSE 保留"** → 补"旧 SSE 不宣告 ≥2026-07-28，会被 transport 过滤"（P-02/M-02）。
3. **"任何非现代错误都回退"** → 精确为"discover 失败且不属于可协商的 unsupported-version 时回退；可协商错误先重试"（P-03）。

## 6. 遗漏清单（供 Reconciliation 增补）

- M-01 `streamable_headers.go`：新增 EK（HTTP header 映射）与 1 条 Flow（Data/Evidence）。
- M-02 旧 SSE 版本排除：补入 EK-03/旧 SSE 段。
- M-03 `CloseSSEStream` no-op + SSE DNS 防护：补入弃用策略与 Authority Flow。

## 7. Benchmark Case（可复用的审计检查项）

**BC-01 "传输层的第二套元数据面"**：当一个协议同时有 body 与 HTTP header 两套参数承载方式时，审计必须检查：

1. header 是否与 body 一致（冲突时谁优先）；
2. 中间件能否在不解析 body 的情况下完成路由/限流；
3. 该能力是否有独立测试。

go-sdk 的 `streamable_headers.go` 满足 1（解码时校验）与 3（1,919 行测试）；2 是设计目的。该检查项建议加入 `knowledge-archaeology` 的 Coverage Auditor checklist（**不在本轮自动修改 skill**，仅作为 Benchmark case 记录）。

## 8. 审计结论

- **总体**：Package 的核心机制（协议迁移、请求级会话、身份绑定、门禁、重放、MRAT）判断可靠；**覆盖不完整**集中在"HTTP header 元数据面"与"旧 SSE 的版本排除"。
- **置信度**：核心结论高；`checkInitialized` 动机与 frame cap 位置为中等（需人工复核）。
- **处理**：按 P-01~P-06、M-01~M-03 在 corpus 的 Reconciliation 中修正；原 Package 不修改（保留审计痕迹）。

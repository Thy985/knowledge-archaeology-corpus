# 01 Project Layer — MCP 项目地图

> 全部事实来自 `docs/specification/2026-07-28/`、`schema/2026-07-28/schema.ts`、`seps/`、根治理文档（commit `ab3a39c`）。证据锚点：[T:x] = 规范正文段落，[S:x] = schema.ts 行号，[SEP:n] = SEP 编号。

## 1. 项目定位
Model Context Protocol（MCP）是 Anthropic 发起、现归 LF Projects 系列治理的**开放协议**，标准化 LLM 应用与外部数据源/工具之间的集成：应用（Host）内含客户端（Client），连接服务端（Server）提供的 Resources/Prompts/Tools 三类能力，全部消息走 JSON-RPC 2.0。[T:index.mdx Overview]

## 2. 架构：三参与者模型
```
Host（LLM 应用，发起连接）
 └─ Client（应用内连接器）
     │  JSON-RPC 2.0（stateless，每请求 _meta 自描述）
     ▼
 Server（提供上下文与能力：Resources / Prompts / Tools）
```
- 服务器三类原语的控制层级：[T:server/index.mdx]
  - **Prompts** = 用户控制（交互式模板，如斜杠命令）
  - **Resources** = 应用控制（上下文数据，如文件内容/git 历史）
  - **Tools** = 模型控制（可执行函数，如 API POST/文件写入）
- 客户端能力：Elicitation（主）/ Sampling / Roots（后两者 Deprecated）[T:index.mdx Key Details]

## 3. 核心抽象（schema.ts 为唯一事实源）
- **消息面**：`JSONRPCMessage`（request/notification 无 id）、`JSONRPCResponse`（result|error）[S:6-309]
- **元数据面**：`RequestMetaObject`（`_meta` 键：progressToken、io.modelcontextprotocol/protocolVersion（必需）、clientInfo、clientCapabilities（必需）、logLevel、subscriptionId、traceparent/tracestate/baggage）[T:basic/index.mdx _meta]
- **多态结果**：`ResultType = "complete" | "input_required" | string`；老服务器缺省视为 complete [S:216][T:basic/index.mdx ResultType]
- **MRTR 载体**：`InputRequests`（服务器请求 map，值=CreateMessageRequest|ListRootsRequest|ElicitRequest）、`InputResponses`、`InputRequiredResult` [S:537-584]
- **能力面**：`ClientCapabilities`/`ServerCapabilities`，均含 `extensions` map（带前缀 id → settings 对象）[S:716-793]
- **资源面**：`Resource`/`ResourceTemplate`/`ResourceContents`（Text|Blob）/`ResourceLink`/`EmbeddedResource` [S:1441-1550]
- **工具面**：`Tool`（inputSchema/outputSchema 2020-12 宽松化）/`ToolAnnotations`/`ToolChoice`/`CallToolResult` [S:1809-2029]
- **提示面**：`Prompt`/`PromptArgument`/`PromptMessage`（Role=user|assistant）[S:1659-1752]
- **发现面**：`DiscoverRequest/DiscoverResult`（supportedVersions/capabilities/instructions）[S:665-707]
- **订阅面**：`SubscriptionsListenRequest/Result`、`SubscriptionsAcknowledgedNotification`、`ResourceUpdatedNotification`、`SubscriptionFilter` [S:1270-1428]

## 4. 生命周期与状态模型
- **请求生命周期**：无状态——每请求自描述（版本/能力/身份在 `_meta`）；服务器不得依赖先前请求推断上下文 [T:basic/index.mdx Statelessness]
- **协议版本生命周期**：date-based versioning（YYYY-MM-DD）；历史 2024-11-05 → 2025-03-26 → 2025-06-18 → 2025-11-25 → **2026-07-28**（当前）→ draft [AGENTS.md Specification Versioning]
- **功能生命周期**：SEP-2596 三态 Active→Deprecated→Removed，最短 12 个月窗口；Deprecated 注册表（deprecated.mdx）：Roots/Sampling/Logging/DCR 2026-07-28 废弃最早 2027-07-28 移除；HTTP+SSE 2025-03-26 起废弃 [T:deprecated.mdx]
- **era 模型**：Modern（2026-07-28+ 每请求元数据）/ Legacy（≤2025-11-25 initialize 握手）/ Dual-era；era 是服务器属性非请求属性，客户端可缓存探测结果 [T:versioning.mdx Terminology]

## 5. 版本兼容机制（互操作矩阵节选）
| Client \ Server | Modern | Legacy | Dual-era |
|---|---|---|---|
| Modern | ✅ | ❌（可确定性失败） | ✅ |
| Legacy | ❌（无 fall-forward） | ✅ | ✅ |
| Dual-era | ✅ | ✅ | ✅ |
- stdio 探测 = 先发 `server/discover`，非现代错误 → 回退 initialize；HTTP = 现代请求 400 且非现代错误体 → 回退 [T:versioning.mdx Compatibility Matrix]

## 6. 传输绑定
- **stdio**：换行分隔 JSON-RPC over 标准流；客户端启动子进程；取消 = `notifications/cancelled` [T:transports/stdio.mdx]
- **Streamable HTTP**：单端点 POST；`Mcp-Method`/`Mcp-Name`/`X-MCP-Protocol-Version`/`x-mcp-header` 头路由；响应 = JSON 或 request-scoped SSE 流；取消 = 关闭响应流；Origin 校验防 DNS rebinding；`X-Accel-Buffering: no` + SSE 注释行 keep-alive；Last-Event-ID 不可恢复（断流丢 in-flight 请求，须新 id 重发）[T:transports/streamable-http.mdx]
- **custom transports**：MUST 保留 JSON-RPC + patterns + per-request metadata 模型；可靠字节流 SHOULD 复用 stdio framing [T:transports/index.mdx]

## 7. 错误处理模型
- 标准 JSON-RPC 错误码（-32700/-32600~-32603）
- **错误码分区政策**：-32000~-32019 legacy（禁新增、除 -32002 外不得假设含义）；**-32020~-32099 规范保留段**：-32020 HeaderMismatch / -32021 MissingRequiredClientCapability / -32022 UnsupportedProtocolVersion；旧码 -32002（resource not found→改 -32602）、-32042 保留不再用；应用自定错误在 JSON-RPC 保留区外分配 [T:basic/index.mdx Error Codes]

## 8. 配置与构建体系
- package.json scripts：`generate`（schema+SEPs）/`generate:schema`（TS→JSON+MDX）/`check`/`prep`（check+generate+format，PR 前无警告）[AGENTS.md]
- 质量门 = check:schema（TS/JSON/examples/MDX）+ check:docs + check:seps；CI 13 workflows 全治理类（main、markdown-format、render-seps、sep-lifecycle cron 周一 9AM、sep-reminder、cut-release、publish-release、deploy-blog、stage-blog、blog-preview、labeler、slash-commands）[.github/workflows/]

## 9. 权限与治理机制（Authority 层）
- **四级治理**（SEP-932）：Contributors（任意参与）→ Maintainers（组件负责，Core 任命）→ Core Maintainers（协议演进，bi-weekly 会议，多数否决权）→ Lead Maintainers（可否决一切，任免 Core，基建管理员）；成员制绑定个人防公司捕获 [SEP:932]
- **SEP 流程**（SEP-1850）：PR 编号=SEP 编号；`seps/{N}-{slug}.md` 为唯一事实源；sponsor 制；状态 Draft→In-Review→Final 由 PR labels + 文件 status 双轨同步；实施先于终稿（SEP-2484 conformance 要求）[SEP:1850]
- **AI 贡献政策**（AGENTS.md + AI_POLICY.md）：非维护者且 <3 合并 PR 的 AI agent 不得开 issue/PR；违规须带 `disclosure.txt`；AI 协助必须在 PR/issue 披露 [AGENTS.md][AI_POLICY.md]
- **安全边界**（SECURITY.md）：私密上报（Security Advisory）；Trust Model 声明 stdio 命令执行、服务器副作用为设计特性非漏洞 [SECURITY.md]
- **扩展治理**（SEP-2133）：propose→implement（SDK 参考实现必先行）→review→publish→adopt；扩展默认关闭需显式 opt-in；破坏性变更须新 identifier [SEP:2133][extensions/overview.mdx]

## 10. 外部依赖
Mintlify（文档站）/ Hugo（blog）/ TypeScript + Node 24（schema 生成）/ JSON Schema 2020-12（协议内嵌 schema 方言）/ OpenTelemetry（trace context 传播，W3C 格式）[T:basic/index.mdx OpenTelemetry][AGENTS.md]

## 11. 扩展体系快照
| 扩展 | identifier | 状态 | 核心机制 |
|---|---|---|---|
| Tasks | `io.modelcontextprotocol/tasks` | 官方扩展（2026-07-28 移出核心，SEP-2663） | tasks/get 轮询 + tasks/update + tasks/cancel；resultType:"task" |
| Skills over MCP | `io.modelcontextprotocol/skills` | 官方扩展（SEP-2640 Final） | skill:// URI + skills/list + skills/get + resources/directory/read |
| MCP Apps | `io.modelcontextprotocol/ui` | 官方扩展（SEP-1865） | 会话内渲染交互 UI（charts/forms/video） |
| Auth 族 | `io.modelcontextprotocol/oauth-client-credentials` 等 | 官方扩展 | OAuth2 client credentials / Enterprise-Managed Authorization |

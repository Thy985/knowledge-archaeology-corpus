# 04 Flow Atlas — MCP 七类流

> 全部从真实规范/代码导出（commit `ab3a39c`）。关键 Edge 可回溯 symbol/file/condition/state transition。

## F1 Control Flow（控制流）— 一次 tools/call 的现代请求生命周期
```
Client 构造 JSONRPCRequest(method=tools/call, id=N)
  → _meta 必填 [io.modelcontextprotocol/protocolVersion, clientCapabilities]（缺→-32602+400）
  → 传输绑定：stdio=换行 JSON | Streamable HTTP=POST + Mcp-Method/Mcp-Name 头
  → Server 校验：版本（不匹配→UnsupportedProtocolVersionError -32022）
                能力（缺→MissingRequiredClientCapabilityError -32021 + data.requiredCapabilities）
  → Server 处理：若需额外输入 → resultType:"input_required"（MRTR）
                 否则 → resultType:"complete" + _meta[serverInfo]
  → 取消路径：stdio=notifications/cancelled；HTTP=关闭响应流
```
证据：[S:268-309][T:basic/index.mdx Messages][T:transports/streamable-http.mdx]

## F2 State Flow（状态流）— 无状态 vs 状态引用
```
协议层无会话状态（无 Mcp-Session-Id，无 initialize）           [EK-01]
  ├─ 每请求状态：_meta（版本/能力/身份/logLevel）→ 请求内封闭
  ├─ 跨请求状态：显式 handle（服务器铸造，作工具参数传递）      [SEP:2567]
  ├─ 长流状态：subscriptions/listen 的流状态 scoped 到该请求，断流即失效重发 [EK-06]
  └─ era 状态：服务器属性（Modern/Legacy/Dual-era），客户端缓存探测结果 [EK-03]
功能生命周期状态机（SEP-2596）：
  Active → Deprecated（≥12 个月窗口）→ Removed（Core Maintainer 发布期决策）
  Deprecated 注册表 = 派生视图；per-feature 通知 + changelog = 规范记录      [EK-14]
```
证据：[T:basic/index.mdx Statelessness][T:versioning.mdx][T:deprecated.mdx]

## F3 Data Flow（数据流）— 三类原语
```
Resources（应用控制，context/data）
  resources/list（分页 Cursor）→ ListResourcesResult（CacheableResult ttlMs/cacheScope）
  resources/read → TextResourceContents | BlobResourceContents | ResourceLink | EmbeddedResource
Prompts（用户控制，模板）
  prompts/list → Prompt[]（name/description/arguments）
  prompts/get → PromptMessage[]（role=user|assistant；content=Text|Image|Audio|ToolUse|ToolResult）
Tools（模型控制，函数）
  tools/list → Tool[]（inputSchema/outputSchema 2020-12；annotations；确定性排序）
  tools/call → CallToolResult（content[] + structuredContent + isError）
变更通知：subscriptions/listen（toolsListChanged/promptsListChanged/resourcesListChanged/resourceSubscriptions）
缓存：list/read 结果带 ttlMs/cacheScope；server/discover 结果可缓存（3600000 示例）
```
证据：[S:1121-1244][S:1566-1650][S:1767-1848][T:server/{resources,prompts,tools}.mdx][SEP:2549]

## F4 Evidence Flow（证据流）— 规范事实源与决策记录
```
唯一事实源：schema/2026-07-28/schema.ts（TS 定义）
  → npm run generate:schema → schema.json（JSON Schema 分发）+ schema.mdx（Schema Reference 文档）
  → npm run check:schema（TS/JSON/examples/MDX 校验）[AGENTS.md]
决策记录：seps/{N}-{slug}.md（44 个 SEP；PR 编号=SEP 编号）[SEP:1850]
  Status 字段（Draft→In-Review→Final）+ PR labels 双轨同步
  sponsor 制：Core/Maintainer 管理状态转移与 Core Maintainer 会议呈报
规范变更须走 SEP（Extensions Track 需先有 SDK 参考实现，SEP-2484 conformance）
```
证据：[AGENTS.md][SEP:1850][SEP:932][SEP:2484]

## F5 Authority Flow（权威流）— 权限与决策权威
```
LF Projects 系列（商标政策/治理变更双批准）          [GOVERNANCE.md]
  └─ Lead Maintainers（2 人，可否决一切，任免 Core，基建管理员）   [SEP:932][MAINTAINERS.md]
      └─ Core Maintainers（6 人，协议演进，bi-weekly 决策，多数可否决 Maintainer）
          └─ Maintainers（组件/仓库负责，Core 任命）
              └─ Contributors（任何人，issue/PR/讨论）
成员制绑定个人（防公司捕获）——治理决策优先协议完整性
扩展决策权威：extension repo maintainers 独立于 Core 发布扩展更新（向后兼容要求保留）
```
证据：[GOVERNANCE.md][SEP:932][MAINTAINERS.md][extensions/overview.mdx Evolution]

## F6 Memory Flow（记忆流）— 版本历史与决策留痕
```
date-based versioning：schema/[YYYY-MM-DD]/ + docs/specification/[YYYY-MM-DD]/ + docs/docs/[YYYY-MM-DD]/
changelog.mdx：2025-11-25→2026-07-28 全部变更（9 大项 + 12 minor + 4 deprecated + 治理/流程变更）
废弃注册表 deprecated.mdx：derived view（per-feature 通知 + changelog 为规范记录）
SEP 文件保留完整决策历史（Abstract/Motivation/Specification/Backwards Compatibility/Reference Implementation）
docs/docs/draft/：in-progress 工作（版本化草稿）
```
证据：[AGENTS.md][T:changelog.mdx][T:deprecated.mdx][SEP:1850]

## F7 Policy Flow（策略流）— 治理闭环
```
决策（SEP 提案，PR）→ 批准（sponsor + Core Maintainers 会议 + LF 双重批准）
  → 固化（规范正文 + schema.ts + changelog + deprecated 注册表）
  → 执行（CI 检查 check:docs/schema/seps + SEP 生命周期自动化 cron 周一 9AM UTC + labeler）
  → 未来决策（新 SEP 引用旧 SEP 决策——如 SEP-2663 显式依赖 SEP-2260/2322/2243/2567/2575）
AI 贡献政策闭环：AGENTS.md 门槛（<3 PR 禁开）+ AI_POLICY.md 披露要求 + disclosure.txt 强制
信任模型闭环：SECURITY.md "Intended Behaviors" 声明 → 安全研究聚焦 → 治理更新
```
证据：[.github/workflows/sep-lifecycle.yml][SEP:2663][AGENTS.md][AI_POLICY.md][SECURITY.md]

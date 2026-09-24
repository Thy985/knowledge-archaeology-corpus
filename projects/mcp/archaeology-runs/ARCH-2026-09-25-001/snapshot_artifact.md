# Repository Snapshot Artifact — ARCH-2026-09-25-001

> 阶段③ Repository Snapshot 产出。事实均来自浅克隆仓库实际内容，可追溯。

## 1. 快照信息

| 项 | 值 |
|---|---|
| repository | https://github.com/modelcontextprotocol/modelcontextprotocol.git |
| commit SHA | `ab3a39c13bd23be691c2760e1c6c5c15a64582e1`（HEAD of main） |
| branch | main |
| 协议版本 | 2026-07-28（LATEST_PROTOCOL_VERSION，`schema/2026-07-28/schema.ts:30`） |
| 快照时间 | 2026-09-25（clone 当日） |
| 远端活跃 | pushed_at 2026-09-24T01:39:54Z（GitHub API） |
| 仓库规模 | 956 文件（git ls-files） |
| 许可证 | MIT（README/LICENSE）＋ Apache-2.0（规范贡献，GOVERNANCE.md） |

## 2. 项目基础地图

### 语言与构建
- **TypeScript**：协议 schema 以 `schema/<version>/schema.ts` 为**唯一事实源**（`schema/2026-07-28/schema.ts`，3197 行）；`npm run generate:schema` 生成 `schema.json` 与 `schema.mdx`（AGENTS.md）
- **Markdown/MDX**：规范正文 `docs/specification/2026-07-28/`（33 文件）；文档站 Mintlify 构建（`npm run serve:docs`）；blog 用 Hugo（`npm run serve:blog`）
- **JSON Schema 2020-12**：协议内嵌 schema 默认方言（basic/index.mdx "Schema Dialect"）

### 入口
- 逻辑入口：`schema/2026-07-28/schema.ts`（协议类型定义）；`docs/specification/2026-07-28/index.mdx`（规范正文索引）；`README.md`（仓库导航）
- 协议 RPC 面（schema.ts 方法清单）：`tools/list`、`tools/call`、`resources/list`、`resources/read`、`prompts/list`、`prompts/get`、`completion/complete`、`server/discover`、`subscriptions/listen`、`elicitation/create`、`roots/list` + notifications（cancelled/progress/message）

### 核心模块
| 路径 | 职责 |
|---|---|
| `schema/2026-07-28/` | 权威协议 schema（TS 源 + JSON + examples/） |
| `docs/specification/2026-07-28/` | 规范正文（architecture/basic/server/client/deprecated/changelog） |
| `docs/extensions/` | 扩展（apps/auth/skills/tasks + overview + client-matrix） |
| `seps/` | 44 个 SEP（Specification Enhancement Proposal，即本仓库 ADR 体系） |
| `docs/docs/` | 教程/指南（按协议版本分目录 2024-11-05→2026-07-28→draft） |
| `blog/` | 官方博客（Hugo） |
| `docs/community/` | 治理/工作组成员/SEP 指南 |
| `.github/workflows/` | 13 个 CI workflow（治理自动化） |

### 核心数据结构（schema.ts）
- `JSONRPCMessage` / `Request` / `JSONRPCRequest` / `Notification` / `Result` / `Error`
- `ResultType` = `"complete" | "input_required" | string`（多态结果判别）
- `InputRequiredResult` / `InputRequests` / `InputResponses`（MRTR 载体）
- `RequestMetaObject`（`_meta`：protocolVersion/clientInfo/clientCapabilities/logLevel）
- `CacheableResult`（`ttlMs`/`cacheScope`，list 缓存）
- `DiscoverRequest`/`DiscoverResult`（supportedVersions/capabilities/instructions）
- `Tool`/`ToolAnnotations`、`Resource`/`ResourceTemplate`、`Prompt`/`PromptMessage`、`ElicitRequest`（form/url 两种模式）
- 错误码：标准 JSON-RPC（-32700/-32600~-32603）+ 规范保留段（-32020 HeaderMismatch / -32021 MissingRequiredClientCapability / -32022 UnsupportedProtocolVersion）

### 状态与生命周期
- 协议版本：date-based versioning（YYYY-MM-DD，非 semver）；历史版本 2024-11-05/2025-03-26/2025-06-18/2025-11-25/2026-07-28 + draft
- **Feature lifecycle**（SEP-2596）：Active / Deprecated / Removed 三态，最短 12 个月废弃窗口；Deprecated 注册表见 `deprecated.mdx`（Roots/Sampling/Logging/DCR 于 2026-07-28 废弃，最早 2027-07-28 移除；HTTP+SSE 自 2025-03-26 废弃）
- 现代/旧版/双时代（Modern/Legacy/Dual-era）互操作矩阵（versioning.mdx）

### 测试体系
- **无单元/集成测试**（协议规范仓库，无 runtime 代码）
- 质量门 = `npm run check`（check:schema TS/JSON/examples/MDX + check:docs 格式/链接 + check:seps）+ `npm run prep`（check+generate+format，AGENTS.md 要求 push/PR 前无警告）
- CI：13 workflows 全部治理类——main（node 24 检查）、markdown-format、render-seps、sep-lifecycle（周一 9AM UTC cron 自动化 SEP 状态）、sep-reminder、sep-lifecycle-manual、cut-release、publish-release、deploy-blog、stage-blog、blog-preview、labeler、slash-commands

### 配置
- `package.json`（npm scripts：generate/format/check/prep）、`tsconfig.json`、`eslint.config.mjs`、`typedoc.config.mjs`、`typedoc.plugin.mjs`、`migrate_seps.js`

### 权限与治理机制
- **GOVERNANCE.md**：LF Projects 系列成员；贡献者层级 Contributors→Maintainers→Core Maintainers→Lead Maintainers；双 Lead Maintainer 可否决；Core Maintainer bi-weekly 会议；个人成员制（防公司捕获）
- **MAINTAINERS.md**（2026-08-05 更新）：Lead（dsp-ant/localden）+ 6 Core + Emeritus + 各 SDK/Project/Working Group 维护者
- **SEP 流程**（SEP-932 + SEP-1850）：PR 编号 = SEP 编号；`seps/{N}-{slug}.md`；sponsor 制；状态经 Draft→In-Review→Final 由 PR labels 同步
- **AGENTS.md**：AI agent 贡献政策——非维护者且 <3 个合并 PR 不得开 issue/PR；违规提交必须带 `disclosure.txt`
- **AI_POLICY.md**：AI 协助必须在 PR/issue 披露（作用范围覆盖 org 全部仓库）
- **SECURITY.md**：GitHub Security Advisory 私密上报；"Intended Behaviors and Trust Model" 声明 stdio 命令执行/服务器副作用等为设计特性非漏洞
- **CONTRIBUTING.md / CODE_OF_CONDUCT.md / ANTITRUST.md**

### 外部依赖
- Mintlify（文档站）、Hugo（博客）、TypeScript/Node 24（schema 生成与检查）、JSON Schema 2020-12（协议内嵌 schema 方言）、OpenTelemetry（trace context 传播约定，`_meta` 键 traceparent/tracestate/baggage）

## 3. 仓库类型判定
**协议规范库（spec-first repository）**：无运行时源码、无测试体系；认知资产集中在 schema（TS 事实源）+ 规范正文 + 44 SEP（决策记录）+ 治理文档。与 A2A 考古（2026-09-24，ARCH-2026-09-24-001）同形态，可直接对照。

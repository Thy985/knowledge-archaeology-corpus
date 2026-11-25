# 01 · Project Layer — chrome-devtools-axi

## 项目定位

`chrome-devtools-axi` 是一个面向 agent 的浏览器自动化 CLI。它不是新的浏览器实现，而是 `chrome-devtools-mcp` 的 AXI 兼容 wrapper：把上游 MCP 工具重新组织成短命令、TOON 输出和上下文建议。`VISION.md` 明确两条边界：追求与 `chrome-devtools-mcp` 的功能对等；不添加上游没有的浏览器能力。

- 包版本：`0.1.46`；HEAD 比 tag `chrome-devtools-axi-v0.1.46` 多 2 个 commit。
- 技术栈：Node.js ≥20、TypeScript、ESM、pnpm、vitest。
- 体量：78 个 tracked files，50 个 `.ts`，17,051 行 TS，21,088 行文本。
- 测试：29 个测试文件，654 个静态 `it(` 站点，**796 个执行用例全部通过**（2026-11-26，`pnpm test`，80.57s）。
- 构建：`pnpm run build` 通过（`tsc` + chmod）。
- CI：Node 24，`pnpm install --frozen-lockfile` → `pnpm run build` → `pnpm test`。

## 架构图

```
Agent / shell
    │  chrome-devtools-axi <command>
    ▼
CLI（短命进程，src/cli.ts）
    │  HTTP POST /call, GET /health, GET /tools
    ▼
Bridge（持久 detached 进程，src/bridge.ts）
    │  stdio  或  Streamable HTTP
    ▼
chrome-devtools-mcp
    │  CDP
    ▼
Chrome（由 AXI 启动，或 attach 到外部 Chrome）
```

默认 session 监听 `127.0.0.1:9224`。bridge 维护一条持久 MCP session；CLI 每次只做参数解析、调用和格式化。

## 入口

| 入口 | 文件 | 说明 |
|---|---|---|
| CLI | `bin/chrome-devtools-axi.ts` → `src/cli.ts` `main()` | 36 个命令，见 `COMMAND_HANDLERS` |
| Bridge | `bin/chrome-devtools-axi-bridge.ts` → `src/bridge.ts` `runBridge()` | 先连 MCP transport，再 `listen(port, "127.0.0.1")` |
| 脚本运行 | `src/run.ts` `runScript()` | stdin → 临时 `.mjs` → 动态 import，注入 `page` 全局 |
| 构建/发布 | `scripts/build-skill.ts`、release-please | 生成 `skills/`、`CHANGELOG.md`、版本清单 |

## 模块责任

| 模块 | 责任 | 关键符号 |
|---|---|---|
| `src/cli.ts` | 命令注册、flag 校验、输出格式、退出码 | `COMMAND_HANDLERS`、`COMMANDS`、`main` |
| `src/client.ts` | CLI→bridge、bridge 发现/健康/终止/启动、`callTool`、错误映射 | `ensureBridge`、`checkBridgeHealth`、`terminateBridgeProcess`、`postTool`、`CdpError` |
| `src/bridge.ts` | HTTP 路由、安全 gate、MCP transport、roots 协商、keychain 参数、idle、工具转发 | `isRequestAllowed`、`handleBridgeRequest`、`handleCallRequest`、`resolveTransport`、`buildTransportArgs` |
| `src/sessions.ts` | session 名称校验、端口推导、状态目录 | `resolveSessionName`、`resolveSessionPort`、`resolveSessionStateDir` |
| `src/selected-page.ts` | 页面路由身份持久化与解析 | `rememberToolRouting`、`createdPageIdFromNewPageDump`、`clearSelectedPageId` |
| `src/pages.ts` | `list_pages` 展示解析；page-scoped 工具集合 | `PAGE_SCOPED_TOOLS`、`needsPageId` |
| `src/snapshot.ts` | 快照截断与 uid generation 标记 | `truncateSnapshot`、`stampSnapshotGeneration` |
| `src/uid-freshness.ts` | ref 新鲜度校验、fresh snapshot | `parseUidFresh`、`captureFreshSnapshot` |
| `src/generation.ts` | generation 计数器 + 页面 MutationObserver | `bumpGeneration`、`getCurrentGeneration` |
| `src/navigation-timeout.ts` | 嵌套 deadline 计算 | `mcpRequestTimeoutMs`、`callRequestTimeoutMs` |
| `src/idle.ts` | opt-in idle 关闭 | `resolveIdleTimeoutMs`、`createIdleTracker` |
| `src/run.ts` | `page` helper、JS 包装、临时脚本 | `runScript`、`createPageHelper`、`wrapJsExpression` |
| `src/paths.ts` | 输出路径按 CLI cwd 解析 | `resolveOutputPath` |

## 关键数据结构

- `CdpError`：`message`、`code`（`BRIDGE_NOT_READY` / `REF_NOT_FOUND` / `STALE_REF` / `TIMEOUT` / `BROWSER_ERROR` / `VALIDATION_ERROR` / `UNKNOWN`）、`suggestions`。
- `bridge.pid`：bridge PID + 端口，跨 CLI 进程复用。
- `selected-page-id`：上一次 `select_page` / `new_page` 的页面 id。
- `snapshot-generation`：每抓一次快照递增的整数。
- `PAGE_SCOPED_TOOLS`：需要注入 `pageId` 的 25 个工具集合。
- `KEYCHAIN_ISOLATION_CHROME_ARGS`：`--use-mock-keychain`、`--password-store=basic`。

## 生命周期

1. CLI 读取 session 名称，计算端口与状态目录。
2. `ensureBridge` 读 `bridge.pid`；PID 活着则 `GET /health?deep=1`。
3. deep health 通过 → 复用；失败 → `terminateBridgeProcess`（SIGTERM 等待 2s，必要时进程组 SIGKILL），清空 selected page，spawn 新 bridge。
4. 新 bridge 先连 MCP，再 bind 127.0.0.1；写 PID，输出 `READY`。
5. 每次 `/call` 先过 Host/Origin gate，再转发 MCP；MCP `isError` 转成 `{error}`。
6. 空闲时默认常驻；设置了 `IDLE_TIMEOUT` 才在无活动、无 in-flight 时退出。
7. bridge 收到 SIGTERM/SIGINT，或 MCP stdio 连接关闭，执行 shutdown，删除 PID，杀进程组。

## 配置面

| 配置 | 默认/范围 | 作用 |
|---|---|---|
| `CHROME_DEVTOOLS_AXI_SESSION` | `default` | 选择 session；命名 session 端口 9225..10224 |
| `CHROME_DEVTOOLS_AXI_PORT` | session 推导 | 显式覆盖端口；冲突 exit 48 |
| `CHROME_DEVTOOLS_AXI_BRIDGE_TIMEOUT_MS` | 30,000；最小 1,000 | bridge 启动等待 |
| `CHROME_DEVTOOLS_AXI_NAVIGATION_TIMEOUT_MS` | 10s 上游默认；最大 1e9 | `open`/`newpage` 导航超时 |
| `CHROME_DEVTOOLS_AXI_IDLE_TIMEOUT_MS` | 0（关闭）；最小 1,000 | bridge 空闲自关 |
| `CHROME_DEVTOOLS_AXI_MCP_PATH` | 自动探测 | 指定 `chrome-devtools-mcp` 可执行文件 |
| `CHROME_DEVTOOLS_AXI_MCP_SERVER_URL` | 空 | 共享 MCP 服务地址 |
| `CHROME_DEVTOOLS_AXI_USER_DATA_DIR` | 空 | 持久 profile；触发 launch 模式 |
| `CHROME_DEVTOOLS_AXI_AUTO_CONNECT` | 空 | attach 到用户 Chrome |
| `CHROME_DEVTOOLS_AXI_BROWSER_URL` | 空 | attach 到已运行 Chrome / ws endpoint |

## 权限与治理

- 网络面：bridge 只 bind `127.0.0.1`；Host/Origin gate 拒绝非 loopback 的 Host 或 Origin。
- 认证面：**没有 token/secret**。`POST /call` 直接映射到 MCP `callTool`，任何能连到该端口的本地进程都拥有该 bridge 的工具权限。
- 文件面：MCP roots 由调用方请求的 cwd + 输出路径祖先目录组成；请求体里的 `roots` 是任意绝对路径数组，没有服务端 allowlist。
- 浏览器隔离：AXI 自己启动浏览器时加 keychain 隔离 flag；attach 模式不加。
- 会话隔离：session 只隔离 bridge 进程/端口/状态文件；多个 attach session 共享同一个外部浏览器，浏览器状态不隔离。
- 发布治理：CI + airlock lint/review gate + generated-files guard + no-mistakes PR attestation + release-please provenance。

## 外部依赖

- `@modelcontextprotocol/sdk`：MCP client/transport。
- `@toon-format/toon`：结构化输出编码。
- `axi-sdk-js`：CLI 运行时与错误类型。
- `chrome-devtools-mcp`：外部可执行文件，默认 `npx -y chrome-devtools-mcp@latest`，或 PATH/global/explicit path。
- Chrome：由 MCP 启动或 attach。

## 边界

本 Package 只覆盖仓库代码、文档、测试与 git 历史。没有启动真实 Chrome，没有执行 live CDP 操作，也没有对运行中的 bridge 发起攻击请求。所有 runtime 行为结论来自测试（S4）或代码阅读（S3）；真实浏览器交互仍是未执行项。

# 04 · Flow Atlas — 7 个真实流

> 42 条边全部从仓库代码导出；`status: verified` 表示**静态代码/测试已验证**，不表示运行时已执行。本轮没有启动真实 Chrome，也没有 live `/call` 攻击。

## 总览

| Flow | Edges | Runtime verified |
|---|---|---|
| Control Flow (`control`) | 8 | false |
| State Flow (`state`) | 7 | false |
| Data Flow (`data`) | 7 | false |
| Evidence Flow (`evidence`) | 7 | false |
| Authority Flow (`authority`) | 7 | false |
| Memory Flow (`memory`) | 6 | false |
| Policy Flow (`policy`) | 6 | false |

## 关键路径读数

- 控制链：`CLI → ensureBridge → deep health → (复用|spawn) → /call → callTool`。8 条边，gate 在第 6 条前。
- 权威链：`HTTP → Host/Origin gate → /call → roots → MCP tool`。gate 只验证网络来源，`/call` 仍无 caller auth。
- 证据链：`snapshot → stamp gN → MutationObserver → action → freshness`。stale 时 fail-closed。
- 状态链：三份文件状态 + 一份 identity epoch；缺一即静默错。
- 策略链：CI/airlock/guard/release 四个 gate 共同定义“什么能进 main / 什么能发布”。

## Control Flow

从短命 CLI 到 MCP tool 的控制链。关键分支是“复用 vs 重启”和“gate 通过 vs 403”。

| Edge | From → To | Condition | Transition | Source | Evidence |
|---|---|---|---|---|---|
| `F-C01` | CLI main → COMMAND_HANDLERS dispatch | argv 首个 token 是已知命令 | argv -> handler | `src/cli.ts:1997-2010` | EV-041 |
| `F-C02` | CLI command → ensureBridge | 每次需要浏览器会话的命令 | CLI 进程 -> 复用或新建 bridge | `src/client.ts:473-510` | EV-004, EV-005 |
| `F-C03` | ensureBridge → GET /health?deep=1 | PID 活着且需要确认 CDP target | bridge 进程 -> 探测结果 | `src/bridge.ts:557-580` | EV-005, EV-006 |
| `F-C04` | ensureBridge → spawnBridgeProcess | 没有可复用 bridge 或 deep health 失败 | 无 bridge -> detached bridge | `src/client.ts:352-370` | EV-001, EV-010 |
| `F-C05` | CLI HTTP client → POST /call | bridge ready | 请求 -> bridge 路由 | `src/client.ts:744-790` | EV-002, EV-026 |
| `F-C06` | handleBridgeRequest → isRequestAllowed | 每个 /health /tools /call 请求 | 请求 -> 放行/403 | `src/bridge.ts:520-546` | EV-015, EV-016 |
| `F-C07` | handleCallRequest → client.callTool | gate 通过且 body 合法 | HTTP 请求 -> MCP tool 调用 | `src/bridge.ts:476-520` | EV-021 |
| `F-C08` | client.onclose → shutdown | MCP 连接关闭且非自身 shutdown | 运行 -> 退出 | `src/bridge.ts:1248-1251` | EV-009, EV-046 |

## State Flow

跨命令状态：PID/port、selected page、generation、identity epoch。每条边都改变下一次命令可观察的行为。

| Edge | From → To | Condition | Transition | Source | Evidence |
|---|---|---|---|---|---|
| `F-S01` | ensureBridge → bridge.pid | 启动/复用 bridge | 无状态 -> PID+port 状态 | `src/client.ts:473-510` | EV-003, EV-011 |
| `F-S02` | select_page/new_page/close_page → selected-page-id | 显式 select 或 new dump 唯一 [selected] 或 close 当前 | selection 空/旧 -> 新/空 | `src/selected-page.ts:109-125` | EV-027, EV-029 |
| `F-S03` | captureFreshSnapshot → bumpGeneration | 每次抓快照 | generation N -> N+1 | `src/uid-freshness.ts:81-84` | EV-033, EV-035 |
| `F-S04` | MCP tool result → didMcpPageIdentityChange | reconnect marker 出现在 structuredContent 或首行 | identity epoch 未变 -> 已变 | `src/bridge.ts:360-410` | EV-031 |
| `F-S05` | bridge call handler → PAGE_IDENTITY_CHANGED_ERROR | pageIdentityChanged && args.pageId | 旧 id -> 明确错误 | `src/bridge.ts:470-520` | EV-031, EV-032 |
| `F-S06` | ensureBridge → clearSelectedPageId | respawn 新 bridge 前 | 保留 selection -> 清空 | `src/client.ts:473-510` | EV-005 |
| `F-S07` | parseUidFresh → getPageRefState | ref 带 gN 前缀 | 未校验 -> generation/mutations 判定 | `src/uid-freshness.ts:118-140` | EV-035 |

## Data Flow

参数、roots、MCP 结果、错误和文件路径的数据变换。注意 `/call` 没有通用 schema 校验。

| Edge | From → To | Condition | Transition | Source | Evidence |
|---|---|---|---|---|---|
| `F-D01` | CLI command args → JSON request body | 命令执行 | argv/对象 -> JSON | `src/client.ts:744-790` | EV-002, EV-026 |
| `F-D02` | handleCallRequest → JSON.parse body | POST /call | 字节流 -> name/arguments/roots | `src/bridge.ts:476-520` | EV-019 |
| `F-D03` | /call roots → applyRootsNow | roots 数组且绝对路径 | 请求字段 -> MCP roots 切换 | `src/bridge.ts:428-441` | EV-019, EV-024 |
| `F-D04` | handleCallRequest → client.callTool(name, args) | gate 通过 | HTTP -> MCP JSON-RPC | `src/bridge.ts:476-520` | EV-018 |
| `F-D05` | MCP tool result → isToolResultError | result.isError === true | MCP 成功响应 -> bridge error | `src/bridge.ts:330-350` | EV-026 |
| `F-D06` | parseCallResponse → CdpError | 响应含 error 字段 | HTTP 200 -> 非零退出 | `src/client.ts:790-850` | EV-026, EV-032 |
| `F-D07` | CLI output arg → resolveOutputPath | screenshot/network/perf/heap/lighthouse 等文件参数 | 相对路径 -> 绝对路径 | `src/paths.ts:7-9` | EV-025 |

## Evidence Flow

快照证据链：页面 -> uid stamp -> action -> freshness gate -> 响亮失败。

| Edge | From → To | Condition | Transition | Source | Evidence |
|---|---|---|---|---|---|
| `F-E01` | snapshot command → capture snapshot | snapshot tool 调用 | 页面 -> 文本快照 | `src/bridge.ts:591-605` | EV-033, EV-037 |
| `F-E02` | raw snapshot → stampSnapshotGeneration | 含 uid= 行 | uid=id -> uid=gN:id | `src/snapshot.ts:51-60` | EV-036 |
| `F-E03` | snapshot injected script → MutationObserver | 页面加载/抓取时 | 无 observer -> 计数 mutations | `src/uid-freshness.ts:45-60` | EV-035 |
| `F-E04` | click/fill/type → uid ref | 命令带 uid 参数 | ref 文本 -> MCP args | `src/cli.ts:1860-1900` | EV-034 |
| `F-E05` | parseUidFresh → generation + mutations check | 每次 uid action 前 | ref -> fresh/stale | `src/uid-freshness.ts:118-140` | EV-035 |
| `F-E06` | throwStaleRef → STALE_REF error | 检查失败 | fresh -> loud error | `src/uid-freshness.ts:118-140` | EV-035 |
| `F-E07` | truncateSnapshot → line-boundary truncation | 长度 > 16000 且未 --full | 完整树 -> 截断树+metadata | `src/snapshot.ts:102-115` | EV-037 |

## Authority Flow

谁能触发、能触达什么权限。Host/Origin 与 caller identity 是两个不同边界。

| Edge | From → To | Condition | Transition | Source | Evidence |
|---|---|---|---|---|---|
| `F-A01` | HTTP request → isRequestAllowed | 任意路由 | 未认证请求 -> 允许/拒绝 | `src/bridge.ts:303-320` | EV-015 |
| `F-A02` | isRequestAllowed false → 403 Forbidden host | Host/Origin 非 loopback 或不可解析 | 请求 -> 403 + 日志 | `src/bridge.ts:531-546` | EV-016 |
| `F-A03` | isRequestAllowed true → handleCallRequest | 允许的 /call | 403 风险 -> 工具执行 | `src/bridge.ts:591` | EV-021 |
| `F-A04` | handleCallRequest roots → applyRootsNow | body.roots 非空 | caller roots -> MCP roots | `src/bridge.ts:1098-1155` | EV-024 |
| `F-A05` | buildTransportArgs → keychain flags | launch(--isolated/--userDataDir) vs attach | launch -> mock keychain；attach -> 不加 | `src/bridge.ts:683-760` | EV-020, EV-021 |
| `F-A06` | resolveSessionPort → session port/state | 命名 session 或显式 PORT | default 9224 -> hash 9225..10224 | `src/sessions.ts:94-122` | EV-011, EV-012 |
| `F-A07` | client.callTool → MCP tool privilege | 任意允许的 tool name | HTTP caller -> 浏览器/CDP 权限 | `src/bridge.ts:591` | EV-018, EV-021 |

## Memory Flow

文件即控制面内存：bridge.pid、selected-page-id、snapshot-generation、session 目录。

| Edge | From → To | Condition | Transition | Source | Evidence |
|---|---|---|---|---|---|
| `F-M01` | spawnBridgeProcess → write bridge.pid | spawn 成功 | 无文件 -> PID+port | `src/client.ts:352-370` | EV-003 |
| `F-M02` | ensureBridge → read bridge.pid | 每次命令 | 文件 -> pid/port | `src/client.ts:473-510` | EV-003 |
| `F-M03` | rememberToolRouting → write selected-page-id | select/new/close | selection -> 文件 | `src/selected-page.ts:109-125` | EV-027 |
| `F-M04` | bumpGeneration → write snapshot-generation | 每次 capture | N -> N+1 文件 | `src/generation.ts:34-47` | EV-033 |
| `F-M05` | session state dir → ~/.chrome-devtools-axi/ | 默认/命名 session | 无目录 -> 目录 | `src/sessions.ts:110-122` | EV-011 |
| `F-M06` | shutdown → unlink bridge.pid | bridge 退出 | PID 文件 -> 删除 | `src/bridge.ts:1232-1270` | EV-009, EV-046 |

## Policy Flow

仓库治理：CI、airlock、生成文件 guard、release provenance。

| Edge | From → To | Condition | Transition | Source | Evidence |
|---|---|---|---|---|---|
| `F-P01` | push/PR → ci.yml build+test | Node 24 + pnpm | 变更 -> CI 结果 | `/.github/workflows/ci.yml:1-30` | EV-042 |
| `F-P02` | local dev → airlock lint | 手动/agent 提交前 | 源码 -> prettier+tsc 结果 | `/.airlock/lint.sh` | EV-043 |
| `F-P03` | PR diff → guard-generated-files | base...head 改 CHANGELOG/manifest | 手改 -> fail | `/.github/workflows/guard-generated-files.yml:1-60` | EV-048 |
| `F-P04` | human PR → no-mistakes-required | 非 release-please 作者 | 缺 attestation -> fail | `/.github/workflows/no-mistakes-required.yml:1-60` | EV-048 |
| `F-P05` | main release → release-please + npm provenance | release PR 合并 | 版本 -> npm publish | `/.github/workflows/release-please.yml:1-60` | EV-049 |
| `F-P06` | release PR → paths-ignore | release-please 输出集 | release PR -> 跳过重复 CI | `/.github/workflows/ci.yml:1-30` | EV-042, EV-049 |

## 边界与未覆盖

- 没有真实 Chrome、没有 live CDP、没有运行中的 bridge；所有“运行”结论只来自测试代码/本地 `pnpm test` 的间接证据。
- HTTP/2 `:authority`、重复 `Host` 头、代理改写、IPv6 zone id 未测。
- 跨用户/跨 UID 的 loopback 可达性未测；不同 OS 的防火墙/套接字权限未测。
- `isBridgeProcess` 的 `ps` 子串身份、PID 复用、命令伪装未做运行时攻击。
- `roots` 的 MCP 侧实际文件访问范围未在 live MCP 上测。

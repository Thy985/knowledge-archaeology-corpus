# 06 · Evidence Index

> 本索引只记录**可回到仓库实际文件/命令**的证据。`runtime` 仅表示本机执行过测试；不代表 live Chrome/攻击验证。

| ID | Source | Observation | Confidence | Status | Supports |
|---|---|---|---|---|---|
| `EV-001` | `README.md:117-160; AGENTS.md invariants` | CLI 是短命令进程，跨命令会话由 detached bridge 持有；session 只隔离 bridge。 | high | verified | EK-01, EK-10 |
| `EV-002` | `src/bridge.ts:6-9, 520-605; src/client.ts:744-790` | bridge 暴露 /health /health?deep=1 /tools /call；CLI 通过 HTTP 调用。 | high | verified | EK-02, EK-01 |
| `EV-003` | `src/bridge.ts:1225` | server.listen(port, "127.0.0.1")，只 bind loopback。 | high | verified | EK-02, EK-03 |
| `EV-004` | `src/client.ts:192-202, 473-510` | checkBridgeHealth({deep}) 请求 /health?deep=1，并作为复用前置。 | high | verified | EK-04, EK-05 |
| `EV-005` | `src/bridge.ts:557-580` | deep health 调用 isBridgeTargetReachable（list_pages）验证 CDP target。 | high | verified | EK-04, EK-05 |
| `EV-006` | `src/bridge.ts:570-580` | deep 失败返回 503 CDP target unreachable + reason。 | high | verified | EK-04 |
| `EV-007` | `src/client.ts:269-279` | isBridgeProcess 用 ps -p <pid> -o command= 并做子串匹配。 | high | verified | EK-07 |
| `EV-008` | `src/client.ts:288-330` | terminateBridgeProcess 先 SIGTERM，2s 后对进程组 SIGKILL。 | high | verified | EK-05, EK-07 |
| `EV-009` | `src/bridge.ts:1248-1251` | client.onclose 非自身 shutdown 时 void shutdown()。 | high | verified | EK-06 |
| `EV-010` | `src/client.ts:352-370` | spawn detached:true / stdio:ignore / unref()。 | high | verified | EK-01, EK-07 |
| `EV-011` | `src/sessions.ts:110-122` | 状态目录 ~/.chrome-devtools-axi/，命名 session 在 sessions/<name>/。 | high | verified | EK-03, EK-08, EK-09 |
| `EV-012` | `src/sessions.ts:28-30, 94-108` | 默认端口 9224；命名 session FNV-1a 映射到 1000 个端口。 | high | verified | EK-08 |
| `EV-013` | `src/sessions.ts:50-90` | validateSessionName 拒绝路径穿越/分隔符/shell 元字符/空/全点/>64。 | high | verified | EK-08 |
| `EV-014` | `src/bridge.ts:642-658; src/bridge-script.ts:31` | EADDRINUSE -> exit code 48。 | high | verified | EK-09 |
| `EV-015` | `src/bridge.ts:217-320, 531-546` | LOOPBACK_HOSTNAMES、extractHostHeaderHostname、isAllowedBridgeHost、isRequestOriginAllowed、isRequestAllowed。 | high | verified | EK-19, EK-20 |
| `EV-016` | `src/bridge.ts:531-546` | 拒绝时 403 {"error":"Forbidden host"} 并记录 host/origin/method/url。 | high | verified | EK-19, EK-36 |
| `EV-017` | `src/bridge.ts:610-620` | createBridgeServer 只有 createServer + handler，无 auth middleware。 | high | verified | EK-21 |
| `EV-018` | `commit 4ee5a49bbdec0c7c0b43aa6d9d6775798fa97534` | fix(bridge): reject non-loopback Host/Origin to block DNS rebinding (GHSA-x439-jhfh-v9x2)；token gate 标为 out-of-scope follow-up。 | high | verified | EK-21, EK-19 |
| `EV-019` | `src/bridge.ts:428-441, 476-520` | /call 的 roots 只校验 Array.isArray + isAbsolute；name 直接传 callTool。 | high | verified | EK-21, EK-22 |
| `EV-020` | `src/bridge.ts:683-760` | buildTransportArgs 的 attach 优先与 launch 分支。 | high | verified | EK-23, EK-24 |
| `EV-021` | `test/keychain-isolation.test.ts; test/bridge.test.ts` | launch 矩阵 7 + attach 矩阵 3；调用方 --chrome-arg 不能挤掉隔离 flag。 | high | verified | EK-23 |
| `EV-022` | `src/bridge.ts:920-1020` | resolveTransportSpec / resolveTransport 的 URL-only / URL+MCP_PATH / local stdio 分支。 | high | verified | EK-24, EK-25 |
| `EV-023` | `test/bridge.test.ts:532-940` | MCP 本地解析 explicit/PATH/global/npx 与 Windows 布局测试。 | high | verified | EK-25 |
| `EV-024` | `src/bridge.ts:1098-1155` | applyRootsNow 发 notifications/roots/list_changed 并等待 2s。 | high | verified | EK-22 |
| `EV-025` | `src/paths.ts:7-9; src/client.ts:640-690` | resolveOutputPath 按 CLI cwd；collectRootDirs 加 cwd 和输出路径最近存在祖先。 | high | verified | EK-22, EK-28 |
| `EV-026` | `src/bridge.ts:330-350, 476-520; src/client.ts:790-850` | isToolResultError 把 isError:true 转成 {error}；parseCallResponse 抛 CdpError。 | high | verified | EK-26, EK-28 |
| `EV-027` | `src/selected-page.ts:1-20, 109-125` | selected page 只由显式 select/new 写入；list_pages 的 [selected] 只展示。 | high | verified | EK-11, EK-12, EK-03 |
| `EV-028` | `src/pages.ts:1-20` | pages parser 是 display-only；不得把解析结果注入 callTool。 | high | verified | EK-11, EK-12 |
| `EV-029` | `src/selected-page.ts:109-125, 198-230` | rememberToolRouting 处理 select/new/close；new 只在唯一 [selected] 行写。 | high | verified | EK-11, EK-13 |
| `EV-031` | `src/bridge.ts:360-410, 470-520` | didMcpPageIdentityChange 认 structuredContent.reconnected 或首行 marker；显式 pageId 返回 identity error。 | high | verified | EK-13, EK-14 |
| `EV-032` | `src/client.ts:790-850` | isMissingPageError 只取最后一个非空行并去 Error: 前缀；注释标 UNVERIFIED DEPENDENCY CONTRACT。 | high | verified | EK-14, EK-26 |
| `EV-033` | `src/generation.ts:23-60; src/uid-freshness.ts:81-84` | snapshot-generation 持久化；每次 capture bump。 | high | verified | EK-15, EK-16, EK-17 |
| `EV-034` | `src/generation.ts:23-47` | getCurrentGeneration / bumpGeneration / resetGeneration 读写状态文件。 | high | verified | EK-15, EK-03 |
| `EV-035` | `src/uid-freshness.ts:45-60, 118-140` | 页面侧 {generation, mutations, observer}；parseUidFresh 要求 generation 相同且 mutations=0。 | high | verified | EK-16, EK-17 |
| `EV-036` | `src/snapshot.ts:51-60; test/snapshot.test.ts:200-240` | uid=id 重写为 uid=gN:id；legacy ref 返回 stale:false。 | high | verified | EK-17 |
| `EV-037` | `src/snapshot.ts:102-115` | truncateSnapshot 默认 16,000 字符，按行边界，保留 truncated/totalLength。 | high | verified | EK-18 |
| `EV-038` | `src/navigation-timeout.ts:22-74` | DEFAULT_MCP_REQUEST_TIMEOUT_MS=60s、DEFAULT_CALL_TIMEOUT_MS=120s、headroom 30s、max 1e9。 | high | verified | EK-27 |
| `EV-039` | `src/idle.ts:24-100; src/bridge.ts:607` | idle 只接受正整数，最低 1s；ambient header 不续命。 | high | verified | EK-29 |
| `EV-040` | `src/run.ts:196-380` | runScript 写临时 .mjs、动态 import、注入 page helper、只捕获 console.log。 | high | verified | EK-30, EK-35 |
| `EV-041` | `src/cli.ts:1829-1900, 1981-2010` | COMMAND_HANDLERS 注册命令；validateCommandFlags 在 handler 前。 | high | verified | EK-31 |
| `EV-042` | `.github/workflows/ci.yml:1-30` | Node 24 + pnpm frozen lockfile、build、test。 | high | verified | EK-32, EK-33 |
| `EV-043` | `.airlock/lint.sh` | pnpm install --frozen-lockfile --ignore-scripts、pnpm exec prettier、pnpm exec tsc --noEmit。 | high | verified | EK-33 |
| `EV-044` | `.airlock/workflows/main.yml` | AIRLOCK_RISK_THRESHOLD: medium。 | high | verified | EK-33 |
| `EV-045` | `README.md keychain isolation; AGENTS.md invariants` | session 只隔离 bridge；keychain 只在 launch 模式隔离。 | high | verified | EK-10, EK-23 |
| `EV-046` | `local: pnpm test` | Test Files 29 passed (29), Tests 796 passed (796), 80.57s。 | high | runtime | EK-05, EK-06, EK-29, EK-32 |
| `EV-047` | `test/bridge.test.ts, test/keychain-isolation.test.ts, test/bridge-lifecycle.test.ts` | 安全矩阵、keychain 矩阵、真实 detached bridge 生命周期测试。 | high | verified | EK-19, EK-20, EK-32 |
| `EV-048` | `.github/workflows/guard-generated-files.yml:1-60; .github/workflows/no-mistakes-required.yml:1-60` | 生成文件 base...head 检查；人工 PR attestation。 | high | verified | EK-34 |
| `EV-049` | `.github/workflows/release-please.yml:1-60; .github/workflows/ci.yml` | release-please + npm publish --provenance；CI paths-ignore 对齐 release 输出。 | high | verified | EK-34, EK-33 |
| `EV-050` | `local: rg -c 'it\(' test/*.test.ts` | 654 个静态 it( 站点；29 个测试文件。 | high | verified | EK-32 |

## 统计

- Evidence rows: 49
- 无 `EV-030`（编号跳号，避免与历史 run 冲突）
- `status: runtime` 仅 EV-046（本机 pnpm test）
- 所有 source 均指向仓库文件/行号或 commit；没有外部网页。

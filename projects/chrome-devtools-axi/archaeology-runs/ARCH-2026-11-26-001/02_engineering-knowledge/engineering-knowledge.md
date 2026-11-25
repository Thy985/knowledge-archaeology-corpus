# 02 · Engineering Knowledge — EK Graph

> 宽底座，回答“这个项目具体怎么解决工程问题”。每条 EK 都是图节点，至少一条边。`abstraction` 只到 L1/L2，不强行升维。

## EK-01 · 短命 CLI + 持久 bridge 的进程分工

- **claim**：CLI 每次调用都是短命进程；跨命令需要保留的浏览器会话由一个 detached bridge 持有。
- **category**：ARCHITECTURE · **abstraction**：L1 · **value**：A · **epistemic_status**：Fact
- **detail**：`README.md:117-160` 的进程图与 `src/client.ts:352-370` 的 `detached: true`、`stdio: "ignore"`、`unref()` 一致。bridge 由 `runBridge` 启动，CLI 只通过 HTTP 访问它。这样避免每条命令重启 Chrome/MCP，也把“短命令”与“长会话”解耦。
- **scope**：适用于需要跨命令保持浏览器登录态、tab、CDP session 的 CLI/MCP wrapper；不适用于无状态单次请求工具。
- **evidence**：EV-001, EV-010, EV-002
- **links**：
  - `EK-02` causal out — CLI 的 HTTP 请求需要 bridge 路由
  - `EK-03` dependency out — bridge 复用依赖 PID/port 状态
  - `EK-10` constraint out — 进程分工决定 session 隔离边界

## EK-02 · Bridge 的 loopback HTTP 路由面

- **claim**：bridge 在 session 端口上暴露 `GET /health`、`GET /health?deep=1`、`GET /tools`、`POST /call`，并只 bind `127.0.0.1`。
- **category**：RUNTIME · **abstraction**：L1 · **value**：A · **epistemic_status**：Fact
- **detail**：`src/bridge.ts:6-9` 是路由契约；`src/bridge.ts:1196-1225` 先 `createTransport` / `client.connect`，再 `server.listen(port, "127.0.0.1")`。`/health` 返回连接状态；`?deep=1` 额外验证 CDP target；`/tools` 列工具；`/call` 转发调用。
- **scope**：本项目的 bridge；不代表所有 MCP bridge 都采用 HTTP。
- **evidence**：EV-002, EV-003
- **links**：
  - `EK-19` subsystem out — 路由前有 Host/Origin gate
  - `EK-21` constraint out — `/call` 无认证，直接映射工具
  - `EK-26` causal out — 路由把 MCP `isError` 转成错误响应

## EK-03 · PID 文件 + session 状态目录是跨进程控制记忆

- **claim**：`bridge.pid` 保存 PID 与端口；`selected-page-id`、`snapshot-generation` 保存在 session 状态目录，供下一个短命 CLI 读取。
- **category**：MEMORY · **abstraction**：L1 · **value**：A · **epistemic_status**：Fact
- **detail**：`src/sessions.ts:110-122` 定义状态目录：默认 `~/.chrome-devtools-axi/`，命名 session 在 `sessions/<name>/`。`src/client.ts` 读写 `bridge.pid`；`src/selected-page.ts` 与 `src/generation.ts` 各自读写文件。这里没有数据库，文件就是控制面内存。
- **scope**：单机、单用户 home 下的状态；不适用于多主机共享。
- **evidence**：EV-011, EV-027, EV-034
- **links**：
  - `EK-04` dependency out — 复用判断依赖 PID/port
  - `EK-05` subsystem out — 生命周期围绕这份状态
  - `EK-11` subsystem out — 页面身份也是文件状态

## EK-04 · Deep health 把“进程活着”提升为“CDP target 可达”

- **claim**：`ensureBridge` 不把 PID 存活当作健康；它调用 `/health?deep=1`，由 bridge 执行一次 `list_pages` 来验证 attached CDP target。
- **category**：RUNTIME · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact
- **detail**：`src/client.ts:192-202` 的 `checkBridgeHealth(opts.deep)` 请求 `/health?deep=1`；`src/bridge.ts:557-580` 的 deep 分支调用 `isBridgeTargetReachable`，失败返回 503 `CDP target unreachable` 并带 reason。PID 活着但浏览器已死时，bridge 会被回收，而不是继续服务一个 stale endpoint。
- **scope**：代理型 supervisor；要求下游资源有可 cheaply 探测的健康信号。
- **evidence**：EV-004, EV-005, EV-006
- **links**：
  - `EK-05` causal out — 探测结果决定复用还是重启
  - `EK-06` contrast out — 另一种失效是 MCP 连接关闭后自退
  - `EK-29` contrast out — idle 是主动生命周期，不是健康失败

## EK-05 · 复用前必须 deep-healthy，否则 terminate + respawn

- **claim**：`ensureBridge` 只在现有 PID 活着且 deep health 通过时复用；否则终止旧 bridge、清空 selected page、spawn 新 bridge，并轮询到 deadline。
- **category**：RUNTIME · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/client.ts:473-530` 依次做 PID/`isProcessAlive`、`checkBridgeHealth({deep:true})`、`terminateBridgeProcess`、`clearSelectedPageId`、`spawnBridge`。`test/bridge-lifecycle.test.ts` 用真实 detached bridge + fake MCP 验证“空闲常驻 / idle 退出 / MCP 死亡退出”。`test/client.test.ts` 还有 deep probe 与 SIGKILL 升级用例。
- **scope**：本地 supervisor，重启成本可接受；不适合必须零停机的高频服务。
- **evidence**：EV-004, EV-008, EV-046
- **links**：
  - `EK-03` dependency in — 依赖 PID/port 文件
  - `EK-07` dependency out — terminate 依赖进程组所有权
  - `EK-09` constraint out — 端口冲突会让 spawn 快速失败

## EK-06 · MCP 连接关闭 → bridge 自退

- **claim**：stdio MCP 进程退出或连接关闭时，bridge 主动 shutdown，而不是继续返回 503 悬挂。
- **category**：RUNTIME · **abstraction**：L2 · **value**：B · **epistemic_status**：Fact + Tested
- **detail**：`src/bridge.ts:1230-1250` 设置 `client.onclose`，非自身 shutdown 时记录并 `void shutdown()`。注释说明：stdio MCP 不在后 bridge 无法服务，且没有别的东西会结束 detached 进程。`test/bridge-lifecycle.test.ts` 的第三例杀掉 fake MCP，断言 bridge 在 8s 内退出且 PID 文件删除。
- **scope**：stdio/local 模式；direct Streamable HTTP 只在自身 close 时触发。
- **evidence**：EV-009, EV-046
- **links**：
  - `EK-04` contrast in — deep health 检测下游死，onclose 是事件驱动检测
  - `EK-07` mechanism out — 退出时同样依赖进程组清理

## EK-07 · Detached bridge 拥有进程组，终止前验证 bridge 身份

- **claim**：bridge 以 `detached: true` 成为进程组 leader；退出时 `process.kill(-pid, SIGTERM)` 清理整个组。CLI 在终止旧 bridge 前用 `ps` 检查命令包含 `chrome-devtools-axi-bridge`，只有确认才启用组杀。
- **category**：RUNTIME · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact
- **detail**：`src/client.ts:352-370` spawn detached，`stdio: "ignore"`；`src/client.ts:269-279` 的 `isBridgeProcess` 执行 `ps -p <pid> -o command=` 并做子串匹配；`ensureBridge` 把 `killProcessGroup: isBridgeProcess(pid)` 传给 `terminateBridgeProcess`。`terminateBridgeProcess` 先 SIGTERM，等 2s，再对组 SIGKILL。`src/bridge.ts:1250-1265` 在 `exit` 时也杀组。
- **scope**：类 Unix（依赖 `ps` 与负 PID 进程组语义）；Windows 支持需另证。
- **evidence**：EV-007, EV-008, EV-010
- **links**：
  - `EK-05` dependency in — ensureBridge 的生命周期动作
  - `EK-06` mechanism in — 自退也走组清理
  - `EK-29` mechanism out — idle shutdown 也是同一进程组清理

## EK-08 · Session 名称校验 + FNV 端口推导

- **claim**：session 名称必须是安全标识符；命名 session 的端口由 FNV-1a hash 映射到 `9225..10224`，显式 `CHROME_DEVTOOLS_AXI_PORT` 优先。
- **category**：RUNTIME · **abstraction**：L2 · **value**：B · **epistemic_status**：Fact + Tested
- **detail**：`src/sessions.ts:28-30` 定义 `DEFAULT_BASE_PORT = 9224`，命名范围 1000 个端口；`validateSessionName` 拒绝路径穿越、分隔符、shell 元字符、空、全点和 >64 字符；`defaultPortForSession` 用 FNV-1a 映射；`resolveSessionPort` 在显式正整数时覆盖。`test/sessions.test.ts` 覆盖确定性、范围、分散、显式覆盖、非法 override。
- **scope**：本地单机 session；不是安全认证，只是命名空间与端口分配。
- **evidence**：EV-011, EV-012, EV-013
- **links**：
  - `EK-09` causal out — 端口分配可能冲突
  - `EK-10` constraint out — 名称只隔离 bridge，不隔离浏览器
  - `EK-03` subsystem out — 名称决定状态目录

## EK-09 · 端口冲突 fail loud（exit 48）

- **claim**：bridge bind 遇到 `EADDRINUSE` 时打印诊断并以固定 exit code 48 退出，不静默复用别的 session。
- **category**：FAILURE · **abstraction**：L2 · **value**：B · **epistemic_status**：Fact + Tested
- **detail**：`src/bridge.ts:642-658` 的 `handleBridgeServerError` 在 `EADDRINUSE` 时 exit `BRIDGE_PORT_IN_USE_EXIT_CODE`；`src/bridge-script.ts:31` 定义为 48。日志明确说全局 export `CHROME_DEVTOOLS_AXI_PORT` 会把所有 session 挤到一个端口。`test/bridge.test.ts` 覆盖该错误处理。
- **scope**：并发 session / 端口固定场景。
- **evidence**：EV-014, EV-011
- **links**：
  - `EK-08` causal in — 端口由 session 推导或显式覆盖
  - `EK-05` constraint in — spawn 失败要快速归因，而不是等满 30s

## EK-10 · Session 只隔离 bridge，attach 模式不隔离浏览器状态

- **claim**：`CHROME_DEVTOOLS_AXI_SESSION` 隔离 bridge 进程、端口、状态文件；如果多个 session attach 到同一个外部 Chrome，浏览器 cookie/storage 仍然共享，per-session generation 不能防止 cross-talk。
- **category**：ARCHITECTURE · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact
- **detail**：`README.md` Keychain isolation 段明确写“Sessions that attach to the same external browser ... are isolated only at the bridge level, where the per-session generation counter does not prevent cross-talk”。`AGENTS.md` 也把“A session isolates only the bridge”列为 invariant。launch 模式下每个 bridge 启一个 Chrome，才不共享浏览器状态。
- **scope**：多 agent / worktree 并发；attach 共享浏览器时不成立。
- **evidence**：EV-001, EV-045
- **links**：
  - `EK-01` constraint in — 进程分工是隔离边界的基础
  - `EK-09` contrast in — 端口冲突是隔离失败的另一种表现
  - `EK-23` mechanism out — keychain 隔离与 session 隔离共享“只对自己启动的资源负责”机制

## EK-11 · Selected page id 只由显式 select/new 写入

- **claim**：页面路由身份只由 `select_page` 的调用参数或 `new_page` dump 中唯一 `[selected]` 完整行写入；`list_pages` 的 `[selected]` 只用于展示。
- **category**：STATE · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/selected-page.ts:1-100` 的文件注释与 `rememberToolRouting` 实现：`select_page` 写 `args.pageId`；`new_page` 只在 `createdPageIdFromNewPageDump` 返回唯一 id 时写，否则 `clearSelectedPageId`；`close_page` 关闭当前 id 则清除。`pages.ts` 注释明确“parser is display-only ... Do not feed ... into callTool injection”。
- **scope**：chrome-devtools-mcp 1.8+ 的 pageId routing；不适用于不需要 pageId 的浏览器工具。
- **evidence**：EV-027, EV-028, EV-029
- **links**：
  - `EK-12` contrast out — list_pages 的 selected 不是路由
  - `EK-13` constraint out — reconnect 会清空这份身份
  - `EK-03` subsystem in — 身份持久化在状态文件

## EK-12 · `list_pages` 的 `[selected]` 是展示态，不是权威

- **claim**：`list_pages` 输出里的 `[selected]` 可能被页面标题/对话框文本伪造或被 MCP 自身移动，因此 AXI 不把它作为 pageId 来源。
- **category**：SECURITY · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact
- **detail**：`src/selected-page.ts:1-20` 解释 “titles and dialog text can forge that marker”；`src/pages.ts:1-20` 说 parser 是 display-only。`overlaySessionSelected` 用 AXI 自己的 id 覆盖显示，避免把 MCP 的 marker 误读为 authority。
- **scope**：所有从上游文本解析控制状态的 wrapper；文本不是 authority。
- **evidence**：EV-027, EV-028
- **links**：
  - `EK-11` contrast in — 显式写入 vs 文本推断
  - `EK-13` dependency out — 身份边界失效时不能从列表重建

## EK-13 · 浏览器 reconnect 是身份 epoch 边界：丢弃 selection，不自动 retarget

- **claim**：bridge 检测到 chrome-devtools-mcp 的 reconnect marker 时，所有 page id 已重发；AXI 清空 selected page，带显式 pageId 的调用返回 identity-changed 错误，`open` 新建 tab 而不是恢复旧 tab。
- **category**：STATE · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/bridge.ts:360-410` 的 `didMcpPageIdentityChange` 接受 structuredContent `reconnected: true` 或首行文本 marker；`handleCallRequest` 在 `pageIdentityChanged && args.pageId` 时返回 `PAGE_IDENTITY_CHANGED_ERROR`；`ensureBridge` 在 respawn 前 `clearSelectedPageId`。`test/bridge.test.ts` 覆盖伪造、对话框换行、结构化 marker。
- **scope**：上游进程重连会重发 id 的协议；不适用于稳定 id 的后端。
- **evidence**：EV-031, EV-029, EV-032
- **links**：
  - `EK-11` dependency in — 依赖 AXI 自己的 selection
  - `EK-12` mechanism in — 不从 list_pages 重建
  - `EK-14` causal out — 身份变化后用错误文本兜底

## EK-14 · Reconnect / missing-page 文本匹配是位置敏感且 fail-closed

- **claim**：reconnect marker 只看首行前缀；missing page 只看最后一个非空行并去掉 `Error:` 前缀。两者都不追求不可伪造，而是在无法区分时选择“响亮失败 + 丢弃路由”，绝不静默 retarget。
- **category**：FAILURE · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact
- **detail**：`src/bridge.ts:360-380` 注释写明 “Only the first line is consulted”；`src/client.ts` 的 `isMissingPageError` 注释解释页面可控文本可能伪造最后一行，但后果是 fail-closed（spurious loud error + dropped routing），不是 silent retarget；上游 page id 是进程级 monotonic counter，stale id 解析不到任何页。两者都标注 “UNVERIFIED DEPENDENCY CONTRACT”，依赖上游输出顺序。
- **scope**：文本协议解析；上游重排时退化，但不至于静默错配。
- **evidence**：EV-031, EV-032, EV-047
- **links**：
  - `EK-13` mechanism in — reconnect 处理的文本侧
  - `EK-26` causal out — 失败文本进入 CdpError/非零退出
  - `EK-35` contrast out — evaluate_script 的 callable 契约也是上游协议假设

## EK-15 · Snapshot generation 是持久的内容 epoch

- **claim**：每次抓快照递增 `snapshot-generation` 并写入 session 状态目录；它跨 CLI 进程存活，用来区分“同一页面但不同快照”。
- **category**：STATE · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/generation.ts:1-60` 的 `getCurrentGeneration` / `bumpGeneration` / `resetGeneration`；文件写失败时仍返回 bump 后的值，代价是“最多漏一次 stale 检测”，不是挂起。`src/uid-freshness.ts:81-84` 每次 capture 都 `bumpGeneration`。
- **scope**：同一 session 的页面 ref；不适用于跨 session 共享 ref。
- **evidence**：EV-033, EV-034, EV-035
- **links**：
  - `EK-16` causal out — generation 需要页面侧 mutation 状态配合
  - `EK-17` causal out — 快照文本被 stamp
  - `EK-18` subsystem out — truncation 不改变 generation 语义

## EK-16 · MutationObserver 把“新鲜”绑定到零突变

- **claim**：页面侧保存 `{generation, mutations, observer}`；每次 snapshot 注入 generation，并用 MutationObserver 计突变。校验要求 generation 等于当前且 mutations 为 0，否则抛 `STALE_REF`。
- **category**：STATE · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/uid-freshness.ts:45-60` 在页面里写 `__chromeDevtoolsAxiSnapshotGeneration` 并注册 observer；`getPageRefState` 读回；`parseUidFresh` 在 generation 不匹配或 `mutations !== 0` 时 `throwStaleRef`。`captureFreshSnapshot` 如果第一次 capture 后立即观察到 mutation，会再抓一次，避免把“抓取过程中变化”的快照发出去。
- **scope**：DOM 可变页面的 uid 操作；纯静态/无 DOM 页面退化为 generation 检查。
- **evidence**：EV-035, EV-033
- **links**：
  - `EK-15` dependency in — generation 由上层递增
  - `EK-17` mechanism out — stamp/parse 共享同一 token 格式
  - `EK-18` constraint out — truncation 不能丢掉 uid token

## EK-17 · UID stamp `gN:` + legacy ref 例外

- **claim**：快照中的 `uid=<id>` 被重写为 `uid=g<generation>:<id>`；带 generation 的 ref 必须匹配当前 generation；没有 generation 的 legacy ref 被视为 not stale，但仍要通过页面状态（generation 相同、mutations=0）检查。
- **category**：ENGINEERING · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/snapshot.ts:45-60` 的 `stampSnapshotGeneration` 用正则替换，已 stamp 的不重复；`test/snapshot.test.ts:200-240` 明确 `checkUidGeneration("@5", 7)` 返回 `stale:false, refGeneration:null`；但 `parseUidFresh` 仍会 `getPageRefState` 并要求 `state.generation === snapshotGeneration && state.mutations === 0`。兼容性例外是有代价的：legacy ref 不绑定它被生成的那个 epoch，只受页面当前状态约束。
- **scope**：向后兼容旧调用方；新调用方应使用 stamp。
- **evidence**：EV-033, EV-036, EV-035
- **links**：
  - `EK-15` causal in — generation 提供 token 前缀
  - `EK-16` mechanism in — 同一状态门
  - `EK-18` contrast out — 截断与 stamp 的顺序/边界

## EK-18 · Snapshot 默认 16,000 字符截断，`--full` 关闭

- **claim**：快照在 16,000 字符处按行边界截断，保留 `truncated/totalLength` 元数据；`--full` 绕过截断。
- **category**：TOOLING · **abstraction**：L1 · **value**：B · **epistemic_status**：Fact + Tested
- **detail**：`src/snapshot.ts:102-115` 的 `truncateSnapshot(snapshot, full, limit=16000)`；`--full` 在 CLI 多个 command 的 `withFullFlag` / `splitFullFlag` 中传递。截断是 token 预算控制，不是安全边界。
- **scope**：大页面；需要完整树时用 `--full`。
- **evidence**：EV-037
- **links**：
  - `EK-15` subsystem in — 同属 snapshot 输出管线
  - `EK-17` contrast in — 截断不能破坏 uid 格式

## EK-19 · Host/Origin gate 在路由前执行

- **claim**：每个请求先过 `isRequestAllowed`：Host 必须存在且 hostname 为 loopback；Origin 缺失允许，存在则必须 loopback；失败返回 403 `{"error":"Forbidden host"}`，并在调用任何 CDP 工具前返回。
- **category**：SECURITY · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/bridge.ts:217-305` 定义 `LOOPBACK_HOSTNAMES`、`extractHostHeaderHostname`、`isAllowedBridgeHost`、`isRequestOriginAllowed`、`isRequestAllowed`；`src/bridge.ts:531-546` 在 `handleBridgeRequest` 最前面 gate，覆盖 `/health`、`/tools`、`/call`。`test/bridge.test.ts:1766-1807` 断言三种路由都被 403，且 CDP 工具未被调用。
- **scope**：浏览器侧 DNS rebinding 防御；不防御同机非浏览器进程。
- **evidence**：EV-015, EV-016, EV-047
- **links**：
  - `EK-20` constraint out — Host 解析严格性是 gate 的前提
  - `EK-21` causal out — origin gate 挡住浏览器侧，反而暴露了 caller 认证缺口
  - `EK-36` causal out — 拒绝会记录 host/origin/method/url

## EK-20 · Host 解析的严格性：bracket、port、后缀

- **claim**：`extractHostHeaderHostname` 对 `[::1]:9224` 取 `::1`，对 `[::1]evil.com` 返回 null；对裸 IPv6 保留整串；对空值返回 null。Origin 用 `new URL(origin).hostname`，解析失败即拒绝。
- **category**：SECURITY · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/bridge.ts:238-269` 的解析逻辑；`test/bridge.test.ts:1663-1713` 覆盖 `[::1]evil.com`、`localhost.evil.com`、大小写、端口剥离。Host 是 forbidden header，页面 JS 不能伪造，因此它是 DNS rebinding 的有效控制点。
- **scope**：HTTP/1.1 Host 语义；HTTP/2 `:authority` 未在本项目测试。
- **evidence**：EV-015, EV-047
- **links**：
  - `EK-19` constraint in — gate 依赖解析器不把恶意后缀当 loopback

## EK-21 · 无 token：`/call` 是任意 MCP 工具调用

- **claim**：bridge 没有 auth middleware，也没有 token/secret；`POST /call` 的 `name` 直接传给 `client.callTool`。修复 commit 明确把 token gate 标为 out-of-scope follow-up。
- **category**：SECURITY · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact
- **detail**：`src/bridge.ts:470-520` 的 `handleCallRequest` 解析 body 后 `client.callTool({name, arguments}, roots)`；`createBridgeServer` 只有 `createServer` + handler，没有认证中间件；`runBridge` 只 `listen(port, "127.0.0.1")`。commit `4ee5a49` message：`A token/secret gate on the loopback API is reasonable defense-in-depth ... but is out of scope for this fix.`
- **scope**：同机本地控制面；远程攻击面主要是浏览器 DNS rebinding（已由 Host/Origin gate 挡）。
- **evidence**：EV-017, EV-018, EV-019
- **links**：
  - `EK-19` constraint in — gate 只解决 origin，不解决 caller 身份
  - `EK-22` causal out — 无认证使 caller 提供的 roots 成为高权输入
  - `EK-36` subsystem out — 拒绝日志是唯一审计线索

## EK-22 · `roots` 由请求体提供，任意绝对路径，无服务端 allowlist

- **claim**：`POST /call` 的 `roots` 字段接受任意绝对路径数组；bridge 把它们传给 `applyRootsNow`，切换 MCP workspace roots。调用方还能通过工具输出路径让 `collectRootDirs` 加入任意祖先目录。
- **category**：SECURITY · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact
- **detail**：`src/bridge.ts:428-441` 只校验 `Array.isArray` 和 `isAbsolute`；`src/bridge.ts:1098-1155` 的 `applyRootsNow` 发 `roots/list_changed` 并等待 2s；`src/client.ts:675-690` 的 `collectRootDirs` 总是加入 cwd，再加入输出路径最近存在的祖先。没有服务端 allowlist，也没有把 roots 绑定到 CLI 的真实 cwd。
- **scope**：本地调用方已被信任的模型；如果 bridge 暴露给非可信本地进程，则 roots 是权限放大面。
- **evidence**：EV-019, EV-024, EV-025
- **links**：
  - `EK-21` causal in — 无认证 -> caller 提供的 roots 成为权威
  - `EK-36` causal out — 高权调用没有 per-caller 审计
  - `EK-28` constraint in — 正常 CLI 路径由 `resolveOutputPath` 约束

## EK-23 · Keychain 隔离 flag 只加在 launch 模式

- **claim**：`--use-mock-keychain`、`--password-store=basic` 只在 AXI 自己启动浏览器时追加（`--isolated` 或 `--userDataDir`）；`--autoConnect`、`--browserUrl`、ws attach 不追加。
- **category**：SECURITY · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/bridge.ts:683-760` 的 `buildTransportArgs`：attach 分支先 return；launch 分支在 `--isolated`/`--userDataDir` 后循环追加 flag。`test/keychain-isolation.test.ts` 用 7 个 launch 矩阵 + 3 个 attach 矩阵断言，且调用方 `CHROME_DEVTOOLS_AXI_CHROME_ARGS` 不能把隔离 flag 挤掉。
- **scope**：macOS/Linux 密码存储；外部浏览器归其 owner。
- **evidence**：EV-020, EV-021, EV-045
- **links**：
  - `EK-24` mechanism in — 由 transport 模式决定
  - `EK-10` contrast in — 同样是“只对自己启动的资源负责”

## EK-24 · Transport 选择：attach 优先、URL 决定 HTTP/stdio

- **claim**：`AUTO_CONNECT` > `BROWSER_URL`（ws→`--wsEndpoint`，http→`--browserUrl`）> launch（`--userDataDir` 或 `--isolated`）。MCP 侧：URL-only 走 Streamable HTTP；URL + MCP_PATH 走带 `--serverUrl` 校验的 stdio proxy；否则本地 stdio。
- **category**：ARCHITECTURE · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/bridge.ts:688-760` 的 `buildTransportArgs`；`src/bridge.ts:920-1020` 的 `resolveTransportSpec` / `resolveTransport`。`resolveTransportSpec` 在 shared URL + explicit path 时先跑 `--help` 验证 `--serverUrl`，不支持则报错而不是启动一个本地浏览器。`test/bridge.test.ts` 覆盖大量组合。
- **scope**：本项目支持的四种部署形态；共享 MCP 服务归外部 operator。
- **evidence**：EV-020, EV-022, EV-023
- **links**：
  - `EK-25` causal out — 选择结果决定 spawn 方式
  - `EK-23` constraint out — keychain 只跟 launch 分支
  - `EK-10` constraint out — attach 共享外部浏览器

## EK-25 · MCP 本地解析顺序：explicit → PATH → global → npx

- **claim**：本地 stdio 模式按 `CHROME_DEVTOOLS_AXI_MCP_PATH` → PATH 上的 `chrome-devtools-mcp` → 全局安装探测 → `npx -y chrome-devtools-mcp@latest` 解析，避免冷启动 npx 超过 bridge deadline。
- **category**：ENGINEERING · **abstraction**：L2 · **value**：B · **epistemic_status**：Fact + Tested
- **detail**：`src/bridge.ts:920-1000` 的顺序与注释；`test/bridge.test.ts:532-940` 覆盖 explicit、PATH、global、Windows npm 布局、npx fallback、PATH 目录误匹配。shared URL 模式强制 explicit path，避免隐式启动本地浏览器。
- **scope**：本地开发/CI/离线机器；不是包管理器的通用解析规则。
- **evidence**：EV-022, EV-023, EV-047
- **links**：
  - `EK-24` dependency in — transport 选择后调用
  - `EK-05` constraint in — 解析太慢会撞 readiness deadline

## EK-26 · MCP `isError` 必须转成 bridge error，不能打印成功

- **claim**：MCP 工具失败常以 JSON-RPC 成功响应 + `isError: true` 返回；bridge 把这个结果转成 `{error}`，CLI 抛 `CdpError` 并非零退出。否则会把被拒绝的截图/写文件报告成成功（issue #96）。
- **category**：FAILURE · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/bridge.ts:330-350` 的 `isToolResultError`；`handleCallRequest` 在 `isToolResultError(result)` 时 `writeJson(res, 200, {error: text || ...})`；`src/client.ts` 的 `parseCallResponse` 把 `{error}` 抛成 `CdpError`。`test/bridge.test.ts:141-149` 只把 `isError: true` 判错。
- **scope**：MCP 工具错误语义；协议层错误另走 JSON-RPC error。
- **evidence**：EV-026, EV-032
- **links**：
  - `EK-14` mechanism in — 文本错误匹配是补充网
  - `EK-35` mechanism out — evaluate_script 的 callable 失败也会被这条链路放大
  - `EK-02` causal in — 路由层负责转换

## EK-27 · 嵌套 deadline：导航 < MCP request < CLI `/call`

- **claim**：`mcpRequestTimeoutMs` 至少 60s，且为 `2 * navigation timeout + 30s`；`callRequestTimeoutMs` 至少 120s，且为 MCP deadline + 30s。外层永远大于内层，避免长导航被上层先切断。
- **category**：RUNTIME · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/navigation-timeout.ts:1-74` 定义 `DEFAULT_MCP_REQUEST_TIMEOUT_MS=60_000`、`DEFAULT_CALL_TIMEOUT_MS=120_000`、`DEADLINE_HEADROOM_MS=30_000`、`MAX_NAVIGATION_TIMEOUT_MS=1e9`；`test/navigation-timeout.test.ts` 39 个用例覆盖边界。`open`/`newpage` 的 `--timeout` 会进入 `args.timeout`，两层 deadline 都从它推导。
- **scope**：多层 RPC/HTTP 超时；不适用于单一层调用。
- **evidence**：EV-038, EV-047
- **links**：
  - `EK-28` constraint out — 输出路径是另一个跨层上下文
  - `EK-24` dependency in — transport 决定内层超时语义

## EK-28 · 输出路径在 CLI cwd 解析，再绝对化传给 bridge / MCP roots

- **claim**：所有调用方提供的输出文件/目录都先经 `resolveOutputPath`：相对路径按**调用 CLI 的 cwd** 解析，绝对路径原样；然后 `collectRootDirs` 把 cwd 和输出路径的最近存在祖先作为 MCP roots。
- **category**：ENGINEERING · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact
- **detail**：`src/paths.ts:1-9` 是唯一 chokepoint；`src/client.ts:640-690` 的 `FILE_OUTPUT_ARGS_BY_TOOL` / `DIR_OUTPUT_ARGS_BY_TOOL` 枚举 screenshot、network-get、perf、heap、lighthouse 等参数；注释明确“Add a key here when a new file-writing tool argument is introduced”。否则 bridge 的 cwd 与 CLI 不同，相对路径会写到错误位置。
- **scope**：所有跨进程文件写；不适用于纯 stdin/stdout。
- **evidence**：EV-026, EV-025
- **links**：
  - `EK-27` mechanism in — 同属跨层上下文传递
  - `EK-22` constraint in — roots 的上游正常路径
  - `EK-30` contrast out — run 脚本的临时文件是另一类路径

## EK-29 · Idle 关闭：opt-in、in-flight 延迟、ambient 探针不续命

- **claim**：默认不启用 idle；只有 `CHROME_DEVTOOLS_AXI_IDLE_TIMEOUT_MS` 为正整数才启用，低于 1s 抬到 1s。请求 in-flight 时暂停倒计时；ambient header 的 home view 探针不算 activity，避免 agent session 启动把废弃 bridge 保活。
- **category**：RUNTIME · **abstraction**：L2 · **value**：B · **epistemic_status**：Fact + Tested
- **detail**：`src/idle.ts:1-100` 的 `resolveIdleTimeoutMs` 只接受 `/^\d+$/`，`createIdleTracker` 用 `inFlight` 和 `lastActivity` 调度，timer `unref()`；`src/bridge.ts` 的 `isIdleActivity` 用 `AMBIENT_REQUEST_HEADER` 排除探针。`test/idle.test.ts` 11 例，`test/bridge-lifecycle.test.ts` 真实进程验证。
- **scope**：无人值守 agent session；需要长期保活的场景应关闭。
- **evidence**：EV-039, EV-046
- **links**：
  - `EK-07` mechanism in — 关闭时同一进程组清理
  - `EK-04` contrast in — 健康失败 vs 主动 idle
  - `EK-06` contrast in — MCP 死 vs 无活动

## EK-30 · `run` 的脚本执行边界：临时 `.mjs` + `page` 全局 + 只听 console.log

- **claim**：`run` 把 stdin 写到临时 `.mjs`，动态 import 以支持 top-level await；注入 `page` helper，只捕获 `console.log`，最后清理临时文件。默认导出函数会被调用。
- **category**：TOOLING · **abstraction**：L2 · **value**：B · **epistemic_status**：Fact + Tested
- **detail**：`src/run.ts:250-380` 的 `runScript`：`mkdtempSync`、`writeFileSync`、`import(tmpFile)`、`console.log = captureLog`、finally 恢复并 unlink/rmdir。`createPageHelper` 提供 open/eval/wait/snapshot/click/fill/type/press/back。`test/run.test.ts` 57 个执行用例，`test/main.test.ts` 80 个。
- **scope**：本项目的 `run` 命令；不是通用沙箱，脚本拥有进程权限。
- **evidence**：EV-040, EV-047
- **links**：
  - `EK-01` subsystem in — CLI 短命进程
  - `EK-28` contrast in — 临时文件路径 vs 输出路径
  - `EK-35` dependency out — eval 包装契约

## EK-31 · 36 个命令的统一注册与 flag 校验

- **claim**：CLI 通过 `COMMAND_HANDLERS` 注册 36 个命令，每个命令有允许 flag、取值 flag、位置参数规则；未知 flag 在 handler 前被校验拒绝。
- **category**：TOOLING · **abstraction**：L1 · **value**：B · **epistemic_status**：Fact + Tested
- **detail**：`src/cli.ts:1860-1900` 是命令表；`COMMAND_FLAGS`、`COMMAND_VALUE_FLAGS`、`COMMAND_POSITIONAL_TEXT_START`、`COMMAND_DASH_POSITIONAL_SLOTS` 定义校验；`COMMANDS` 先 `validateCommandFlags` 再 handler。`test/cli.test.ts`、`test/main.test.ts` 覆盖 help/unknown。
- **scope**：本项目 CLI；不是通用参数解析框架。
- **evidence**：EV-041, EV-047
- **links**：
  - `EK-01` causal in — 命令层是短命进程入口
  - `EK-26` subsystem out — 命令失败最终走 CdpError/exit

## EK-32 · 测试体系：796 个执行用例 + 真实进程生命周期测试

- **claim**：仓库有 29 个 vitest 文件、654 个静态 `it(` 站点，运行时展开为 796 个用例；其中 `bridge-lifecycle` 用真实 detached bridge + fake MCP 观察退出，`bridge.test` 覆盖安全 gate 和 transport 矩阵。
- **category**：TESTING · **abstraction**：L1 · **value**：A · **epistemic_status**：Fact + Runtime
- **detail**：本地 `pnpm test` 输出 `Test Files 29 passed (29)`、`Tests 796 passed (796)`、80.57s；`test/bridge-lifecycle.test.ts` 启动 `bin/chrome-devtools-axi-bridge.ts`，等待 `READY`，然后验证 idle/MCP 死亡。`test/bridge.test.ts` 139 个静态用例，6 个 `it.each`。`test/keychain-isolation.test.ts` 12 个。
- **scope**：本机 Node 22；CI 使用 Node 24，未在本轮执行 CI 运行时。
- **evidence**：EV-046, EV-047, EV-050
- **links**：
  - `EK-05` mechanism in — 生命周期测试证明行为
  - `EK-19` mechanism in — 安全矩阵测试
  - `EK-33` dependency out — CI 门禁消费测试结果

## EK-33 · CI + airlock 的本地质量门

- **claim**：CI 在 Node 24 上 frozen lockfile、build、test；`.airlock/lint.sh` 用 pnpm 执行 Prettier + `tsc --noEmit`；airlock review gate 风险阈值为 medium。
- **category**：WORKFLOW · **abstraction**：L1 · **value**：B · **epistemic_status**：Fact
- **detail**：`.github/workflows/ci.yml:1-30`；`.airlock/lint.sh`；`.airlock/workflows/main.yml` 的 `AIRLOCK_RISK_THRESHOLD: medium`。`test/airlock-lint.test.ts` 断言 lint 脚本必须含 `pnpm install --frozen-lockfile --ignore-scripts`、`pnpm exec prettier`、`pnpm exec tsc --noEmit`，且不得含 `npm install` / `npx `。
- **scope**：本仓库贡献流程；不替代运行时安全测试。
- **evidence**：EV-042, EV-043, EV-044
- **links**：
  - `EK-32` dependency in — 测试是门禁输入
  - `EK-34` causal out — 发布治理的另一半

## EK-34 · 生成文件/发布产物的机器强制所有权

- **claim**：`guard-generated-files.yml` 禁止 PR 手改 `CHANGELOG.md` / `.release-please-manifest.json`；`no-mistakes-required.yml` 要求人工 PR 带 attestation；`release-please.yml` 用 release-please + npm `--provenance`。CI 的 `paths-ignore` 与 release-please 输出集对齐，避免 release PR 产生永远 action_required 的 run。
- **category**：WORKFLOW · **abstraction**：L2 · **value**：B · **epistemic_status**：Fact + Tested
- **detail**：`guard-generated-files.yml:1-60` 的 `git diff base...head` 检查；`no-mistakes-required.yml:1-60` 的 author 豁免与 pinned action；`ci.yml` 的 `paths-ignore`；`test/release-ci-exclusions.test.ts` 6 个用例从 `release-please-config.json` + workflow 推导期望输出集。
- **scope**：单一仓库的 release 治理；多包 monorepo 需扩展。
- **evidence**：EV-042, EV-048, EV-049
- **links**：
  - `EK-33` mechanism in — 同属仓库治理

## EK-35 · `evaluate_script` 必须收到 callable，不是表达式结果

- **claim**：chrome-devtools-mcp 的 `evaluate_script` 会调用传入的 function payload；因此 `wrapJsExpression` 把普通表达式包成 `() => (expr)`，已是箭头函数/function 的源码原样传递，并特判无参 IIFE 解包。
- **category**：ENGINEERING · **abstraction**：L2 · **value**：A · **epistemic_status**：Fact + Tested
- **detail**：`src/run.ts:20-95` 的 `unwrapNoArgIIFE` 与 `wrapJsExpression`；`createPageHelper.eval` 对真实函数用 `String(fn)` 原样发送，因为 head regex 不能识别所有函数源码；`test/wrap-js.test.ts` 13 例，`test/main.test.ts` / `test/run.test.ts` 执行发送的源码。`AGENTS.md` 把这条列为 invariant。
- **scope**：MCP evaluate_script 协议；不是通用 JS 沙箱。
- **evidence**：EV-040, EV-047
- **links**：
  - `EK-30` dependency in — run helper 使用
  - `EK-26` mechanism in — 调用失败走错误翻译
  - `EK-14` contrast in — 都是上游协议假设，需显式标注

## EK-36 · 拒绝日志：唯一的本地控制面审计线索

- **claim**：gate 拒绝时记录 `host`、`origin`、`method`、`url`；日志函数由调用方注入，测试可不输出。除此之外，`/call` 没有 per-caller 审计。
- **category**：SECURITY · **abstraction**：L1 · **value**：B · **epistemic_status**：Fact
- **detail**：`src/bridge.ts:531-546` 的 `logForbidden?.("Rejected request with disallowed host: host=... origin=... method=... url=...")`；`createBridgeServer` 传入日志实现。没有调用方身份、没有 tool 参数日志、没有持久审计。
- **scope**：排障与粗粒度入侵检测；不是合规审计。
- **evidence**：EV-016
- **links**：
  - `EK-19` causal in — 由 gate 触发
  - `EK-21` contrast in — 无认证使日志无法归属到主体

## 质量统计

- EK 总数：36
- 有 links 的 EK：36 / 36（孤立 0，0%）
- 平均出边：1.33；总声明边 59（无向边只计一次）
- 边类型分布：subsystem 8、causal 15、constraint 13、dependency 7、mechanism 8、contrast 8
- 证据锚点：所有 EK 的 `source` 都指向仓库文件/行号或 commit；无“凭记忆”条目。

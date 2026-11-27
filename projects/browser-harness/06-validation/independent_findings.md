# 阶段一 · 独立盲重建发现（Independent Findings）

- 审计对象：browser-harness v0.1.15，commit `373aecc4c40ef67d97b5ee69c144684a64a42a11`（2026-11-25，shallow + grafted + detached HEAD）。
- 只读事实来源：`repo/` 仓库本体（源码 + 单元/集成测试 + 文档 + 配置）。阶段一未读 `package/`、未读 `audit/` 下任何其他审计者文件。
- 校准依据：knowledge-archaeology skill 的 L1–L5、Fact/Observation/Hypothesis 与 Flow 定义仅作术语校准，未作为项目事实来源。
- 方法：逐文件读源码（daemon.py / admin.py / helpers.py / auth.py / run.py / mcp_server.py / _ipc.py / paths.py / telemetry.py / recorder.py / report.py / 其余模块），并用 `tests/unit/` 与 `tests/integration/` 中的测试做反向检验；Q5 另从 PyPI 下载 `browser-use==1.0.8` 与 `browser-harness==0.1.13`、`0.1.14` wheel 读源码与元数据（存放于 `audit/_scratch/`）。
- 计数口径（本人实测，`wc -l`）：`src/` Python 共 **8428** 行；`tests/unit/` 共 **6356** 行；`tests/integration/` 共 **437** 行；`src+tests` Python 合计 **15258** 行。`agent-workspace/domain-skills/` 共 **209** 个文件。prompt 所述"约 15.4k 行 Python"与实测 15258 行基本吻合。
- 认识论标记：**Fact**＝代码/配置/元数据中可直接定位；**Observation**＝由代码路径可推断行为但未运行验证；**Hypothesis**＝合理推断但无直接证据。

> 冻结声明：本文写入后结论不得回改。阶段二若发现错误，只能在文件末尾追加 `## Errata（冻结后修正）` 段，原结论原文保留。

---

## Q1 · Owner 身份与传递

### Q1-1 owner key 是什么、从哪来、怎么标识调用方

- **[Fact]** owner key 是一个字符串；客户端侧取值链为 `BH_OWNER` 环境变量 → `CLAUDE_CODE_SESSION_ID` → `CODEX_THREAD_ID` → 兜底 `"default"`。证据：`src/browser_harness/_ipc.py:33-36`。
- **[Fact]** helpers 层每次发 IPC 请求时把当前 `OWNER` 注入请求体字段 `owner`；`OWNER` 在模块加载时按同一取值链解析。证据：`src/browser_harness/helpers.py:20-26`、`helpers.py:77-81`（`{**req, "owner": OWNER}`）。
- **[Fact]** daemon 侧从请求体取 key，缺失时兜底 `"default"`，随后 `setdefault` 建 Owner 对象并刷新 `seen` 时间戳。证据：`src/browser_harness/daemon.py:1087-1088`、`daemon.py:532-535`。
- **[Fact]** admin 侧的健康探测（connection_status）同样携带 `ipc.owner()`。证据：`src/browser_harness/admin.py:477`；测试佐证 `tests/unit/test_admin.py:168-180`。
- 边界条件：key 只在单 daemon 命名空间内有意义；不同 `BU_NAME` 的 daemon 之间 owner 互不可见。

### Q1-2 同一 owner 的 tab 集合如何组织

- **[Fact]** daemon 维护四张归属表：`owners`（key → Owner）、`tab_owner`（target_id → key）、`session_owner`（session_id → key）、`downloads`（guid → key）；另有 `info`（target_id → 最新 targetInfo 缓存）。证据：`daemon.py:509-513`。
- **[Fact]** `Owner` 内部状态：`tabs`（tid → session_id，未挂接时为 None）、`current`、`marks`（受保护标记）、`created`（本 owner 自建的 tab）、`released`（被空闲交还的 tab）、`events`、`dialog`、`name`、`seen`、`lock`。证据：`daemon.py:497-507`。
- **[Fact]** 归属与标记跨 daemon 重启持久化到 STATE 文件，启动时按现存页面恢复。证据：`daemon.py:635-643`（`_save`）、`daemon.py:645-660`（`_load`）。
- **[Fact]** 同一 owner 的多个 tab 以 `tabs` 字典组织；`current` 指向其当前操作页；`mark_tab` 可给 tab 打 `deliverable`/`handoff` 标记，空闲回收时受保护。证据：`daemon.py:608-617`、`helpers.py:471-475`。

### Q1-3 多 agent/客户端并发时归属如何划分

- **[Fact]** tab 归属判定 `_holder(tid)`：先沿 frame 树回溯到根 tab（`_root`，最多 16 层），再查 `tab_owner`；若该 owner 距 `seen` 超过 `OWNER_IDLE`（默认 1800s），视为空闲、tab 可被夺。证据：`daemon.py:541-554`。
- **[Fact]** 夺 tab 路径 `_take`：旧 owner 的会话被 `Target.detachFromTarget` 断开，新 owner `_own` 登记；旧 owner 下次命令收到 `tab-taken` 错误。证据：`daemon.py:579-591`。
- **[Fact]** 每个 Owner 持有一把 asyncio 锁，挂接/开 tab/回收在锁内进行，避免并发竞态。证据：`daemon.py:664`、`daemon.py:675`、`daemon.py:767`。
- **[Fact]** 全新 owner 不会抢占用户已有页面：没有空闲可复用空白页时，后台 `Target.createTarget` 自建 about:blank。证据：`daemon.py:662-671`；测试 `tests/unit/test_daemon.py:289-299`。
- **[Observation]** 归属划分是"按 key 分桶"：同一 key 的所有连接被视为同一 owner，共享同一 tab 集合；代码中未见按连接/进程再细分的机制。

### Q1-4 身份是声明式还是证据式

- **[Fact]** daemon 侧不验证 owner key 与调用方真实身份的对应关系：任何能连上 IPC 通道的请求都可在请求体里写任意 `owner` 字段，daemon 直接采信。证据：`daemon.py:1087-1088`（无校验逻辑）。
- **[Fact]** 身份边界实际由传输层承担：POSIX 为 AF_UNIX socket（目录 0700、socket 0600、umask 0o077）；Windows 为 TCP loopback + 每进程 token，请求必须带匹配 token，否则 `unauthorized`。证据：`src/browser_harness/_ipc.py:38-41`、`daemon.py:1057-1063`。
- **[Observation]** 结论：owner 身份是**声明式**（客户端自称），daemon 侧无证据式核验；传输层权限是唯一的身份边界。
- **[Hypothesis]** 若两个不相关客户端碰巧使用相同 session id 字符串（或都不设 `BH_OWNER` 而落到同一 `default`），会被并入同一 owner 共享 tab；代码未见防猜测/防碰撞机制。

---

## Q2 · Tab 生命周期

### Q2-1 完整状态机：新建 / attach / detach / close / 回收

- **[Fact]** 状态迁移主路径：
  1. **free → owned**：`_own` 写 `tab_owner`、`tabs`、`created`/`current`，持久化。证据：`daemon.py:608-617`。
  2. **owned → session 挂接**：`_session` 在首次使用时 `Target.attachToTarget(flatten=True)`，登记 `session_owner[sid]=key`，注入 Network/Page 域。证据：`daemon.py:673-695`。
  3. **detach（交还但不关页）**：`_release` 对未关闭的 tab 发 `Target.detachFromTarget`，清标题（未标记页），`_drop` 摘除归属。证据：`daemon.py:726-746`、`daemon.py:619-633`。
  4. **close（关页）**：仅当 `closed` 条件成立时发 `Target.closeTarget`，并从 `info` 摘除。证据：`daemon.py:738-741`。
  5. **destroyed / crashed**：`Target.targetDestroyed`、`Target.targetCrashed` 事件触发 `_drop`；当前页丢失时写入 `o.gone`，下次命令回报 `tab-closed`。证据：`daemon.py:847-864`、`daemon.py:619-633`。
- **[Fact]** 弹窗（popup）继承 opener：`targetCreated` 时若 `openerId` 属于本 daemon 已知 tab，归属沿用 opener 的 owner。证据：`daemon.py:852-854`。
- **[Fact]** 原始 CDP `Target.attachToTarget` 也会夺 tab（`_take`，`current=False`）；iframe 目标需先证明落在本 owner 的 tab 内，否则 `frame-not-yours`。证据：`daemon.py:1215-1226`。

### Q2-2 空闲多久被回收、被谁回收、回收时关不关 tab

- **[Fact]** 回收阈值：`OWNER_IDLE`（`BH_OWNER_IDLE`，默认 1800s）后释放非当前 tab；`OWNER_FORGET`（`BH_OWNER_FORGET`，默认 86400s）后连当前 tab 一并交还；空 owner 在 2× `OWNER_FORGET` 后被遗忘。证据：`daemon.py:144-145`、`daemon.py:759-770`。
- **[Fact]** 回收者是 daemon 内部 sweeper 协程（`sweep`，周期 `SWEEP_EVERY = min(60, OWNER_IDLE, OWNER_FORGET, IDLE_EXIT)`）；另有主动路径 `release_tabs` meta（`how="done"`）。证据：`daemon.py:147`、`daemon.py:752-775`、`daemon.py:1277`。
- **[Fact]** 默认（idle）回收**不关任何 tab**：当前 tab 跳过；其余 tab 仅 detach、留开。证据：`daemon.py:733-735`、`daemon.py:742-746`；测试 `tests/unit/test_daemon.py:970-986`（断言 `Target.closeTarget` 调用为空）。
- **[Fact]** 关页条件（`closed`）：`close` 开关成立（`how=="done"`，或 `BH_CLOSE_IDLE_TABS=="1"` 且非当前 tab）**且** tab 在 `o.created` 中**且**不在 `o.marks` 中**且**不是最后一个真实页。证据：`daemon.py:736-738`。
- **[Fact]** `BH_CLOSE_IDLE_TABS=1` 才在空闲回收时关"本 owner 自建、未标记"的非当前 tab；测试佐证 `tests/unit/test_daemon.py:989-996`。

### Q2-3 "最后一个真实页"保护规则与判断依据

- **[Fact]** 计数口径：遍历 `self.info.values()`，统计 `type=="page"` 且 `is_real_page` 的条目；`<= 1` 时禁止关页。证据：`daemon.py:737`。
- **[Fact]** `is_real_page`：`type=="page"` 且 URL 不以内部前缀 `INTERNAL` 开头；`is_blank` 把 about:blank / newtab 等视为空页。证据：`daemon.py:447-455`。
- **[Observation]** `self.info` 是 daemon 侧 CDP 事件缓存：启动时快照一次，之后由 `Target.targetCreated` / `Target.targetInfoChanged` 事件更新；回收判断时**不发** `Target.getTargets` 做新鲜交叉校验。证据：`daemon.py:1018`（启动快照）、`daemon.py:850-857`（事件更新）、`daemon.py:737`（仅读缓存）。
- 边界条件：若缓存滞后（页面已被外部关闭/新开而事件未到），最后真实页计数可能失真；本审计未运行实测，属 Observation。

---

## Q3 · Daemon 竞争与重启

### Q3-1 多客户端如何争抢同一个 daemon

- **[Fact]** 启动争抢由 `_spawn_lock`（文件锁）串行化；并发启动者通过 PID 指纹 / ping 握手加入已在启动的 daemon，而不是再起一个。证据：`admin.py:279`、`admin.py:715`、`admin.py:784`；测试 `tests/unit/test_admin.py:253-259`。
- **[Fact]** 存活判定走 ping 握手（非裸 connect），防止崩溃后 stale `.port`/socket 文件 + 端口复用把无关进程误认成本 daemon；ping 回包带 `pid`、`browser_kind`、`endpoint`、`protocol`、`active`、`cdp`（脱敏标签 + 哈希）。证据：`daemon.py:1065-1077`、`admin.py:391`。
- **[Fact]** 浏览器一致性校验 `_check_same_browser`：daemon 实际驱动的 endpoint 必须与 `BU_CDP_URL`/`BU_CDP_WS` 指向一致，否则 `browser-mismatch` 拒绝。证据：`admin.py:448-457`；测试 `tests/unit/test_admin.py:205-215`。
- **[Fact]** `active` 计数：ping 时统计"其他 owner 仍持有 tab 且未空闲"的数量，供协议升级判断使用。证据：`daemon.py:1072-1073`；测试 `tests/unit/test_daemon.py:1328-1346`。

### Q3-2 协议不匹配：重启还是不重启，代价与风险

- **[Fact]** 判定函数 `_restart_if_outdated(name, may_restart=True)`，分支矩阵（证据 `admin.py:422-445`，测试 `tests/unit/test_admin.py:218-241`）：
  1. ping 无应答，或协议号相同 → 不动，返回 False（`admin.py:428-429`）。
  2. daemon 协议 **高于** 本客户端 → **永不停止**，报错 "it is newer than this client"（`admin.py:430-431`）。
  3. `may_restart=False` → **不重启**，"it must stay as provisioned"（`admin.py:432-433`）。
  4. `browser_kind != "local"`（cloud / cdp）→ **不重启**：cloud 重启会停掉计费浏览器；cdp 重启后新进程不知道该驱动哪个浏览器（`admin.py:434-436`）。
  5. `active > 0`（其他 agent 在用）→ **不重启**，"other agents are using it"（`admin.py:437-438`）。
  6. 仅当 local、空闲、旧协议、允许重启 → `restart_daemon(name)`，返回 True（`admin.py:439-441`）。
- **[Fact]** 不重启分支统一抛 `RuntimeError("daemon-outdated: … Nothing was sent. …")`，请求不发出。证据：`admin.py:442-445`。
- **[Observation]** 代价：旧协议客户端在上述 2–5 分支下只能升级自身或手动 `--reload`，不能自动恢复；风险是两个不同版本客户端反复重启同一 daemon 造成抖动，故 newer 方向 fail-closed。

### Q3-3 谁有权限杀死 daemon

- **[Fact]** 可终止 daemon 的路径（代码内）：
  1. IPC `meta=shutdown`：任何能连上通道的客户端可发；Windows 需 token 匹配，POSIX 受 socket 0600 约束。证据：`daemon.py:1079-1086`。
  2. `admin.restart_daemon`：先 `identify()` 取活 daemon PID 并核验身份，再信号终止；带 spawn 锁。证据：`admin.py:904-950`。
  3. 进程信号：SIGTERM / SIGHUP 走正常 shutdown 流程（第二次信号才用默认动作）。证据：`daemon.py:1362-1377`。
  4. `BH_OWNER_PID` 看门狗：所指父进程退出后 daemon 自行退出。证据：`daemon.py:1302-1312`。
  5. CDP 断连看门狗 `_watch_cdp`：浏览器侧 WebSocket 结束即 `stop.set()`。证据：`daemon.py:1350-1359`。
  6. 空闲退出：`exit_when_idle`（`BH_IDLE_TIMEOUT`）与 sweep 内 `IDLE_EXIT` 退出；owned browser 在空闲退出前检查是否仍有其他 CDP 客户端附着。证据：`daemon.py:1039-1055`、`daemon.py:771-775`、`daemon.py:1388-1397`。
- **[Observation]** POSIX 上 owner 身份本身**不构成** shutdown 授权边界：同用户本地进程只要能连 socket，即可发 shutdown meta；跨用户则被 socket 权限挡在门外（Hypothesis，未实测）。
- **[Fact]** 外部嵌入方（browser-use 1.0.8）另有一条 SIGKILL 路径 `_kill_daemon`（不经 shutdown 请求、不做退出清理）。证据：`audit/_scratch/bu108/browser_use/browser/browser.py:371-382`。

---

## Q4 · Raw CDP 隔离与隐私

### Q4-1 raw CDP 直通下隔离靠什么保证

- **[Fact]** 三组守护检查：
  1. **Tab 级**：`Target.attachToTarget` / `closeTarget` / `activateTarget` 走 `_holder` / `_mine` 校验，非本 owner 的页返回 `tab-taken`。证据：`daemon.py:43`（GUARDED 常量）、`daemon.py:598-606`、`daemon.py:1188-1214`。
  2. **Session 级**：显式 `session_id` 发送前查 `session_owner.get(sid, key) != key`。证据：`daemon.py:1232-1235`；set_session 兼容路径同构检查 `daemon.py:1181`。
  3. **Frame 级**：iframe/worker 原始 attach 需证明落在本 owner tab 内。证据：`daemon.py:1215-1218`。
- **[Fact]** 非 GUARDED 的普通 CDP 命令，在无显式 sid 时路由到本 owner 当前 tab 的会话；stale session 只重试挂接同一 tab，绝不换到别的 tab。证据：`daemon.py:1239-1253`。
- **[Observation]** 隔离强度依赖"调用方拿不到他者 sessionId / targetId"：`Target.getTargets` / `getTargetInfo` 经 `_visible` 对他者 tab 清空 url/title，但 targetId 本身仍在列表中。证据：`daemon.py:556-564`、`daemon.py:1227-1230`。

### Q4-2 browser-wide session（Target.attachToBrowserTarget）在所有权语义上是否被特殊对待

- **[Fact]** 是，被特殊对待：浏览器级会话只加入 `browser_sessions` 集合，**不**登记进 `session_owner`；注释明确 "browser-wide, like no session"。证据：`daemon.py:530`、`daemon.py:1220-1221`。
- **[Fact]** 事件路由对 browser session 单独分支：事件归属按 `frameId → tab_owner`，再按 `downloads[guid]` 兜底；不按 session_owner 归属。证据：`daemon.py:824-833`；测试 `tests/unit/test_daemon.py:654-671`。
- **[Observation]** 命令方向上，browser session 的 sid 未登记，命中 session 校验的默认值分支（详见专项 A）；这是隔离语义上的已知薄弱点。

### Q4-3 下载事件、隐私敏感动作如何归属

- **[Fact]** 下载归属：`Browser.downloadWillBegin` 按 `frameId` 找到所属 tab 的 owner，记录 `downloads[guid]=key`；`downloadProgress` 沿用；`completed`/`canceled` 后摘除。证据：`daemon.py:829-833`。
- **[Fact]** JS 对话框按 owner 隔离：`Page.javascriptDialogOpening` 记入 `o.dialog`，`dialogClosed` 清除；他者对话框不出现在本 owner 的 `pending_dialog`。证据：`daemon.py:838-841`、`daemon.py:1099-1100`。
- **[Fact]** 验证码事件：归属本 owner 当前 tab 的 solve 只投递给该 owner；无归属/无 tab 的状态广播给全部；`BU_CAPTCHA_SOLVER=off` 时不重新打开。证据：`daemon.py:867-876`、`daemon.py:912-959`。
- **[Fact]** 隐私可见性：他者 tab / 未知 iframe 的 url、title 对本 owner 置空。证据：`daemon.py:556-564`。
- **[Observation]** 录音文件按 daemon 名 + owner 分标记，共享 daemon 的 agent 互不覆盖录音。证据：`src/browser_harness/recorder.py:94-97`。

---

## Q5 · 与 browser-use 1.0.8 的版本缝隙

### Q5-1 版本固定关系（PyPI 元数据实证）

- **[Fact]** `browser-use-1.0.8` wheel 的 `METADATA` 第 21 行：`Requires-Dist: browser-harness==0.1.13`（精确钉死，非范围约束）。证据：`audit/_scratch/bu108/browser_use-1.0.8.dist-info/METADATA:21`。
- **[Fact]** 0.1.13 wheel 的 `daemon.py` 中 `owner` / `session_owner` / `tab_owner` / `PROTOCOL` 均零命中：0.1.13 不存在 owner 归属体系，也无协议号握手。证据：`audit/_scratch/bh0113/browser_harness/daemon.py`（grep 零结果）；0.1.15 对照 `daemon.py:509-513`、`_ipc.py:30`。
- **[Fact]** 0.1.13 helpers 函数表中不存在：`release_tabs`、`list_browsers`、`use_browser`、`name_session`、`mark_tab`、`solve_captcha`、`captcha_note`。证据：`audit/_scratch/bh0113/browser_harness/helpers.py` 函数清单；0.1.15 对照 `helpers.py:441-475`、`helpers.py:538-590`。

### Q5-2 browser-use 1.0.8 是否使用 owner API

- **[Fact]** 1.0.8 源码全量 grep 未见 `BH_OWNER`、请求体 `owner` 字段、owner 相关 helper 调用；`_daemon_request` 构造的请求 dict 不含 `owner`。证据：`audit/_scratch/bu108/browser_use/browser/browser.py:350-356`。
- **[Fact]** REPL worker 仅包装/复用 0.1.13 已有的 helpers：`current_tab`、`switch_tab`、`cdp`、`_send`、`http_get`、`wait_for_network_idle`；验证码由 worker 自带 `repl/captcha.py` 注入 `solve_captcha` 命名空间，不依赖 harness helper。证据：`audit/_scratch/bu108/browser_use/repl/worker.py:324-334`、`worker.py:344-347`、`worker.py:496-536`、`worker.py:677-711`、`worker.py:908`。
- **[Fact]** 共享 `Browser('chrome')` 不带 owner 身份：`_daemon_name()` 回退链 `self.name → BU_NAME → 'default'`；请求不带 owner，daemon 侧落 `"default"`。证据：`browser.py:1036-1037`；对照 `daemon.py:1087`。

### Q5-3 哪些能力在 1.0.8（钉 0.1.13）上不可用或不生效

- **[Fact]** 多 owner tab 隔离 / `tab-taken` / `tab-released` 语义：0.1.13 无此机制，1.0.8 运行时多个 Agent 共享同一无归属 daemon。证据：0.1.13 daemon.py grep 零结果；0.1.15 机制见 Q1/Q2。
- **[Fact]** owner 协作 API（`name_session` / `mark_tab` / `release_tabs` / `list_browsers` / `use_browser`）：0.1.13 helpers 无定义，1.0.8 不调用。证据：见 Q5-1 函数清单。
- **[Fact]** 0.1.14 引入的 daemon 侧能力在 0.1.13 上不存在：浏览器级会话下载归属、idle exit（`BH_IDLE_TIMEOUT`）、remote-browser 断连 fail-closed、cdp-mismatch 校验、`BH_OWNS_BROWSER`、协议号握手。证据：0.1.13 `daemon.py` 无 `OWNS_BROWSER`/`idle_exit`/`PROTOCOL`；0.1.15 对照 `daemon.py:127`、`daemon.py:528`、`_ipc.py:30`、`admin.py:448-457`。
- **[Observation]** 反向兼容：0.1.15 daemon 保留了 0.1.13 客户端使用的 `meta=set_session` 旧协议路径（测试名直指 0.1.13 行为）。证据：`daemon.py:1185-1194`；测试 `tests/unit/test_daemon.py:674-700`。即"新 daemon 接旧客户端"有兼容路径；"旧 daemon 接新客户端"无对应能力。
- **[Fact]** 1.0.8 源码注释本身已按 0.1.13 行为描述 daemon 退出时延（"browser-harness 0.1.13 daemons exit ~20 s after acknowledging shutdown"）。证据：`browser.py:385-390`。

---

## 专项验证 A–F（逐条：判定 + 证据 + 反面检验）

### A. attachToBrowserTarget 的 sid 是否未登记 session_owner；默认值放行是否成立

**判定：部分成立（机制成立，实际暴露面需限定）。**

- **[Fact] 登记点（session_owner 写入处，共 4 处）**：
  1. `_session` 挂接：`daemon.py:685` —— `o.tabs[tid], self.session_owner[sid] = sid, key`。
  2. 自动挂接的 iframe/worker 事件：`daemon.py:823` —— `self.session_owner[params["sessionId"]] = key`。
  3. 旧协议 `set_session` meta：`daemon.py:1187` —— 同构赋值。
  4. 原始 `Target.attachToTarget`：`daemon.py:1223` —— `self.session_owner.setdefault(result["sessionId"], key)`。
- **[Fact] 校验点（共 2 处）**：`daemon.py:1181`（set_session）、`daemon.py:1233`（显式 sid 发送），均为 `self.session_owner.get(sid, key) != key`。
- **[Fact]** browser session 只进 `browser_sessions`（`daemon.py:530`、`daemon.py:1220-1221`），从未进 `session_owner`。
- **[Fact]** 默认值 `key` 等于当前调用方 key：未登记 sid 命中默认值 → 校验表达式为 False → **默认放行**。
- **反面检验 / 边界**：
  - 未知 sid 在 Chrome 侧会回 "Session with given id not found"；显式 sid 分支（`daemon.py:1235`）不做 stale 重试，错误直接上抛，因此不能借此驱动一个**已存在的他者会话**。stale 重挂逻辑只在无显式 sid 分支（`daemon.py:1246-1252`）。
  - 真实薄弱面是**浏览器级会话**：sid 有效、Chrome 认账，但 sid 归属检查被默认值绕过；可向 browser target 发浏览器级命令。
  - 测试 `tests/unit/test_daemon.py:654-671` 覆盖的是 browser session **事件**归属（按 frameId/guid），未覆盖 browser session **命令**的 sid 校验。

### B. `_shares_parents_owner()` 是否只拒绝破坏性操作，cdp/js 仍可驱动父 owner current tab

**判定：成立。**

- **[Fact]** 共享父 owner 的判定：无 `BH_OWNER` 且 `CLAUDE_CODE_CHILD_SESSION=="1"`。证据：`helpers.py:109-111`。
- **[Fact]** 拒绝分支仅 2 个 helper：`release_tabs`（`helpers.py:445`）、`close_tab`（`helpers.py:523`），均调 `_refuse_for_shared_owner`（`helpers.py:114-117`），报错 "owner-shared: … Nothing was sent. Set BH_OWNER=<name>"。
- **[Fact]** 绕过该分支的 helper（无 refuse、无 per-call warn）：`cdp`（`helpers.py:96-103`）、`js`（`helpers.py:684`）、`goto_url`（196）、`click_at_xy`（234）、`type_text`（256）、`fill_input`（268）、`press_key`（355）、`scroll`（386）、`capture_screenshot`（391）、`current_tab`（477）、`switch_tab`（501）、`new_tab`（510）、`iframe_target`（531）、`wait_for_element`（603）、`upload_file`（746）、`http_get`（788）等。
- **[Fact]** `_warn_shared_owner`（`helpers.py:120-124`）不在 helper 调用路径上，仅被 CLI 启动期调用一次：`run.py:471`。
- **[Fact]** 未设 `BH_OWNER` 时 `OWNER` 继承父会话 id（`_ipc.py:36`），故上述绕过 helper 驱动的正是父 owner 的 current tab；设置 `BH_OWNER` 后拒绝解除。测试佐证：`tests/unit/test_helpers.py:700-714`（仅 close/release 被拒；设 `BH_OWNER=child` 后两者都发出）。
- **边界**：本结论针对 harness helpers 层；daemon 侧 tab/session 归属检查对这些命令仍生效（见 Q4-1），共享父 owner 只是免去了 helper 层的额外破坏性操作拦截。

### C. `_release` / `_release("idle")` 默认是否真的不关 tab；最后真实页计数是否只依赖 self.info

**判定：部分成立。**

- **[Fact] idle 默认不关 tab：成立。** 当前 tab 在 idle 模式跳过；其余仅 detach 留开。证据：`daemon.py:733-735`、`daemon.py:742-746`；测试 `tests/unit/test_daemon.py:970-986`。
- **[Fact] 最后真实页计数只遍历 `self.info`：成立。** 证据：`daemon.py:737`。
- **[不成立部分]** "self.info 是客户端上报" 的说法：`self.info` 是 **daemon 侧 CDP 事件缓存**（启动快照 + targetCreated/InfoChanged 事件更新），不是客户端上报字段。证据：`daemon.py:1018`、`daemon.py:850-857`。
- **[Fact] 无服务端新鲜交叉校验：成立。** `_release` 时不调用 `Target.getTargets` 刷新；计数完全基于缓存。证据：`daemon.py:726-750` 全段无 getTargets 调用。
- **[Fact] 环境变量开关**：`BH_CLOSE_IDLE_TABS=1` 让空闲回收也关自建未标记非当前页（`daemon.py:736`）；`BH_OWNER_IDLE`（默认 1800）、`BH_OWNER_FORGET`（默认 86400）（`daemon.py:144-145`）。
- **Observation**：缓存滞后期间，最后真实页保护可能误判；未运行实测。

### D. daemon-outdated 是否存在"永不停止更新协议的 daemon"的分支；条件、调用方、后果

**判定：成立。**

- **[Fact] 永不停止分支（共 4 类）**，见 Q3-2 矩阵：newer daemon（`admin.py:430-431`）、`may_restart=False`（`432-433`）、非 local kind（`434-436`）、`active>0`（`437-438`）。唯一重启分支：local + 空闲 + 旧协议 + 允许（`439-441`）。
- **[Fact] 调用方**：`ensure_daemon` 默认 `may_restart=True`（`admin.py:667` 一带，函数定义 `admin.py:644`）；`require_existing_daemon` 传 `may_restart=False`（`admin.py:880` 一带，定义 `admin.py:859`）。
- **[Fact] 后果**：抛 `RuntimeError("daemon-outdated: … Nothing was sent. …")`，daemon 继续运行，本次请求不发出；测试断言重启函数零调用。证据：`admin.py:442-445`、`tests/unit/test_admin.py:239-241`、`tests/unit/test_admin.py:245-250`。
- **[Fact] "must stay as provisioned" 语义**：`require_existing_daemon` 路径假定该 daemon 由外部编排者预置了 scoped 传输（cdp/cloud），客户端不得代其重启；报错指引 "Have its orchestrator restart it."。证据：`admin.py:432-433`、`admin.py:859-903`。
- **边界**：`may_restart=False` 与 `kind != local` 是叠加保护；即使 local，被 `require_existing_daemon` 取得的 daemon 也不自动重启。

### E. browser-use 1.0.8 是否固定 browser-harness==0.1.13；是否零使用 owner API；共享 Browser('chrome') 是否不带 owner 身份

**判定：成立（三点均有元数据/源码证据）。**

- **[Fact] 版本钉死**：`METADATA:21` `Requires-Dist: browser-harness==0.1.13`。
- **[Fact] 零使用 owner API**：1.0.8 源码无 `BH_OWNER`/请求 `owner` 字段/owner helper 调用；`_daemon_request` 请求体不含 owner。证据：`browser.py:350-356`；worker 复用清单 `worker.py:324-334, 496-536, 677-711`。
- **[Fact] 共享 `Browser('chrome')` 不带 owner 身份**：`_daemon_name` 回退 `'default'`，daemon 侧兜底 `"default"`。证据：`browser.py:1036-1037`、`daemon.py:1087`。
- **边界**：1.0.8 可通过 `BU_NAME` 区分独立 daemon 名，但这只隔离 daemon 实例，不提供 owner 归属；在 0.1.13 daemon 上 owner 体系本身不存在。

### F. 0.1.15 是否使用了 CHANGELOG 0.1.14 条目之外的未记录变更

**判定：成立（存在未记录变更；CHANGELOG 当前无 0.1.15 段落）。**

- **[Fact] 文档现状**：`CHANGELOG.md` 顶部版本段为 `## 0.1.14`，**不存在 0.1.15 段**；`pyproject.toml` 版本号为 `0.1.15`。证据：`CHANGELOG.md:3`、`pyproject.toml:6`。
- **[Fact] 版本间实测 diff**（0.1.14 wheel ↔ HEAD 源码，`audit/_scratch/bh0114/`）：仅 `helpers.py`、`run.py`、`telemetry.py` 三文件差异；`daemon.py` / `admin.py` / `auth.py` / `_ipc.py` / `recorder.py` / `report.py` / `video.py` / `video_render.py` / `mcp_server.py` 与 0.1.14 一致。
- **[Fact] 无对应条目的行为（0.1.15 相对 0.1.14）**：
  1. **cli_event 载荷瘦身**：不再发送 `task`（脚本文本，0.1.14 截断 20000 字符）与 `output`（stdout/stderr 流尾），只保留 `task_length` / `output_length`。证据：`telemetry.py:282-315`（HEAD）对照 0.1.14 wheel 同名段；`run.py:323-356`。
  2. **trace 步骤不再记录参数与异常文本**：0.1.14 的 `_step_args`（300 字符截断）与 `error=str(exc)[:300]` 被移除；步骤只记 helper 名，错误记布尔位；helper 名经 `helpers._OWN_NAMES` 快照归类，`agent_helpers.py` 注入的名字归 `"custom"`。证据：`helpers.py:843`、`run.py:207-221`、`run.py:226`、`run.py:234-237`。
  3. **错误归类改为内置异常类名**：`_error_class` 取 MRO 中最近的内置类名；`SystemExit` 的 error_message 记 `"SystemExit"` 而非退出码字符串/stderr 尾。证据：`run.py:212-216`、`run.py:330`、`run.py:344`。
- **边界 / 防误判**：0.1.14 CHANGELOG "Feedback and run reports" 段的 scrubbing 宣称（`CHANGELOG.md:40`）针对 `report_issue` / `report_run`（`report.py` 的 library 事件），**不能**当作 cli_event 已 scrub 的证据；0.1.14 源码 cli_event 确实仍发送 task 文本、参数与输出尾（diff 实证）。本轮未拿 0.1.14 条目充当 0.1.15 证据。

---

## 附：阶段一未决 / 需阶段二注意的点

1. 仓库内 `skills/`、`interaction-skills/`、`agent-workspace/domain-skills/`（209 文件）本轮仅清点数量与用途，未逐篇核对内容；若考古包把某 skill 文档陈述升格为代码事实，需在阶段二逐条回查。
2. `auth.py`（569 行）、`macos.py`、`video*.py`、`report.py`、`recorder.py` 已通读主干，未发现与 Q1–Q5 冲突的归属/隔离机制；细节如与 package 陈述冲突，阶段二再回查具体行号。
3. 本轮所有 Observation 均未在真实 Chrome / 真实 IPC 上运行验证；测试为单元级（mock CDP），集成测试未执行。

---

## Errata（冻结后修正）

> 本段为阶段二复核及第二复核者对照后追加，阶段一原结论原文保留不动；以下为补充与未覆盖项，不构成对原判定的推翻。

1. **[补充 · 专项 F]** 0.1.15 相对 0.1.14 的未记录变更还包含：stderr 不再被 `_StreamTail` 流式包装截取（0.1.15 仅 `run.py:260` 的 `_CountingStream` 计字符数）；配套删除常量/类 `MAX_TASK_LENGTH`（0.1.14 `telemetry.py:22`）、`_MAX_STEP_ARGS_LENGTH`、`_MAX_OUTPUT_LENGTH`（0.1.14 `run.py:253`）、`_StreamTail`（0.1.14 `run.py:256`，stdout/stderr 包装见 `run.py:313-314`）。HEAD 内 grep 四者零命中；0.1.14 wheel 均在。原 F-#1 所述"output 流尾不出域"成立，但未单列包装层移除这一配套清理。
2. **[补充 · 专项 C / Q2]** 显式 `close_tab` meta 命令只过 `_mine()` 归属校验即 `Target.closeTarget`，**不经过**释放路径 737 行的"最后真实页"判断；即显式关闭可关掉全局最后一个真实页（`daemon.py:1144-1152`）。原 Q2 状态机描述的 last-real-page 保护仅覆盖 `_release`（idle/forget/done）路径，未声明此例外。
3. **[补充 · Q4]** GUARDED 清单（`daemon.py:43`）仅覆盖 `closeTarget`/`attachToTarget`/`activateTarget` 三个方法；未列入的 `Target.detachFromTarget`、`Target.setAutoAttach`、`Target.createTarget` 及 `Browser.*` 方法不经 `_holder` 归属校验直达 CDP。原 Q4 隔离三层守护结论未展开此方法面边界。

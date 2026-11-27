# 02 Engineering Knowledge — browser-harness（47 EK · EK Graph）

> 宽底座：推理原材料。每条 EK 声明 links（六类边）+ 证据来源。证据格式 `file:line` 或 `符号@file`。EK 是图节点，KO 从图上按 R1-R4 聚合（见 03）。
> 证据基准：commit afbcc381（2026-09-07）。行号为实测。
> 本文件为 corpus 合并版（原包 45 EK + Reconciliation 修正：E-01/E-02/E-03 + M-01..05 → EK-46/47 新增、EK-04/34/35/41 修订）。

## A. 核心机制（EK-01~12）

**EK-01 stdin-heredoc 执行模型（L1 · 核心机制）**
CLI 以 `browser-harness <<'PY'` 把 agent 代码经 stdin 交给 run.py，`exec(code, globals())` 在 CLI 进程内执行；helpers 模块先 import 再 exec，所以 helper 名对 agent 直接可见。无框架/无 recipes/无 rails。
Evidence: `_install_helper_trace()@run.py:118-140`；`exec(code, globals())@run.py:330`
Links: mechanism→EK-02, EK-04；subsystem→EK-03

**EK-02 helper 追踪包装：500 步上限 + args 截断（L2 · 核心机制）**
_traced 包装器在每个 helper 调用后观察并记录：调用了哪个 helper、args（截 300 字符）、耗时、结果摘要（截 500 字符）；步数超 500 抛 StopIteration。截断防 trace/telemetry 记录爆炸。录制/遥测都建立在此观察钩子上。
Evidence: `_traced@run.py:169-231`（MAX_TRACED_STEPS=500）
Links: mechanism→EK-01；causal→EK-31（telemetry capture）；dependency→EK-34（recorder observe）

**EK-03 agent_helpers 运行时注入：自愈能力的实现点（L2 · 核心机制）**
helpers.py 底部 `_load_agent_helpers()`：若 BH_AGENT_WORKSPACE/agent_helpers.py 存在，用 importlib 动态加载并把其公共名注入 helpers 模块 globals。agent 在执行中写的缺失 helper 立即成为新原语——"harness 每次运行自我改进"的机制点。核心 src/ 受保护，agent 扩展被隔离在独立工作区。
Evidence: `_load_agent_helpers()@helpers.py:655-668`；agent-workspace/agent_helpers.py（空模板）
Links: mechanism→EK-01；causal→EK-41（domain-skills 策略）；constraint→EK-05（注入到受控命名空间）

**EK-04 daemon 中间人架构：会话权威与任务进程分离（L2 · 核心机制）**
每次 CLI 调用是独立进程，但 CDP WS 由长驻 daemon 持有；helper 经 IPC（_send → ipc.connect/request → daemon.handle → cdp.send_raw）转发。CLI 进程崩溃不丢浏览器会话；daemon 是会话唯一权威。超时族三层预算（Reconciliation E-02 细化）：IPC_CONNECT_TIMEOUT_SECONDS=5.0（连接）、DEFAULT_IPC_RESPONSE_TIMEOUT_SECONDS=5.0（响应）、SCREENSHOT_IPC_RESPONSE_TIMEOUT_SECONDS=60.0（截图，注释 "Keep their IPC socket alive within the caller's existing 90-second process budget"）；超时抛 _IPCResponseTimeout 且携带 detail（M-05 修复：裸类导致 str(exc) 为空，caller 报错无细节）。
Evidence: `_send@helpers.py:26-51`（超时族常量 helpers.py:31-36）；`serve()@_ipc.py:167`；`class Daemon@daemon.py:16`
Links: subsystem→EK-18（IPC）；dependency→EK-05；contrast→EK-42（双锁）

**EK-05 单 daemon 单可变 attached tab（L2 · 核心机制）**
Daemon 持有 session/target_id/dedicated_target_id；set_session 切换时旧会话 Network.disable + 新会话 4 域（Network/Page/Runtime/Target）enable 并行执行。SKILL.md 明示"one daemon has one mutable attached/current tab"；多 agent 共享同一浏览器 lane（串行），禁止为并发另起本地 daemon。
Evidence: `set_session@daemon.py:347-370`；`_set_session_state`；SKILL.md "One daemon has one mutable attached/current tab"
Links: causal→EK-07, EK-09；constraint→EK-11；subsystem→EK-04

**EK-06 stale session 自愈：Session with given id not found → recovery 链（L2 · 核心机制）**
CDP 报 "Session with given id not found"（Chrome 崩溃/导航销毁 target）时，daemon 置 _state=recovering，经 _begin_recovery → attach_first_page → 若仍 stale 则 _schedule_recovery 排队（_recoveries 集合），_recoveries_idle 等待 drain（2s 预算）。恢复成功后 events 清空，防旧事件污染新会话。
Evidence: `handle()@daemon.py:524-570`（异常分支）；`attach_first_page@daemon.py:211-267`；`_drain_recoveries@daemon.py:438`
Links: causal→EK-10（attach 回退）；mechanism→EK-43；dependency→EK-42

**EK-07 事件缓冲 + 会话过滤：后台 tab 不污染当前判断（L2 · 核心机制）**
daemon 维持 events deque(maxlen=500) 全局事件缓冲，drain_events() 取出；wait_for_network_idle 只认 active_session 的事件（后台轮询/SSE 页持续发 Network 事件会毒化 idle 判断）。旧会话切走时先 Network.disable 减少噪声。
Evidence: `drain_events@helpers.py:54-70`；`wait_for_network_idle@helpers.py:512-534`（active_session 过滤）；`set_session 旧会话 disable@daemon.py:347`
Links: mechanism→EK-05；dependency→EK-40（wait 族）

**EK-08 wait_for_network_idle：双条件空闲判定（L1 · 核心机制）**
轮询循环：drain 事件更新 inflight 集合 + last_activity；当 inflight 空且静默 ≥ idle_ms 才返回 True。表单提交/SPA 路由后无 DOM 变化的 XHR 场景用它。
Evidence: `wait_for_network_idle@helpers.py:500-534`
Links: mechanism→EK-07；causal→EK-40

**EK-09 tab marker（🐴）：受控 tab 的视觉标定（L1 · 核心机制）**
switch_tab 用 fire-and-forget JS 给标题加 "🐴 " 前缀标记受控 tab，agent 切走切回可辨认；页面 title 变化时检查并移除（slice(3)）。BH_TAB_MARKER=0 可关。discovery 事件页加载时禁用标记（防污染）。
Evidence: `_mark_tab/_unmark@daemon.py`；test_tab_marker_*（test_daemon.py:97-148）
Links: mechanism→EK-05；constraint→EK-11（named daemon）；causal→EK-09→EK-05

**EK-10 attach_first_page 分层回退：真实页→blank→newtab→inspect→新建（L2 · 核心机制）**
恢复/初始化时按序尝试：真实 page target → about:blank → chrome://newtab → chrome://inspect take_over → 最后 Target.createTarget 新建。每层失败捕获异常继续下一层；取 target 用 URL 前缀匹配而非全等（含 🌱 marker 的页）。
Evidence: `attach_first_page@daemon.py:211-267`
Links: causal→EK-06；mechanism→EK-12（连接回退）；contrast→EK-11

**EK-11 named daemon 专属 tab + 双锁：并行隔离的最后手段（L2 · 关键实现）**
named daemon（BU_NAME=r7k2）创建 dedicated_target_id 专属 tab；切换 session 时先取 _dedicated_target_lock 防并行 daemon 抢 tab。SKILL.md 定位为"last resort"——同一本地 Chrome profile，非另一 profile/进程；远程隔离优先于它。
Evidence: `_dedicated_target_lock@daemon.py:38`；`set_session 双锁@daemon.py:347-370`；SKILL.md "A named local daemon is a last resort"
Links: constraint→EK-05；contrast→EK-10；mechanism→EK-42

**EK-12 连接多路径回退：BU_CDP_WS→BU_CDP_URL→DevToolsActivePort→9222/9223（L2 · 核心机制）**
get_ws_url：显式 BU_CDP_WS 直用；BU_CDP_URL 则 /json/version 取 ws url（404 处理：Chrome 147+ 新 remote debugging 协议删了 /json/version，改 DevToolsActivePort by port 匹配）；否则扫描 ~/.config/chromium/DevToolsActivePort 或探测 9222/9223。浏览器进程身份经 /json/version 的 Browser 字段验证（防非 Chrome 伪装）。
Evidence: `get_ws_url@daemon.py:70-165`；`_local_chrome_listening@daemon.py:230`；test_local_chrome_listening_*（test_run.py:242-262）
Links: causal→EK-13；mechanism→EK-10；constraint→EK-27

**EK-13 Chrome 147+ /json/version 404 处理（L1 · 失败与修复）**
新版 Chrome remote debugging 移除了 /json/version（devtools endpoint 行为变化）；修复：URL 解析失败时按端口读 DevToolsActivePort 文件、用端口匹配定位 ws url，再验证 Browser 字段。测试 test_local_chrome_listening_accepts_devtools_response 固定该行为。
Evidence: `get_ws_url@daemon.py:118-165`；test_run.py:255-262
Links: causal←EK-12；mechanism→EK-27

## B. 生命周期与可靠性（EK-14~27）

**EK-14 permission-blocked 纪律：被拒/超时绝不自动重连（L2 · 关键决策）**
ensure_daemon 对 Chrome "Allow remote debugging?" 弹窗被拒/超时抛 permission-blocked，明示 "browser-harness did not retry or create another connection"；pending daemon 保持运行（弹窗由它的连接持有），等用户批准原弹窗。防"一个批准变无尽弹窗"（修复历史）。
Evidence: `ensure_daemon@admin.py:683-706`（permission_blocked 分支）；注释 "how a single approval turned into an endless prompt"
Links: constraint→EK-25；causal→EK-22；contrast→EK-24（云停止权威）

**EK-15 键盘物理键语义：press_key 完整 US layout（L2 · 关键实现）**
press_key 支持 shift 组合（如 "+"、"A"）、shortcut 意图（ctrl+l 清地址栏、ctrl+w 关 tab），修饰键按 browser OS 处理；对非字母键走 physical key 语义。测试 test_press_key 族固定行为（含 vk 对照）。
Evidence: `press_key@helpers.py:331-352`；test_helpers.py press_key 系列
Links: mechanism→EK-16, EK-17；contrast→EK-39（js 层）

**EK-16 fill_input 受控组件处理：select-all 直接 dispatch 而非 press_key（L2 · 关键实现）**
对 React/Vue 受控输入：清空用 select-all + Backspace 直接 dispatch 组合（避免 press_key 逐键触发 char 事件的组件状态问题）；输入后合成 input/change 事件（capture events 缺失时）；focus 用 Emulation.setFocusEmulationEnabled + finally 恢复。浏览器 OS 一次查询缓存（_browser_os）。
Evidence: `fill_input@helpers.py:388-448`；`_select_all_modifier@helpers.py:452-474`；test_fill_input_*（test_helpers.py:126-241）
Links: mechanism→EK-15；causal→EK-17；constraint→EK-40

**EK-17 _select_all_modifier 按浏览器 OS 而非本进程 OS（L2 · 边界与例外）**
macOS 浏览器用 Meta+A，其余 Ctrl+A；判定依据浏览器 User-Agent 的 OS（非运行 harness 的进程 OS）——同一 harness 机器控制远程/本地异构浏览器时修正键位。无 UA 时默认 Ctrl（Linux 假设备）。
Evidence: `_select_all_modifier@helpers.py:452-474`；test_fill_input_uses_ctrl_for_linux_browser
Links: dependency→EK-16；contrast→EK-15

**EK-18 IPC 双安全边界：AF_UNIX 0600 vs TCP+token（L2 · 关键实现）**
POSIX：AF_UNIX socket，bind 前 umask 077 → 0600（无 TOCTOU 窗口）；Windows：TCP loopback + secrets.token_hex(32)，daemon handle() 要求每个请求带 token（TCP 无 chmod 等价物，否则任意本地进程可发 CDP 命令）。port 文件原子写（tmp + os.replace）。
Evidence: `serve()@_ipc.py:167-190`；`handle token 校验@daemon.py`；`request()@_ipc.py:88`
Links: mechanism→EK-19；constraint→EK-20；subsystem→EK-04

**EK-19 ping/identify 端到端身份验证：防 stale port + port reuse（L2 · 关键实现）**
ping 只认 {"pong":true} 字典（拒绝 list/scalar——request 返回任意 JSON）；防 bare TCP connect 撞上无关进程（崩溃后端口被抢占）。identify 返回 daemon 自报 PID，供 restart_daemon 对"已验证身份"的进程发信号。
Evidence: `ping@_ipc.py:96-118`；`identify@_ipc.py:121-145`
Links: mechanism→EK-18；causal→EK-20, EK-22；constraint→EK-21

**EK-20 identify 严格 int 校验：拒 bool、0<pid<2^31（L2 · 安全实现）**
`type(pid) is int`（非 isinstance——isinstance(True,int) 为 True，敌意 daemon 可回 {"pid":True} 当 PID 1）；拒绝 0/负（os.kill(0) 杀进程组、os.kill(-1) 杀所有可杀进程）；上限 2^31（C pid_t 有符号 32 位，防 OverflowError）。Linux pid_max 实际 2^22。
Evidence: `identify@_ipc.py:136-140`（注释详述每个边界）
Links: mechanism→EK-19；causal→EK-22；constraint→EK-18

**EK-21 PID 指纹发布：原子写 + 防兄弟 daemon 竞争（L2 · 关键实现）**
daemon 启动即 _publish_own_pid（atomic_write_text tmp+replace）；ensure_daemon 读 pending pid 识别"正在启动的 daemon"（_parked_daemon_pid/_starting_daemon_pid），不重复 spawn；本地模式 spawn 前持 _spawn_lock（timeout=startup_wait），fd None 表示他人启动中 → 报错重试而非并发起进程。dead pid 清理受锁保护且校验 generation。
Evidence: `_publish_pid@admin.py:222`；`_spawn_lock@admin.py:256-318`；`ensure_daemon@admin.py:567-585`
Links: mechanism→EK-19；causal→EK-22；constraint→EK-23

**EK-22 restart_daemon 三重身份验证：identify PID + start-time 指纹 + generation（L2 · 关键实现）**
停 daemon 前：ipc.identify 拿自报 PID；_process_start_time 快照作次级身份（socket 可能在进程退出前消失，identify 返回 None 不证明进程死）；SIGTERM 前再验证（identify 同 PID 或 start-time 未变，否则 PID 可能被复用，跳过 SIGTERM）。pending（handshake-wait）daemon 用 fingerprint generation 匹配，owner 变化则不动手。
Evidence: `restart_daemon@admin.py:760-880`
Links: mechanism→EK-20, EK-21；causal→EK-14；dependency→EK-23

**EK-23 spawn lock：并发启动互斥（L1 · 关键实现）**
本地 Chrome 模式 spawn 前必须持 _spawn_lock（timeout=startup_wait=60s）；锁失败抛 "another browser-harness daemon is still starting"。清理（unlink dead pid）同样持锁，且只清"自己观察到的那一代"（generation 匹配）。
Evidence: `_spawn_lock@admin.py:256-318`；cleanup 分支 admin.py:647-663
Links: mechanism→EK-21；dependency→EK-22

**EK-24 cloud 计费权威：shutdown 失败保持 daemon 存活重试清理（L2 · 关键决策）**
cloud daemon 的 shutdown handler 先 PATCH /browsers/{id} {"action":"stop"}（计费结束 + profile 持久化）再确认；stop 失败时 daemon 保持存活（_shutting_down 回滚），让后续调用能重试清理——"stale Cloud daemon still owns a billable browser"，替换它会孤儿化计费浏览器。stop_remote_daemon 3 次重试。
Evidence: `handle shutdown@daemon.py:594-620`；`stop_remote_daemon@admin.py:733`；test_shutdown_keeps_daemon_alive_when_cloud_stop_fails
Links: constraint→EK-14；causal→EK-26；contrast→EK-25

**EK-25 ensure_daemon 自愈分级：真实 CDP call 探测 stale（L2 · 关键实现）**
daemon_alive 为真还不够——stale daemon 能答纯 Python 的 meta:* 但 CDP WS 已死；ensure_daemon 用真实 CDP 调用（Target.getTargets 要 "result"）探测，连试 2 次（0.5s 间隔）。stale 时 cloud→stop_remote_daemon（计费权威）、本地→restart_daemon。随后分级处理：Chrome 未开→拉起；inspect 未开→引导；弹窗→等批准。
Evidence: `ensure_daemon@admin.py:525-560`
Links: mechanism→EK-14；causal→EK-24；constraint→EK-22

**EK-26 cloud bootstrap 守卫：显式端点阻断自动云启动（L2 · 关键决策）**
自动 cloud bootstrap 条件：BU_AUTOSPAWN=1 + 未设 BU_CDP_URL/WS + 有 cloud auth + 无本地 Chrome 可连。显式 BU_CDP_URL/WS 即使为空值也阻断（test_explicit_bu_cdp_url_blocks_cloud_bootstrap）；bad stored auth 不崩溃（捕获后走本地/报错）。test_run.py 5 个 bootstrap 守卫测试固定该矩阵。
Evidence: `_should_autospawn_cloud@run.py`；test_run.py:73-177
Links: constraint→EK-24；mechanism→EK-27；causal→EK-12

**EK-27 本地 Chrome 身份探测：/json/version Browser 字段校验（L2 · 安全实现）**
_local_chrome_listening 对 127.0.0.1:9222/9223 发 HTTP，要求 /json/version 返回含 Browser 字段的 DevTools 响应——拒绝任意 TCP 服务伪装成 Chrome（test_local_chrome_listening_rejects_non_chrome）。
Evidence: `_local_chrome_listening@daemon.py:230-268`；test_run.py:242-262
Links: mechanism→EK-12, EK-13；constraint→EK-26

## C. 数据与隐私（EK-28~37）

**EK-28 日志脱敏：只留 endpoint 拓扑（L1 · 安全实现）**
_safe_connection_label 从 ws url 提取 scheme/host/port/path 拓扑，剥掉凭证与 query（密码/路径/查询参数全去除）。连接日志不泄真实端点身份。
Evidence: `_safe_connection_label@daemon.py:45`；test_safe_connection_label_removes_credentials_paths_and_queries
Links: mechanism→EK-29, EK-30；subsystem→EK-35

**EK-29 recorder URL secret scrub：code/token/secret→REDACTED（L2 · 安全实现）**
events.jsonl 里每条 URL 经 _URL_SECRETS 正则清洗：code/access_token/id_token/refresh_token/token/assertion/client_secret/client_info/session_state/api_key/sig/signature/auth/authorization/password/secret 等参数值 → REDACTED。防 OAuth 重定向把真实密钥落进共享文件夹。
Evidence: `_scrub_url@recorder.py:52-67`（_URL_SECRETS 正则）
Links: mechanism→EK-28；constraint→EK-34；causal→EK-36

**EK-30 telemetry 脱敏：FORBIDDEN_KEYS 17 项 + 长度上限（L2 · 安全实现）**
capture 前 _safe_properties 递归清洗：键命中 FORBIDDEN_KEYS（api_key/content/cookie/email/href/key/message/password/path/prompt/query/secret/selector/text/title/token/url/uri）即丢弃；嵌套 dict 递归；MAX_TASK_LENGTH 20000 截断。事件禁用 env（BH_TELEMETRY/BROWSER_HARNESS_TELEMETRY/ANONYMIZED_TELEMETRY 任一=1 即关）。
Evidence: `_safe_properties@telemetry.py:128-162`；`FORBIDDEN_KEYS@telemetry.py:26-42`
Links: mechanism→EK-28, EK-29；causal→EK-31

**EK-31 telemetry 捕获链：命令/步骤/输出尾全捕获（L2 · 关键实现）**
capture_cli_event 记录 command/step（argv）、output_tail（_StreamTail 20000 字符上限）、agent_name/run_id；agent 进程识别 _detect_agent_client（env/argv 嗅探）。detached 上报（_send_detached 后台线程/进程）。opt-out 可配（set_enabled 持久化 telemetry.json）。
Evidence: `capture_cli_event@telemetry.py:247-296`；`_send_detached@telemetry.py:189`
Links: dependency→EK-30；mechanism→EK-02；causal→EK-44

**EK-32 OAuth PKCE + 本地 callback server：浏览器认证（L2 · 关键实现）**
auth login 起本地 HTTPServer（127.0.0.1 随机端口，CALLBACK_PATH=/browser-use-cloud/callback），PKCE（S256）+ state 校验；_exchange_authorization_code 换 token；AuthRecord 存 config 目录（chmod 0600）。AUTH_TIMEOUT_SECONDS=600。
Evidence: `start_browser_auth@auth.py:209`；`complete_browser_auth@auth.py:252`；`_callback_server@auth.py:397`
Links: causal→EK-33；constraint→EK-24；mechanism→EK-18（本地 secret 存储）

**EK-33 device code 流程：SSH/headless 场景（L2 · 关键实现）**
start_device_auth 走 OAuth device flow（user_code + verification_uri_complete + interval=5s 轮询），适用于无本地浏览器可打开的远程/SSH 环境。
Evidence: `start_device_auth@auth.py:298`；`complete_device_auth@auth.py:326`
Links: contrast→EK-32；constraint→EK-26；mechanism→EK-38

**EK-34 recorder 帧预算：ACTIONS 过滤只读 helpers（L2 · 关键实现）**
observe() 只对 ACTIONS 集合内 helper 出帧（goto/click/type/fill/press/scroll/upload/tab 管理/wait 族）；只读 helper（js/page_info/capture_screenshot）不帧——避免 inspection-heavy 会话录制膨胀。_SETTLE_SECONDS=0.15 等页面绘制。录制失败被吞（绝不破坏 run）。auto-recording 为 opt-in 偏好（Reconciliation M-02：CLI 入口 `browser-harness recordings enable|disable`@run.py:351 → recorder.set_auto_recording；BH_RECORD=1/0 单进程覆盖）。
Evidence: `ACTIONS@recorder.py:31-44`；`observe@recorder.py:239`；`recordings 子命令@run.py:351`
Links: mechanism→EK-02；causal→EK-35；dependency→EK-29

**EK-35 video 流水线：init→compile→review→export（L2 · 核心机制）**
video 子命令四步：init_recording（读录制，可 require_explicit——meta 缺失或 auto 录制时拒绝："not an explicit recording"）→ compile（事件→beats：默认时长/旁白节奏/隐私校验 validate_privacy + SENSITIVE/ROUTE_UNSAFE 正则）→ review（时长预算 22-32s、narration cadence）→ export（HTML 模板 video-template.html + source manifest video-source.json 校验 hash；Reconciliation M-04：export 支持 --reviewed flag，区分已审与原始导出，video_render.export(recording, output, reviewed)）。HOUSE_STYLE 默认样式 dict。
Evidence: `init_recording@video.py:661`；`compile_brief@video.py:473`；`validate_privacy@video.py:438`；`run_cli export --reviewed@video.py:710`
Links: causal←EK-34；mechanism→EK-36；constraint→EK-29

**EK-36 video 隐私双层校验：SENSITIVE + ROUTE_UNSAFE（L2 · 安全实现）**
validate_privacy 对 reviewed 文本查 SENSITIVE 正则（@、onmicrosoft.com、tenant/user/object-id、GUID）；ROUTE_UNSAFE 更严（@、?#、://、onmicrosoft、GUID）用于路由/文件路径。发现即拒绝导出（不是静默清洗——"Validation（含 Contradictions 与 Counterexamples）必须保留"同精神）。
Evidence: `SENSITIVE/ROUTE_UNSAFE@video.py:27-38`；`validate_privacy@video.py:438`
Links: mechanism→EK-29；causal←EK-35

**EK-37 MCP stdio server：22 工具薄封装（L1 · 关键实现）**
mcp_server.py 用 @SERVER.tool 把 helpers 包装成 MCP 工具（browser_goto/click/type/fill/screenshot/js/cdp/http_get/recording 等 22 个）；_json_default 处理非 JSON 值。mcp_cli 提示 pip install 'browser-harness[mcp]'。与 CLI 同一 helpers 底座，无第二套语义。
Evidence: `browser_*@mcp_server.py:139-292`；`mcp_cli.py:1-16`
Links: subsystem→EK-01；mechanism→EK-04；contrast→EK-01（执行面不同：MCP 工具 vs heredoc）

## D. 边界与细节（EK-38~45）

**EK-38 http_get 代理路由：BROWSER_USE_API_KEY → fetch-use（L2 · 关键实现）**
设了 BROWSER_USE_API_KEY 时经 fetch_use.fetch_sync（bot 检测/住宅代理/重试）；否则本地 urllib + gzip 解压 + Mozilla UA。纯 HTTP 不占浏览器（ThreadPoolExecutor 批量）。
Evidence: `http_get@helpers.py:579-598`
Links: mechanism→EK-26；constraint→EK-33；contrast→EK-04（不经 daemon）

**EK-39 js() iframe 会话管理：attach→evaluate→detach（L2 · 关键实现）**
js(target_id=...) 先 Target.attachToTarget 拿临时 session，evaluate（await_promise=True）后 finally detach——防轮询循环每 call 累积一个活会话+事件流；Chrome 已丢的会话（iframe 导航/关闭）不算泄漏（detach 报 no session 即静默）。illegal top-level return 时重试包函数包装。
Evidence: `js/_js_evaluate/_detach_iframe_session@helpers.py:540-574`
Links: mechanism→EK-07；constraint→EK-05；contrast→EK-15（js 层 vs 物理键层）

**EK-40 wait_for_element visible：checkVisibility 祖先链（L2 · 关键实现）**
visible=True 用 checkVisibility({checkOpacity,checkVisibilityCSS})——遍历祖先链，尊重 display:none/visibility:hidden/opacity:0 继承；旧 Chrome 无 checkVisibility 回退单元素 getComputedStyle（注释明确：单元素检查看不到祖先的"是否渲染"状态）。wait_for_load 会漏 SPA（document complete 早于框架渲染），故需 wait_for_element。
Evidence: `wait_for_element@helpers.py:473-498`
Links: mechanism→EK-08；causal→EK-16；constraint→EK-07

**EK-41 domain-skills 发现：goto 附带站点技能列表（L1 · 核心机制）**
goto_url 时（BH_DOMAIN_SKILLS=1）按 hostname **首标签**找 agent-workspace/domain-skills/<site>/ 目录（Reconciliation E-01 修正：实际实现 `(hostname).removeprefix("www.").split(".")[0]`——youtube.com→youtube、www.example.co.uk→example，非完整 hostname；skills/ 单名单目录布局与此一致），把可用技能列表注入返回（agent 据此读技能再操作）。skills/ 内置 100+ 站点技能（amazon/xiaohongshu 等），AGENTS.md 策略："agent-generated when possible — hand-author only when necessary"。
Evidence: `goto_url@helpers.py:106-133`（`d = (AGENT_WORKSPACE / "domain-skills" / ...split(".")[0])`）；skills/ 目录
Links: causal←EK-03；subsystem→EK-03；constraint→EK-05

**EK-42 双锁序：shutdown 与 session 切换的死锁防护（L2 · 关键实现）**
shutdown 路径固定顺序：_session_state_lock → _dedicated_target_lock；set_session 同序获取。恢复任务 _active_recoveries 受同一锁族约束。锁序不变量防 session 切换与关机互锁。
Evidence: `shutdown@daemon.py:594-620`；`set_session@daemon.py:347-370`；`_active_recoveries@daemon.py:31`
Links: constraint→EK-06, EK-11；mechanism→EK-43

**EK-43 recovery 并发注册与 drain 预算（L2 · 关键实现）**
_begin_recovery/_finish_recovery 原子注册；_recoveries_idle 判空；drain 用 gather+timeout(2s)——超时不无限等（shutdown 不能被恢复拖死）。恢复后 _cleanup_stale 依 _session_replacements（≤32 dict）清理旧会话。
Evidence: `_begin_recovery/_finish_recovery/_drain_recoveries@daemon.py:380-445`
Links: mechanism→EK-06；dependency→EK-42；causal→EK-24（关机路径）

**EK-44 _StreamTail 输出截尾：telemetry 上下文（L1 · 关键实现）**
stdout tail 上限 20000 字符、stderr 500；输出尾部进 telemetry 事件属性。防止 CLI 大输出刷爆遥测 payload。
Evidence: `_StreamTail@run.py:241-270`
Links: dependency→EK-31；mechanism→EK-02

**EK-45 Windows UTF-8 重配置（L1 · 失败与修复）**
Windows 下 stdout/stderr reconfigure(encoding="utf-8")（#124(4) 修复）——防 locale 编码在中文/非 ASCII 页面报 UnicodeEncodeError。
Evidence: `sys.stdout.reconfigure@run.py`（注释 #124(4)；run.py:4 "Windows default stdout/stderr encoding is cp1252"）
Links: contrast→EK-44；subsystem→EK-01

## E. Reconciliation 新增（EK-46~47，独立审计 MISSING 项）

**EK-46 dialog 状态机：native 对话框的全局阻塞语义（L2 · 关键实现）**
daemon 订阅 Page.javascriptDialogOpening（存 params 至 self.dialog）/ Page.javascriptDialogClosed（清 None）（Reconciliation M-01）；meta=pending_dialog 返回 dialog 状态；page_info 在 dialog 打开时返回 {dialog: {type,message,...}} 而非页面信息——对话框冻结页面 JS 线程直到被处理（interaction-skills/dialogs.md）。这是 State/Data Flow 的关键节点：agent 必须先处理 dialog 才能继续读页。
Evidence: `dialog@daemon.py:439,628-631,739`；`page_info@helpers.py:145-152`；interaction-skills/dialogs.md
Links: mechanism→EK-07；constraint→EK-05；causal→EK-40（wait 族在 dialog 下的行为）

**EK-47 BH_DEBUG_CLICKS：点击可视化调试（L1 · 关键实现）**
click_at_xy 在 BH_DEBUG_CLICKS=1 时用 PIL 在截图绘制点击标记（debug_click_<n>.png，BH_TMP_DIR）——辅助 agent 校验坐标正确性（Reconciliation M-03）。与 _send 的 response 超时无关，属纯调试面。
Evidence: `click_at_xy@helpers.py:154-165`（_debug_click_counter + PIL ImageDraw）
Links: contrast→EK-02；subsystem→EK-15

---
## EK Graph 统计
- 总 EK：47（L1×12 / L2×35）
- 平均出边：47 条 EK 共 107 条边 → 2.3 条/EK
- 游离 EK：0（全部 ≥1 link）
- 覆盖类别：核心机制 12 / 生命周期与可靠性 14 / 数据与隐私 10 / 边界与细节 8 / Reconciliation 新增 2（dialog 状态机、debug-clicks）；其中关键决策 5、失败与修复 3、边界与例外 1

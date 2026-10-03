# Snapshot Artifact — browser-harness（ARCH-2026-10-04-001）

## 仓库身份
| 字段 | 值 |
|---|---|
| project | browser-harness |
| repository | https://github.com/browser-use/browser-harness |
| commit SHA | afbcc381b963040c19627d788e40c7e7663171ee |
| branch | main（浅克隆 grafted） |
| commit date | 2026-09-07 13:13:31 -0700 |
| subject | Merge pull request #757 from warun7/fix/video-export-nested-output |
| version（pyproject.toml） | 0.1.13（`name = "browser-harness"`） |
| license | MIT |
| requires-python | >=3.11 |
| 文件数 | 190（浅克隆） |
| src 代码行数 | 6,732（14 个模块） |
| tests 行数 | 4,099（12 个单元文件 + 1 集成文件） |

## 项目定位（README 原话）
"Connect an LLM directly to your real browser through one editable CDP websocket. The agent writes missing helpers as it works, so the harness improves with every task."
即：极简自愈浏览器 harness——LLM 经一根 CDP WebSocket 直连真实 Chrome，agent 在执行中自写缺失 helper，每次运行自我改进。README 明确："You will never use the browser again."

## 语言与运行
- Python ≥3.11；入口脚本 `browser-harness`（仓库根 939B 可执行文件，git checkout 本地启动器）
- 三个 console entry（pyproject `[project.scripts]`）：
  - `browser-harness` → `browser_harness.run:main`（stdin heredoc 执行 agent 代码）
  - `browser-harness-mcp` → `browser_harness.mcp_cli:main`（可选 MCP 服务器）
  - daemon 进程：`python -m browser_harness.daemon`（长驻中间人）

## 入口与模块地图
| 模块 | 行数 | 职责 |
|---|---|---|
| src/browser_harness/run.py | 413 | CLI 入口：命令分发（--version/--doctor/auth/skill/recordings/video/--update/--reload/--debug-clicks）、stdin 脚本执行（exec）、helper 追踪包装（_traced）、telemetry 捕获 |
| admin.py | 1588 | daemon 生命周期：ensure_daemon/restart_daemon/require_existing_daemon/stop_remote_daemon、spawn lock、PID 指纹、doctor、cloud 浏览器管理（start_remote_daemon/sync_local_profile） |
| daemon.py | 894 | 长驻进程：CDP WS 持有 + IPC relay（Daemon 类）、attach/recovery/session 切换、事件缓冲、stale session 自愈、shutdown 协商 |
| helpers.py | 668 | CDP wrapper + 浏览器原语（goto/click/type/fill/press/scroll/screenshot/tab 管理/js/wait/upload/http_get）+ agent_helpers 动态加载 |
| _ipc.py | 201 | IPC 管道：AF_UNIX（POSIX 0600）/ TCP loopback+token（Windows）、ping/identify 端到端身份验证、spawn 参数、原子 port 文件 |
| auth.py | 546 | Browser Use Cloud OAuth（PKCE + 本地 callback server + device code + API key stdin） |
| paths.py | 49 | 文件系统布局：BH_HOME/XDG、config/runtime/tmp/workspace 私有目录（0700） |
| recorder.py | 343 | 会话录制：每动作一帧 + trace 行（events.jsonl）、URL 脱敏、自动录制偏好 |
| telemetry.py | 308 | PostHog 匿名遥测（opt-out）、脱敏键、detached 上报 |
| video.py | 749 | 录制→视频：composition 编译/校验/审查/导出流水线（init/review/export） |
| video_render.py | 524 | HTML 模板渲染合成视频 |
| macos.py | 131 | macOS Chrome 远程调试授权（mac-approve） |
| mcp_cli.py | 16 | MCP 可选依赖提示入口 |
| src/mcp_server.py | 300 | MCP stdio server：22 个浏览器工具（browser_new_tab/goto/click/type/fill/...） |

## 核心数据结构 / 状态
- `Daemon`（daemon.py）：cdp / session / target_id / dedicated_target_id / events（deque maxlen=500）/ dialog / stop / _shutting_down / _recovery_tasks / _session_replacements（dict ≤32）/ _active_recoveries——单 daemon 一个可变 attached tab
- `AuthRecord` / `BrowserAuthStart` / `DeviceAuthStart` / `PendingCallback`（auth.py）
- recorder：录制 = 一个文件夹（meta.json + events.jsonl + 帧 jpg）
- video：`composition` dict（window.COMPOSITION，HOUSE_STYLE 默认样式）+ source manifest（video-source.json）
- `_ipc`：`_server_token`（Windows，token_hex(32)）；port 文件 {"port","token"}
- 环境变量族：BU_NAME/BU_CDP_WS/BU_CDP_URL/BU_AUTOSPAWN/BU_BROWSER_ID/BROWSER_USE_API_KEY + BH_HOME/BH_RUNTIME_DIR/BH_TMP_DIR/BH_AGENT_WORKSPACE/BH_TAB_MARKER/BH_DEBUG_CLICKS/BH_DOMAIN_SKILLS/BH_RECORD/BH_TELEMETRY/BH_REQUIRE_EXISTING_DAEMON/BH_OPEN_LIVE_URL

## 生命周期（关键路径）
1. `browser-harness <<'PY' ... PY` → run.py main → 命令分发 → ensure_daemon()（idempotent 自愈：stale daemon 探测→重启/云停；Chrome 未开→拉起；remote debugging 未开→打开 chrome://inspect 引导）→ _install_helper_trace() → exec(code)
2. daemon：start()（get_ws_url 多路径回退 → _PatientCDPClient 握手 → attach_first_page 分层回退）→ serve()（IPC handler 循环）
3. 每次 helper 调用：_send → ipc.connect/request → daemon.handle → cdp.send_raw
4. 关机：meta=shutdown → 置 _shutting_down → cancel/drain recoveries → stop_remote（cloud 计费停止）→ stop.set() → 清理 dedicated tab → cleanup_endpoint

## 测试体系
- pytest（`[tool.pytest.ini_options] pythonpath=["src"]`）
- 单元测试 12 文件：test_admin(1450)/test_daemon(847)/test_helpers(740)/test_run(301)/test_ipc(128)/test_macos(127)/test_recorder(96)/test_skill(35)/test_auth(34)/test_mcp_server(38)/test_mcp_cli(21)/test_video_render(22)/test_brave_origin(67)
- 集成 1 文件：tests/integration/test_js.py（193，需活浏览器/CDP）
- AGENTS.md 指出："Integration tests may need a live browser/CDP — prefer unit + doctor for routine PR gates"
- 关键测试揭示行为：test_admin/test_daemon 覆盖 spawn lock/PID 指纹/stale session/权限弹窗/cloud 停失败保持 daemon 存活；test_helpers 覆盖 fill_input 受控组件/focus emulation/wait 语义；test_run 覆盖 cloud bootstrap 守卫（BU_CDP_URL/BU_CDP_WS 阻断）

## 配置与治理机制
- .env 自动加载（仓库根 + agent-workspace，`os.environ.setdefault` 语义）
- 权限与安全：Windows TCP token 认证（_server_token）；POSIX AF_UNIX umask 077（无 TOCTOU）；BU_NAME 正则校验（防路径穿越）；日志 URL 脱敏（_safe_connection_label 只留 endpoint 拓扑）；recorder URL secret scrub（code/token/secret 等参数 → REDACTED）；telemetry FORBIDDEN_KEYS 17 项；录制 password 字段 text mask；restart_daemon 进程身份三重验证（identify PID / start-time 指纹 / fingerprint generation）
- 治理约束（SKILL.md/AGENTS.md 硬规则）：核心 src/ 受保护（"stays protected while the agent writes reusable helpers in its local workspace"）；agent 只编辑 agent-workspace/（agent_helpers.py + domain-skills/）；登录墙必须停下询问；不自动 activate_tab（后台操作原则）；远程浏览器计费停止询问
- 外部依赖：cdp-use==1.4.5、fetch-use==0.4.0、pillow==12.3.0、websockets==15.0.1；optional mcp==2.1.1；外部工具 profile-use（list/sync cookie）；Browser Use Cloud API（api.browser-use.com/api/v3）；PostHog（eu.i.posthog.com）

## 关键设计文档
- SKILL.md（仓库内 14,336B）：agent 工作流规范（tab 纪律/后台操作/网络空闲等待/焦点模拟/录制视频/dialogs 处理等）
- install.md：安装/连接/排障（chrome://inspect 引导、mac-approve、Snap Linux headless）
- AGENTS.md：代码优先级（Clarity/Precision/Low verbosity/Versatility）、提交纪律、domain-skills 策略（"agent-generated when possible — hand-author only when necessary"）
- interaction-skills/ 19 篇：connection/cookies/cross-origin-iframes/dialogs/downloads/drag-and-drop/dropdowns/iframes/make-video/network-requests/print-as-pdf/profile-sync/screenshots/scrolling/shadow-dom/tabs/uploads/viewport
- skills/ 目录：browser-harness 技能注册 + 100+ 站点 domain skills（amazon/bilibili/xiaohongshu/youtube/gmail 等）
- docs/MCP.md：MCP server 配置
- 注意：SKILL.md front matter `name: browser-harness`，而 AGENTS.md 声明 "Skill identity (name + trigger) = browser-use (do not rename)"——二者不一致，疑为版本演进遗留（记入 Candidates）

## 显著设计证据（snapshot 级）
- 自愈哲学：helpers.py 底部 `_load_agent_helpers()` 运行时注入 agent-workspace/agent_helpers.py 到 globals——"agent 写缺失 helper"机制的实现点
- 双身份验证：ping/identify 端到端（防 stale port + port reuse + PID reuse）；`type(pid) is int` 拒 bool、0<pid<2^31 防 os.kill 组信号
- 权限弹窗纪律："browser-harness did not retry or create another connection"——被拒/超时绝不自动重连，避免弹窗风暴
- cloud 计费权威：shutdown 失败保持 daemon 存活（_shutting_down 回滚）以便重试计费清理
- 后台操作原则：switch_tab 默认不 activate_tab；focus 模拟用 Emulation.setFocusEmulationEnabled 并 finally 关闭

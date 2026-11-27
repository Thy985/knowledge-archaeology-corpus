# 01 Project Layer — browser-harness（项目地图）

> 事实底座。全部可回溯到仓库实际内容（commit afbcc381）。浓缩自 snapshot_artifact.md，此处保留项目层关键事实。

## 1.1 是什么 / 怎么运行
- 极简 CDP 浏览器 harness：`browser-harness <<'PY' ... PY`（stdin heredoc）→ run.py exec 执行 agent 代码，helpers 预导入，daemon 自动启动并连接运行中的 Chrome（README、run.py HELP）。
- 三种运行形态：本地 Chrome（默认）、BU_CDP_URL/WS 指定端点（dedicated automation Chrome）、Browser Use Cloud 远程浏览器（计费）。
- pyproject version 0.1.13，license MIT，requires-python >=3.11。

## 1.2 模块地图（src/browser_harness/，行数实测）
| 模块 | 行数 | 职责 |
|---|---|---|
| run.py | 413 | CLI 分发 + stdin 执行 + helper 追踪 + telemetry 捕获 |
| admin.py | 1588 | daemon 生命周期（ensure/restart/stop）、spawn lock、PID 指纹、doctor、cloud 管理 |
| daemon.py | 894 | 长驻中间人：CDP WS + IPC relay、session 自愈、shutdown 协商 |
| helpers.py | 668 | CDP wrapper + 浏览器原语 + agent_helpers 动态注入 |
| _ipc.py | 201 | AF_UNIX/TCP+token 管道、ping/identify 身份验证 |
| auth.py | 546 | Cloud OAuth（PKCE/device code/API key） |
| recorder.py | 343 | 会话录制（events.jsonl + 帧） |
| telemetry.py | 308 | PostHog 匿名遥测 |
| video.py | 749 | 录制→视频编译/审查/导出 |
| video_render.py | 524 | HTML 模板渲染 |
| macos.py | 131 | mac-approve |
| mcp_cli.py / mcp_server.py | 16/300 | 可选 MCP stdio server（22 工具） |
| paths.py | 49 | 文件系统布局（0700 私有目录） |

## 1.3 入口
- CLI：`browser-harness`（root 可执行脚本 → `browser_harness.run:main`）
- MCP：`browser-harness-mcp`（`browser_harness.mcp_cli:main`，需 `browser-harness[mcp]`）
- daemon 进程：`python -m browser_harness.daemon`（subprocess.Popen 启动，detached）

## 1.4 生命周期（状态机要点）
- 冷启动：ensure_daemon()（admin.py:525）idempotent 自愈——stale daemon 用真实 CDP call 探测（非 ping）→ cloud/重启；Chrome 未开 → 拉起（_launch_browser）；remote debugging 未开 → 打开 chrome://inspect 引导用户勾选。
- 热调用：run.py → ensure_daemon 短路 → exec → helpers 经 IPC → daemon → CDP。
- 关机：meta=shutdown（daemon.py handle）→ 置 _shutting_down → cancel/drain recoveries（2s 预算）→ stop_remote（cloud PATCH 计费停止）→ stop.set() → 清理 dedicated tab → cleanup_endpoint。

## 1.5 测试体系
- pytest，unit 12 文件（test_admin 1450 行最大）+ integration/test_js（193 行，需活浏览器）。
- AGENTS.md 指示：routine PR gate 用 unit + doctor，integration 可能需要 live browser。

## 1.6 配置
- .env 自动加载（repo 根 + agent-workspace，setdefault 语义）；.env.example 仅 BROWSER_USE_API_KEY。
- BU_*：BU_NAME（daemon 名，默认 default）、BU_CDP_WS/BU_CDP_URL（显式端点，阻断 cloud bootstrap）、BU_AUTOSPAWN（cloud 自动起，opt-in）、BU_BROWSER_ID（cloud 浏览器 id）。
- BH_*：BH_HOME/BH_CONFIG_DIR/BH_RUNTIME_DIR/BH_TMP_DIR/BH_AGENT_WORKSPACE（路径与隔离）、BH_TAB_MARKER（🐴 标记开关）、BH_DEBUG_CLICKS（点击可视化调试）、BH_DOMAIN_SKILLS、BH_RECORD、BH_TELEMETRY、BH_REQUIRE_EXISTING_DAEMON（fail-closed 模式）、BH_OPEN_LIVE_URL。

## 1.7 权限与治理机制
- IPC 安全：POSIX AF_UNIX + umask 077（无 TOCTOU）；Windows TCP loopback + token_hex(32)（handle() token 校验）。
- 进程身份：ping/identify 端到端（防 stale port/port reuse）；identify 严格 int 校验（拒 bool、0<pid<2^31）；restart_daemon 三重验证（identify PID + start-time 指纹 + fingerprint generation）。
- 数据脱敏：日志 _safe_connection_label（只留 endpoint 拓扑）；recorder _URL_SECRETS（code/token/secret→REDACTED）；telemetry FORBIDDEN_KEYS（17 键）+ MAX_TASK_LENGTH 20000；录制 password 字段 mask。
- agent 边界：核心 src/ 受保护；agent 只编辑 agent-workspace/（agent_helpers.py + domain-skills/）；AGENTS.md "prefer the smallest change"、不扩 CDP 面。
- 行为约束（SKILL.md）：不自动 activate_tab（后台操作）；登录墙停下询问；remote 计费停止询问用户；权限弹窗被拒绝不自动重连。

## 1.8 外部依赖
- pip：cdp-use==1.4.5、fetch-use==0.4.0、pillow==12.3.0、websockets==15.0.1；optional mcp==2.1.1。
- 外部工具：profile-use（list/sync cookie，shell out）；Browser Use Cloud API（api.browser-use.com/api/v3，X-Browser-Use-API-Key）；PostHog（eu.i.posthog.com）。

## 1.9 重要设计证据（文档层）
- SKILL.md（14,336B）：agent 工作流规范——tab 纪律（reuse/switch、不重复 new_tab）、后台操作原则、网络空闲等待、焦点模拟（Emulation.setFocusEmulationEnabled finally 关闭）、录制视频。
- AGENTS.md：代码优先级 Clarity/Precision/Low verbosity/Versatility；Skill identity 声明 "browser-use (do not rename)"（与 SKILL.md front matter `name: browser-harness` 不一致 → Candidates）。
- install.md：安装/连接/排障；macOS mac-approve；Snap Linux headless（docs/snap-linux-headless.md）。
- interaction-skills/ 19 篇：dialogs/cookies/profile-sync/iframes/shadow-dom/make-video 等（agent 任务期查阅）。
- skills/：browser-harness 技能注册 + 100+ 站点 domain skills（amazon/xiaohongshu/youtube/gmail 等）。

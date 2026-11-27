# 04 Flow Atlas — browser-harness（七类流）

> 从真实代码导出。关键 Edge 标注 symbol/file/condition/state transition。基准 commit afbcc381。

## 1. Control Flow（控制流）
```
stdin heredoc → run.py main (命令分发)
  → ensure_daemon() [admin.py:525]
      ├─ daemon_alive? → CDP 探测 Target.getTargets 要 "result" → 健康短路
      ├─ stale? → cloud? → stop_remote_daemon [计费权威] / 本地 → restart_daemon
      ├─ Chrome 未开 → _launch_browser → 等 supported_browser_running (≤15s)
      ├─ inspect 未开 → 打开 chrome://inspect 引导 → 抛 remote-debugging-setup
      └─ 弹窗挂起 → 抛 permission-blocked（不重试）
  → exec(code, globals()) [run.py:330] → helper 调用
      → _send [helpers.py:26] → ipc.connect/request → daemon.handle [daemon.py]
          → cdp.send_raw → CDP → Chrome
      ← 结果回传（5s 超时 / 截图 60s → _IPCResponseTimeout）
```
关键分支：`ensure_daemon` 的三层自愈分支（admin.py:525-560）；exec 是 CLI 进程内同步执行（无异步循环）。

## 2. State Flow（状态流）
```
daemon 状态机：_state ∈ {starting, ready, recovering, shutting_down} [daemon.py:16]
  ready ──CDP session 失效──▶ recovering（_begin_recovery 注册）
  recovering ──attach_first_page 成功──▶ ready（_finish_recovery）
  recovering ──drain 超时/失败──▶ _schedule_recovery（_recoveries 集合排队）
  ready ──meta=shutdown──▶ _shutting_down（停止接收新动作）
  _shutting_down ──stop_remote 成功──▶ stop.set() → 退出
  _shutting_down ──stop_remote 失败──▶ 回滚保持存活（下次重试清理）

会话切换：session/target_id（attached）+ dedicated_target_id（named daemon）
  set_session：旧会话 Network.disable ‖ 新会话 4 域 enable（并行，IPC 5s 预算）
  session_replacements：dict ≤32，恢复后清理旧会话
```
关键转换：recovering→ready 以 `_recoveries_idle` 为闸（daemon.py:438）；shutdown 时 `_shutting_down` 先置位（daemon.py:594）。

## 3. Data Flow（数据流）
```
helper args (Python) → JSON (json.dumps) → IPC socket (newline-delimited)
  → daemon.handle 解析 → {method, params} → CDP → Chrome
  ← Chrome 响应 → daemon 组 {"result"/"error"} → 回传 → helper 返回 dict

截图数据：capture_screenshot → CDP Page.captureScreenshot (base64)
  → max_dim 缩放（LLM 视觉上限）→ BH_TMP_DIR 落盘

录制数据：observe → {helper, args(300截), duration, result(500截)} [run.py]
  → recorder events.jsonl（URL 经 _URL_SECRETS 清洗）+ 帧 jpg（ACTIONS 才帧）
```
关键 Edge：`_send` 的 JSON 序列化与超时预算（helpers.py:26-51）；截图不走 events.jsonl（只读 helper 不帧）。

## 4. Evidence Flow（证据流）
```
录制文件夹 <workspace>/recordings/<name>/
  meta.json（name/title/started）← start_recording
  events.jsonl ← observe 每次动作
  0001.jpg… ← 动作后帧（_SETTLE_SECONDS=0.15）
    → video init（read recordings / require_explicit）
    → compile（events→beats：default_action_duration / narration cadence / validate_privacy）
    → review（时长预算 22-32s / reveal text）
    → export（video-template.html + video-source.json 源 manifest + file_hash 校验）
```
关键 Edge：evidence 链的校验点 = `verify_source_manifest`（hash 比对，video.py:160）；隐私校验发生在编译期（`validate_privacy`，拒绝导出而非静默清洗）。

## 5. Authority Flow（权威流）
```
Browser Use Cloud:
  browser-harness auth login → PKCE (S256) + state → 本地 callback server
    → _exchange_authorization_code → AuthRecord (config 目录 0600)
  ──或── device code flow（headless/SSH）→ verification_uri_complete 轮询
  ──或── api_key_stdin_login（printf | auth login --api-key-stdin）
  → auth.get_browser_use_api_key() → _browser_use POST /browsers (X-Browser-Use-API-Key)
  → start_remote_daemon → BU_CDP_WS + BU_BROWSER_ID → daemon

本地 Chrome 授权:
  chrome://inspect/#remote-debugging 勾选 → 用户点 "Allow remote debugging?"
  → macOS: browser-harness mac-approve（须用户授予 Accessibility）
  → daemon 持 CDP 会话（唯一权威）→ agent 经 IPC 请求（无直接权威）

IPC 权威: Windows TCP → token 校验（_server_token）｜POSIX → socket 0600
```
关键 Edge：权威移交点 = ensure_daemon 成功（agent 进程获得 CDP 操作权但非会话所有权）；permission 弹窗被拒 → 权威不转移且不重试（EK-14）。

## 6. Memory Flow（记忆流）
```
短期（进程内）：
  daemon events deque(maxlen=500) ← CDP 事件（session 过滤）
  session_replacements dict ≤32 ← 恢复链
  _recoveries 集合 ← 恢复任务注册

跨进程（文件）：
  BH_HOME/{runtime,tmp,agent-workspace} 私有目录（0700）
  recordings/<name>/（meta+events+帧）——会话记忆的可导出形态
  telemetry.json（install_id + opt-out 偏好）——身份记忆
  auth 文件（config 目录）——凭证记忆
  domain-skills/<site>/——技能记忆（agent 复用）
```
关键 Edge：记忆的遗忘策略 = deque 上限 500 + _session_replacements ≤32 + drain 2s 预算（防无限累积）；录制文件夹是唯一长期记忆载体。

## 7. Policy Flow（治理闭环）
```
Decision: SKILL.md/AGENTS.md 定义行为规范（后台操作/不自动激活/tab 复用/登录墙停下/计费询问）
  → Approval: 人类批准点 = Chrome Allow 弹窗 / mac-approve / cloud 登录
  → Policy: 守护进程强制的不变量
      · permission-blocked 不重连（admin.py:683）
      · shutdown 失败保持存活重试计费清理（daemon.py:594）
      · BH_REQUIRE_EXISTING_DAEMON fail-closed（require_existing_daemon 拒绝 auto-start）
  → Enforcement: 代码路径（ensure_daemon/restart_daemon/handle 异常分支）
  → Future Decision: agent 遇到新站点→写 domain-skills（AGENTS.md 策略）→ 治理知识回流技能目录
```
关键 Edge：治理闭环的证据 = EK-14（纪律）、EK-24（计费权威）、EK-05（tab 纪律）；SKILL.md 是治理文档载体（仓库内 14,336B），非仅 README。

## Flow→KO 交叉校验
| KO | 依赖的 Flow Edge | 一致性 |
|---|---|---|
| KO-01 | Authority Flow（IPC token/PID 验证）| 一致 |
| KO-02 | State Flow（recovering 状态机）| 一致 |
| KO-03 | Authority Flow（permission 不重连）| 一致 |
| KO-04 | Control/Data Flow（exec→daemon→CDP）| 一致 |
| KO-05 | Authority Flow（cloud 全链）| 一致 |
| KO-06 | Data/Evidence Flow（四层脱敏位置）| 一致 |
| KO-07 | Control Flow（后台操作路径）| 一致 |
| KO-08 | Evidence Flow（录制→视频链）| 一致 |
无矛盾（Flow Edge 真实性独立审计见 06 §盲重建-Flow）。

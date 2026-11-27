# Independent Validation Report — browser-harness（ARCH-2026-10-04-001）

> 独立 Auditor 盲重建。**不把考古结果当事实来源**——独立重读仓库建立 Independent Findings 后对比。本报告禁止修改原考古产物（package/ 只读）；修正经阶段⑥ Reconciliation 写入 corpus。

## 审计方法
独立重读（与考古主分析师的盲重建不同的补充阅读面）：helpers.py 1-180 行（_send 超时族/goto_url/click_at_xy/page_info）、daemon.py dialog 相关（grep 定位 628-631/739）、run.py 命令分发（grep traced/StreamTail/dialog 面）、video.py init_recording 全文 + run_cli、mcp_server.py 导入面。攻击焦点：单案例→Pattern、Pattern→L4、项目经验→通用 Principle、ADR→实现事实、Flow Edge 真实性、bypass/override/exception/alternate/direct call/admin path/fallback/legacy path、Epistemic 状态混淆。

## 判定统计
| 判定 | 数量 | 项 |
|---|---|---|
| CONFIRMED | 8 | EK-01 / EK-02 / EK-37 / EK-45 / _IPCResponseTimeout 携带 detail / video require_explicit / mcp 同底座 / MAX_TRACED_STEPS=500 |
| PARTIALLY_CONFIRMED | 2 | EK-04（超时族需细分） / EK-35（export --reviewed 未述） |
| DOWNGRADED | 2 | EK-41（domain-skills 路径） / P-02（三重验证范围） |
| OVER_GENERALIZED | 0 | （CM-01 已在原包 06 自我降级） |
| MISSING | 5 | dialog 状态机 / recordings CLI enable-disable / BH_DEBUG_CLICKS / video export --reviewed / _IPCResponseTimeout 修复事件 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | C-01（SKILL.md name 冲突） |

## 3 项成功（CONFIRMED，代表性）
1. **EK-02 追踪包装**：`MAX_TRACED_STEPS = 500`@run.py:145，`if len(_helper_trace) < _MAX_TRACED_STEPS`@run.py:164——独立复现考古结论。
2. **EK-45 Windows UTF-8 重配置**：run.py:4 注释 "Windows default stdout/stderr encoding is cp1252" + `for _stream in (sys.stdout, sys.stderr): reconfigure`——修复动机独立证实。
3. **EK-04 中间人 + 截图预算**：helpers.py `SCREENSHOT_IPC_RESPONSE_TIMEOUT_SECONDS = 60.0` 且注释 "Cloud screenshots routinely take longer than ordinary CDP round trips. Keep their IPC socket alive within the caller's existing 90-second process budget."——60s 特例预算独立证实。

## 3 项错误（DOWNGRADED / PARTIALLY_CONFIRMED，修正）
1. **E-01（DOWNGRADED）EK-41 domain-skills 查找路径**：goto_url 实际实现 `d = (AGENT_WORKSPACE / "domain-skills" / (urlparse(url).hostname or "").removeprefix("www.").split(".")[0])`——取 **hostname 首段**（youtube.com→youtube；www.example.co.uk→example），非完整 hostname。原 EK-41 表述"按 hostname 找 domain-skills/<site>/"过宽。修正：改为"按 hostname 首标签"；skills/ 目录单名单布局与此一致。
2. **E-02（PARTIALLY_CONFIRMED）EK-04 IPC 超时族需三层细分**：实测 `IPC_CONNECT_TIMEOUT_SECONDS=5.0`（连接）、`DEFAULT_IPC_RESPONSE_TIMEOUT_SECONDS=5.0`（响应）、`SCREENSHOT_IPC_RESPONSE_TIMEOUT_SECONDS=60.0`（截图）。原 EK-04 "5s 超时（截图 60s）"表述正确但粒度粗。修正：明确三层预算。
3. **E-03（DOWNGRADED）P-02 三重验证范围**：pending daemon（handshake-wait，无 IPC socket）路径走 **fingerprint generation 匹配**（_fingerprinted_pending_generation），非 identify PID + start-time 三重验证——三重验证只覆盖"有 IPC 的 ready daemon"路径。P-02 表述需限定范围：ready 路径三重验证，pending 路径 generation 指纹。

## 遗漏（MISSING，5 项，建议入 Reconciliation）
1. **M-01 dialog 状态机**：daemon.py:628-631（Page.javascriptDialogOpening 存 params / javascriptDialogClosed 清 None）、739（meta=pending_dialog 返回）；page_info@helpers.py 在 dialog 打开时返回 {dialog:...} 而非 page info。考古 02 无 EK 覆盖——属于 State/Data Flow 的显著缺口（interaction-skills/dialogs.md 存在佐证其重要性）。→ 新 EK-46。
2. **M-02 recordings CLI enable/disable**：run.py:351 `recordings [--latest|enable|disable]`——auto-recording 偏好的 CLI 入口（recorder.set_auto_recording）。→ EK-34 补充。
3. **M-03 BH_DEBUG_CLICKS**：click_at_xy 在 BH_DEBUG_CLICKS=1 时用 PIL 在截图画点击标记（debug_click_<n>.png，BH_TMP_DIR）。→ 新 EK-47（或并入 EK-02 调试面）。
4. **M-04 video export --reviewed**：video.py run_cli export 子命令带 --reviewed flag（video_render.export(recording, output, reviewed)）——reviewed 态与原始导出的分支未在 EK-35 述。→ EK-35 补充。
5. **M-05 _IPCResponseTimeout 携带 detail**：helpers.py:33-40 "Raising the bare class left str(exc) empty, so every caller that reported the error had [nothing]"——失败修复事件（异常裸类导致 str 为空的修复）。→ EK-04 补充（失败与修复）。

## Flow Edge 真实性攻击
- State Flow "recovering→ready 以 _recoveries_idle 为闸"：daemon.py:438 ✓（_recoveries_idle 存在）。
- Authority Flow "token 校验"：_ipc.py `_server_token` + handle 校验 ✓。
- Evidence Flow "verify_source_manifest hash 校验"：video.py:160 ✓。
- Data Flow "dialog 打开时 page_info 返回 dialog"：page_info@helpers.py 实测 ✓（本项未在 04 Data Flow 列出 → 并入 M-01）。
- bypass 路径检查：http_get（EK-38）直连不经 daemon ✓（已述）；**MCP server 面**：mcp_server.py `from browser_harness.helpers import …` + `ensure_daemon`——MCP 工具仍走 helpers→IPC→daemon，无旁路 ✓（确认 EK-37 语义）。
- fallback 路径：attach_first_page 五层回退 ✓；get_ws_url 多路径 ✓；无未记录的 alternate 路径。

## Benchmark case（供 skill CI 用）
**BK-01 进程身份端到端验证**（browser-harness）：PID 文件/端口/socket 存在 ≠ 身份；信任重建 = ping 协议应答（{pong:true} 严格 dict 检查）+ identify 自报 PID（type is int、0<pid<2^31）+ start-time 指纹 + generation 指纹；SIGTERM 前必须双重确认。与 deepseek-harness daemon 管理的对照价值高——可作为"守护进程生命周期"gold record 候选（corpus benchmarks/）。

## 结论
原考古 45 EK 整体真实可回溯；2 项表述需收紧（E-01/E-03），1 项粒度需细化（E-02）；5 项遗漏建议补入；无 CONTRADICTED、无 OVER_GENERALIZED。Epistemic 状态未发现混淆（C-01 保留 NEEDS_HUMAN_REVIEW）。

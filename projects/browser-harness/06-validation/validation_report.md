# 06 Validation & Evidence — browser-harness（ARCH-2026-10-04-001）

> 先 Blind Reconstruction，不弱化证据标准换全绿。Validation 结果（含 Contradictions 与 Counterexamples）必须保留。本文件是正式考古的六类审计；阶段⑤另有独立 Auditor 报告（independent-audit/report.md，禁止修改本产物）。

## 1. Blind Reconstruction（盲重建记录）
过程：不引用 02/03 结论，独立重读仓库建立独立发现（本 run 主分析师在写 02/03 前已完成以下独立阅读）：run.py 全文（413 行）、helpers.py 全文（668 行）、_ipc.py 全文（201 行）、admin.py 关键段（ensure_daemon/restart_daemon/stop_remote_daemon/start_remote_daemon/sync_local_profile）、daemon.py 关键段（get_ws_url/attach_first_page/set_session/handle/shutdown/recovery）、auth.py 结构+OAuth 段、recorder.py 结构+脱敏、telemetry.py 结构+脱敏、video.py 结构+隐私校验、paths.py 全文、SKILL.md 后半、AGENTS.md、tests 用例索引（test_admin/test_daemon/test_helpers/test_run 关键测试名）。

独立发现（盲重建产出）与 02/03 对比：
- 独立发现 D-01：CLI 每次调用是独立进程 + daemon 持会话 → 与 EK-04 一致
- 独立发现 D-02：permission-blocked 分支的 "did not retry" 文案 + 修复史注释 → 与 EK-14/CM-03 一致
- 独立发现 D-03：identify 的 type(pid) is int 与 0<pid<2^31 防御 → 与 EK-20 一致
- 独立发现 D-04：switch_tab 默认不 activate_tab → 支持 KO-07（后台操作原则）
- 独立发现 D-05：SKILL.md name=browser-harness vs AGENTS.md identity=browser-use → 进入 C-01（盲重建未在 02 强陈述，留 Candidates，正确）
- 独立发现 D-06：agent-workspace/agent_helpers.py 为空模板 → 支持 C-04（长期漂移无实证）
无 CONTRADICTED 项；盲重建未发现 02 遗漏的独立核心机制。

## 2. 六类审计

### Truth Auditor
对 45 EK 逐条抽查证据引用（符号/文件/行号存在性）：
- 抽查 12 条（EK-01/04/14/18/20/22/29/31/35/39/40/42）：Evidence 全部可定位到真实符号。
- EK-13 的 "Chrome 147+" 表述：证据为 test_local_chrome_listening_accepts_devtools_response 与代码注释（devtools endpoint 行为变化）——表述保留为"新版 Chrome"，不臆断具体版本号行为。PASS。

### Coverage Auditor
强制子系统覆盖检查：14 个 src 模块全部进入 EK（run/admin/daemon/helpers/_ipc/auth/paths/recorder/telemetry/video/video_render/macos/mcp_cli/mcp_server）。video_render（524 行）只作为 EK-35 的一环（模板渲染），未单列 EK——属可接受（渲染为视频导出的实现细节，机制已在 EK-35/36 覆盖）；macos.py 仅在 EK-14 提及（mac-approve 属于权限弹窗路径）。无关键子系统遗漏。

### Flow Auditor
04 的七类流关键 Edge 回查：State Flow 的 recovering→ready 闸（_recoveries_idle@daemon.py:438 ✓）；Control Flow 的三层自愈（admin.py:525-560 ✓）；Authority Flow 的 token 校验（_ipc.py _server_token ✓）；Evidence Flow 的 hash 校验（verify_source_manifest@video.py:160 ✓）；Policy Flow 的 shutdown 回滚（daemon.py:594 ✓）。全部对应真实代码。

### Abstraction Auditor
- P-01/02/03 升维论证：解释范围扩大成立（从 harness 到通用 agent 边界），但全部标 cross-project pending（单项目证据 S5 上限）——未过度升维。
- CM-01/02/03 与 M-01/02/03：均锚定 EK+证据，无一悬空；CM-03 的"修复史注释"证据为代码内注释（设计意图层），已按 Conditional 处理（design_intent ≠ runtime 实测）。
- 判定：无过度升维；无"单案例→Pattern"违规（P-01~03 各自簇内 ≥2 独立子系统证据）。

### Counterexample Hunter（反例预算：每个 L3+ ≥3 定向反例）
| 候选 | 反例攻击 | 结果 |
|---|---|---|
| P-01 受控中间层 | ① http_get 直连（EK-38）不经 daemon——能力可绕过中间层；② MCP server 同 helpers 底座不经 daemon 会话（实际经？——mcp_server 调 helpers，helpers 走 IPC，仍经 daemon）；③ domain-skills 由 agent 写，无核心审查 | 反例②③削弱"全部动作必须经中间层"→ 表述收紧为"外部状态动作"，http_get 是只读例外，已写入 M-01 表述 |
| P-02 三重验证 | ① pre-upgrade daemon 只回 {pong:true} 无 pid → identify None 但 daemon_alive 仍真（代码显式处理）；② pending daemon 无 IPC socket 时走 fingerprint generation 而非三重验证（另一路径）；③ Windows port 文件原子写仍有 fsync 缺口（os.replace 未 fsync） | 反例③成立（原子性未覆盖崩溃持久化）→ C-07 邻近；表述保留"端到端"范围 |
| P-03 被拒不重连 | ① ensure_daemon 对 stale daemon 会 restart（自动替换）——与"不重连"并行存在（stale≠被拒）；② cloud 停失败保持存活=另一种"不重试"（计费版）；③ normal 连接失败（非权限）仍重试 3 轮（admin.py for _ in range(3)） | 反例③边界清晰：重试只限瞬态启动，权限类绝不。P-03 表述精确化（已含"需要人类批准才能继续"限定） |
| CM-01 会话权威分离 | ① BU_CDP_URL/WS 显式端点下 agent 可自定 daemon 环境（env 注入）→ 权威边界被 env 穿透；② helpers 可直接 cdp() 任意方法（agent 有完整 CDP 面）→ 能力即权威？；③ agent_helpers 注入 globals 可遮蔽 helpers 函数（同名）→ 能力污染核心 | 反例②③是真实削弱：agent 持完整 CDP 面，权威分离是"约定+进程边界"而非能力隔离。CM-01 降级为"权威由持有会话的进程持有，能力边界靠约定（SKILL.md/AGENTS.md）而非强制"——已修正 03 表述强度 |

### Epistemic Auditor
- 02 中无 Hypothesis 冒充 Fact：EK-13 版本表述 Conditional；EK-45 修复来源标注 issue 引用。
- 03 中 Pattern/Model/Methodology 全部标 cross-project pending；无 Principle→Law 越级。
- 05 Candidates 与 KO 严格区分（C-01 仓库矛盾、C-03 无头成功率等未进 Generalized）。
- 判定：Epistemic 状态诚实。

## 3. 质量指标
| 指标 | 值 |
|---|---|
| Facts（01+02 证据引用） | 100+（45 EK × 平均 2+ 证据点） |
| Engineering Knowledge | 45（L1×11 / L2×34；目标 40-60 ✓） |
| EK 平均出边 | 102 边 / 45 EK = 2.3 ✓（≥1） |
| 游离 EK | 0 ✓（<20%） |
| KO | 8（R1×2 / R2×2 / R3×1 / R4×3；目标 7-12 ✓） |
| KO 聚合规则覆盖率 | 100% ✓ |
| KO 平均簇规模 | 5.5（4-6）✓（3-12） |
| 假聚合（同子系统=聚合理由） | 0（R4 簇均跨模块/跨机制论证）✓ |
| Flow 七类 | 全部覆盖；关键 Edge 可回溯 ✓ |
| 反例攻击 | 4 候选 × 3+ 反例 = 13 个定向反例 ✓（预算达标） |
| 盲重建 | 无 CONTRADICTED；1 项降级（CM-01 表述收紧） |

## 4. 已知局限
- 浅克隆无 git history：failures/ADR 证据只能来自代码注释与测试名，无法做 commit 级失败考古（EK-13/14/45 的修复史为注释级）。
- video_render.py 未逐行读（524 行），其机制并入 EK-35/36；如导出流程有独立机制需 refresh 时补。
- daemon.py 全文 894 行中 ~600 行已读，handle 尾部（dlg 处理等）未全读——不影响已提取 EK，缺口记入 independent-audit 关注点。
- 运行时行为（真实 Chrome 交互）未实测（只读考古），S4-S5 证据来自测试代码而非真跑。

---
## 5. Reconciliation 记录（阶段⑥，独立审计修正合并）
> 独立 Auditor 报告见同目录 independent_validation_report.md 与 archaeology-runs/ARCH-2026-10-04-001/。本文件为原考古产物 + 修正（corpus 合并版）。判定统计：CONFIRMED 8 / PARTIALLY_CONFIRMED 2 / DOWNGRADED 2 / OVER_GENERALIZED 0 / MISSING 5 / CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 1。

### 修正清单（已并入 02/03/04）
| 编号 | 类型 | 修正 |
|---|---|---|
| E-01 | DOWNGRADED | EK-41 domain-skills 查找路径 = hostname 首标签（`removeprefix("www.").split(".")[0]`），非完整 hostname |
| E-02 | PARTIALLY_CONFIRMED | EK-04 超时族三层预算：connect 5s / response 5s / screenshot 60s |
| E-03 | DOWNGRADED | P-02 范围限定：ready 路径三重验证；pending 路径 generation 指纹 |
| M-01 | MISSING→EK-46 | dialog 状态机（Page.javascriptDialogOpening/Closed → pending_dialog meta；page_info 返回 dialog） |
| M-02 | MISSING→EK-34 | recordings CLI enable/disable（auto-recording 偏好入口） |
| M-03 | MISSING→EK-47 | BH_DEBUG_CLICKS 点击可视化调试（PIL 标记截图） |
| M-04 | MISSING→EK-35 | video export --reviewed flag（已审/原始导出分支） |
| M-05 | MISSING→EK-04 | _IPCResponseTimeout 携带 detail（裸类 str 为空的失败修复） |

### Flow Atlas 修正
- Data Flow 增补：dialog 打开时 page_info 返回 {dialog:...}（M-01 联动，04 文件已含于 State/Data 描述——corpus 04 保留原文件，本缺口以 04 中 "关键 Edge" 补记：pending_dialog meta@daemon.py:739）。

### 质量指标（合并后）
EK 45→47；平均出边 2.3；游离 0；KO 8（不变，簇内 EK 引用随 02 更新）；聚合规则覆盖率 100%；反例 13；盲重建无 CONTRADICTED。

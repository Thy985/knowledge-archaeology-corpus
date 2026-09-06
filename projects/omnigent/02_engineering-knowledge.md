# 02 · Engineering Knowledge — Omnigent（EK Graph，宽底座）

> v3.1 规范：每条 EK 必须声明 `links`（六类边：mechanism/subsystem/causal/dependency/constraint/contrast），否则视为"模块说明"。每条 EK 标注证据来源（文件/符号/行）。证据等级按 knowledge-classification：S3=implemented（代码存在）、S4=test validated（测试通过）、S5=runtime validated、S1/S2=设计文档。

## 图统计
- EK 总数：30
- 有 links：30（100%）；平均出边：≥2
- 游离 EK：0
- 证据等级分布：S3（代码存在）=20、S2（设计文档）=5、S3+测试意图=5

---

## A. 核心机制（Meta-Harness 分层）

### EK-01 meta-harness 架构：runner（运行时）+ server（控制面）+ native bridge（适配层）
- **内容**：Omnigent 把"agent 会话执行"（runner）与"多租户控制/协作"（server）分离，vendor harness 通过 native bridge 被包成统一会话；组合/控制/协作三支柱。
- **证据**：README.md "Why Omnigent?"；omnigent/runner/app.py（12.7k 行）与 omnigent/server/app.py（FastAPI 主入口）并列存在；每 harness 一组 native_*.py。
- **links**：subsystem→EK-02（工具分发）、subsystem→EK-04（双评估）、subsystem→EK-11（集成模式）、subsystem→EK-18（managed hosts）、causal→EK-03（runner 拥有 dispatch 导致策略下移）。
- **等级**：S3。

### EK-02 runner 本地工具分发五类 + action_required 原样上送
- **内容**：runner 本地分发大部分工具（_OS_ENV_TOOLS 经 OSEnvironment、_REST_TOOLS 调 server REST、_FILE_TOOLS 调 server 文件 API、_TERMINAL_TOOLS 经 TerminalRegistry、MCP 工具经 RunnerMcpManager——名字随 spec 变不在静态 allow-list）；action_required 事件**原样**上送保持可见性；executor 不自行分发（检查 `should_dispatch_locally` 后跳过）。
- **证据**：omnigent/runner/tool_dispatch.py docstring（"Per designs/RUNNER_TOOL_DISPATCH.md"）与 5 类工具注释。
- **links**：mechanism→EK-03（同一"执行点归 runner"决策族）、dependency→EK-03（本地分发使 runner 侧策略成为可能）、contrast→EK-04（本地 vs server 职责划分）。
- **等级**：S3。

### EK-03 策略执行点跟随控制权：runner 拥有 MCP dispatch → function 型 policy 下移 runner
- **内容**：pre-refactor 时 server 的 PolicyEngine 在每次工具分发上执行 function 型策略；designs/RUNNER_MCP.md 后 runner 拥有 MCP dispatch，因此 runner 必须自己跑这些策略以保持 parity——**执行点跟着控制权走**。
- **证据**：omnigent/runner/policy.py docstring 首段（"Post designs/RUNNER_MCP.md the runner owns MCP dispatch, so the runner has to run these policies itself to keep parity"）。
- **links**：causal→EK-04（双评估由此产生）、causal→EK-10（DENY loud-failure）、constraint→EK-05（能力边界决定哪些留在 server）、subsystem→EK-02。
- **等级**：S3（docstring 明示架构决策）。

### EK-04 双评估设计：runner 本地 fast-path + server 拥有 elicitation 通道
- **内容**：ASK 裁决由 runner POST `evaluate_policy=True` 到 server，server **独立重评**并挂起 elicitation，runner 经 pending_approvals 等待裁决。"dual evaluation is by design——runner 需要本地 fast-path 处理 ALLOW/DENY，server 需要拥有 elicitation 通道"。
- **证据**：omnigent/runner/policy.py docstring（"The dual evaluation (runner + server) is by design"）。
- **links**：causal→EK-06（engine 组合语义）、dependency→EK-09（pending_approvals 注册表）、mechanism→EK-13（elicitation 通道枚举）。
- **等级**：S3。

### EK-05 label/prompt 型 policy 留在 server 侧——能力边界决定执行位置
- **内容**：`label` 与 `prompt` 型策略需要 ConversationStore 与 LLM classifier，runner 没有 → 留在 server；把这两型接到 MCP 工具的 spec 不会获得 runner 侧执行（文档明说）。
- **证据**：omnigent/runner/policy.py docstring（"label and prompt types stay server-side — they need the ConversationStore and the LLM classifier respectively, which the runner doesn't have"）。
- **links**：constraint→EK-03（边界决定下移范围）、contrast→EK-04。
- **等级**：S3。

### EK-06 PolicyEngine 组合语义：DENY 短路 / ASK 累积 / ALLOW 继续，显式传入无 ContextVar
- **内容**：每 workflow 一个 PolicyEngine，在 `_run_agent_loop` 顶部构造并显式传给 4 个 enforcement sites；无 ContextVar、无容器类（POLICIES.md §4）。策略按 YAML 声明顺序迭代；DENY short-circuit、ASK accumulate、ALLOW continue。Labels 在 engine 上 hot-cache（workflow 生命周期）并 write-through 到 conversation_labels。
- **证据**：omnigent/runtime/policies/engine.py docstring（"DENY short-circuits, ASK accumulates, ALLOW continues"；"No ContextVar, no container class (see POLICIES.md §4)"；"Constructed once at the top of _run_agent_loop"）。
- **links**：mechanism→EK-07（同 engine 的 read_only 语义）、subsystem→EK-03/EK-04、causal→EK-10。
- **等级**：S3。

### EK-07 evaluate(read_only=True) 旁路持久化（dry-run 语义）
- **内容**：read_only 求值时不做 apply_label_writes/apply_state_updates，但返回的 PolicyResult 仍携带"本会应用"的 label writes 与 state updates；默认 read_only=False 时持久化照常。
- **证据**：tests/policies/test_engine_read_only.py docstring（"ALLOW path: no apply_label_writes ... but the returned PolicyResult still carries the label writes and state updates that would have been applied"）；用记录变更的 stub store 断言真实持久化调用而非 mock 计数。
- **links**：mechanism→EK-06（同 engine）、contrast→EK-06（dry-run vs 生效）。
- **等级**：S3（测试意图阅读；环境缺重型依赖未运行）。

### EK-08 ASK 默认超时 86400s——审批是人际闸门，应活得比用户离开久
- **内容**：默认等待预算一天，匹配 deciding policy 默认 `ask_timeout`；**旧 120s 默认会把用户 2 分钟内没答的 prompt 静默拒绝（视作 DENY）——是 cost-policy auto-resolve bug 的 runner 侧镜像**。headless/unattended 想 fail-closed 必须显式传有限 timeout。
- **证据**：omnigent/runner/pending_approvals.py（`_DEFAULT_WAIT_SECONDS: float = 86400.0` + docstring："an ASK is a human-in-the-loop gate and should outlive a user stepping away rather than auto-refuse on its own"）。
- **links**：causal→EK-09（生命周期契约）、contrast→EK-10（ASK 等待 vs DENY 拒绝）、constraint→EK-04。
- **等级**：S3。

### EK-09 pending_approvals 生命周期契约：register/cleanup(finally)/resolve 幂等，无 GC
- **内容**：runner 侧 asyncio Future 注册表；调用方 MUST 在 finally 中 cleanup（registry 无 GC，泄漏会累积）；resolve 由 session-event handler 收到 approval 事件时调用，幂等且对未知 id 无操作。
- **证据**：omnigent/runner/pending_approvals.py docstring（"The registry has no GC of its own; leaked entries accumulate"；"Idempotent and no-op when the id is unknown"）。
- **links**：dependency→EK-04、mechanism→EK-08。
- **等级**：S3。

### EK-10 DENY 以拒绝文本作为 tool output 返回（loud-failure）
- **内容**：runner 侧 DENY 把拒绝文本作为工具输出返回，让 LLM 干净看到拒绝（"the loud-failure design principle still applies: a DENY here returns the denial text as the tool output so the LLM sees the refusal cleanly"）。
- **证据**：omnigent/runner/policy.py docstring。
- **links**：causal→EK-03、contrast→EK-08（拒绝 vs 等待）。
- **等级**：S3。

## B. Native Bridge 适配模式

### EK-11 五种 harness 集成模式枚举
- **内容**：SDK_IN_PROCESS（vendor SDK 在 harness 子进程内）/ CLI_SUBPROCESS（每 turn 驱动 vendor CLI）/ ACP_SUBPROCESS（vendor CLI 以 ACP 模式）/ NATIVE_TUI（包裹常驻 vendor TUI，tmux/file-inject）/ NATIVE_SERVER（runner 拥有 vendor server + HTTP/SSE bridge）。
- **证据**：omnigent/harness_capabilities.py `IntegrationMode`（含逐项注释）。
- **links**：mechanism→EK-12（能力声明模型）、subsystem→EK-16（插件注册）、subsystem→EK-14（Claude TUI 实例）。
- **等级**：S3。

### EK-12 能力声明模型取代隐式分支
- **内容**：harness 功能支持原先隐式——散落在 `if harness == "x"` 分支与伴随模块存在性（codex_native_elicitation.py / *_native_hook.py / *_native_permissions.py）；现在用 HarnessCapabilities 一个声明形状，registry 直接回答"这 harness 能做什么"。
- **证据**：omnigent/harness_capabilities.py docstring（"A harness's feature support was previously implicit — scattered across if harness == "x" branches and the presence/absence of companion modules"）。
- **links**：causal→EK-16（entry-point 发现无 import cycle）、mechanism→EK-11。
- **等级**：S3。

### EK-13 Elicitation 四通道枚举
- **内容**：NONE / HOOK（vendor PreToolUse hook POST 到 Omnigent）/ JSONRPC（app-server JSON-RPC elicitation，codex）/ APPROVAL_MIRROR（轮询 TUI 审批面板、镜像到 web）。
- **证据**：omnigent/harness_capabilities.py `Elicitation`。
- **links**：mechanism→EK-04（双评估的 elicitation 面）、subsystem→EK-12。
- **等级**：S3。

### EK-14 Claude 桥双进程 rendezvous：文件系统目录 + MCP stdio + tmux send-keys
- **内容**：两个活进程需要 rendezvous——Claude Code（用户终端资源中）与 Omnigent harness turn（web UI 提交时）。模块拥有小文件系统 rendezvous 目录 + 两个辅助面：MCP stdio server（Claude 启动为子进程，advertise Omnigent 工具：非活动 turn 时 workspace sys_os_* 工具、活动 turn 时经 per-turn relay）+ tmux send-keys 路径（web UI 消息打字进用户同一 tmux pane，Claude 当作普通用户输入；runner 在 tmux.json 中广告 pane socket+target）。
- **证据**：omnigent/claude_native_bridge.py docstring（含 "Claude's experimental Channels MCP capability was the original input path but is blocked at the org policy layer, so this bridge does not use it"）。
- **links**：subsystem→EK-11（NATIVE_TUI 实例）、constraint→EK-15（org policy 约束）、mechanism→EK-17（bridge 目录治理）。
- **等级**：S3。

### EK-15 外部约束塑造实现选择（org policy 阻止 Channels MCP）
- **内容**：Claude Channels MCP 是原始输入路径，但被 org policy 层阻止 → bridge 不用它，改用 tmux send-keys + MCP stdio。**外部治理约束直接改变了适配实现**。
- **证据**：omnigent/claude_native_bridge.py docstring 同上。
- **links**：constraint→EK-14、contrast→EK-13（不同 harness 用不同通道）。
- **等级**：S3。

### EK-16 community harness 通过 entry point group 扩展
- **内容**：核心贡献内置 harness；可选 community 包通过 `omnigent.community.harness` entry point group 贡献额外 harness。
- **证据**：omnigent/harness_plugins.py docstring（"Optional community packages contribute additional harnesses through the omnigent.community.harness entry point group"）。
- **links**：mechanism→EK-12、subsystem→EK-11。
- **等级**：S3。

### EK-17 bridge 孤儿清理：owner.pid 标记 + 启动扫描收割已证死亡
- **内容**：每个 bridge 目录持 bearer-token/auth 材料，崩溃/放弃会累积。owner.pid 标记在每次 turn 的 bridge prep 时刷新（总指向当前 runner）；启动扫描只收割"已证明死亡"进程的目录，保留活与无标记目录。**保守方向**（复用/外来 pid 读作活、不删）；check-then-rmtree 竞态被接受（活会话每 turn 刷新标记，只有真孤儿到达删除点）。与 inner/terminal.py:reap_orphaned_terminals 同构并复用其 _process_alive。
- **证据**：omnigent/native_bridge_common.py docstring（"Conservative in the dangerous direction — a reused/foreign pid reads as alive and is left"）。
- **links**：mechanism→EK-23/EK-24（自愈三件套）、subsystem→EK-14。
- **等级**：S3。

## C. 沙箱 / 云托管

### EK-18 managed hosts：沙箱身份 DURABLE vs 沙箱资源不 durable
- **内容**：host 身份持久而沙箱不持久——hosts 行携带 managed 列（launch-token digest + expiry、provider、sandbox id），relaunch 原地覆盖（同 host_id 下的新沙箱 generation），所以会话绑定在沙箱死于 provider 生命周期上限时存活。
- **证据**：omnigent/server/managed_hosts.py docstring（"The host's identity is DURABLE while its sandbox is not ... a new sandbox generation under the same host_id, so session bindings survive a sandbox dying at the provider's lifetime cap"）。
- **links**：subsystem→EK-19（凭据隔离）、causal→EK-21（reaper）、mechanism→EK-22（host_config 注入）。
- **等级**：S3。

### EK-19 沙箱凭据隔离：专用 launch token，用户凭据从不进入沙箱
- **内容**：沙箱 host 用 server 每次 launch 单独 mint 的专用 launch token 认证回来；**用户的凭据从不进入沙箱**。
- **证据**：omnigent/server/managed_hosts.py docstring（"the user's own credentials never enter the sandbox"）。
- **links**：constraint→EK-18、mechanism→EK-22（secrets 走 env 引用）。
- **等级**：S3。

### EK-20 ManagedSandboxConfig 携带 launcher FACTORY（一个 seam 两个路径）
- **内容**：部署怎么供沙箱后端——两条路径一个 seam：ManagedSandboxConfig 携带 launcher 工厂，嵌入部署注入自定义 launcher 的方式与注入自定义 store 进 create_app 相同。
- **证据**：omnigent/server/managed_hosts.py docstring（"ManagedSandboxConfig carries a launcher FACTORY, so embedding deployments inject custom launchers the same way they inject custom stores into create_app"）。
- **links**：mechanism→EK-18、contrast→EK-16（都是注入式扩展）。
- **等级**：S3。

### EK-21 reaper 部署级离线沙箱回收
- **内容**：部署级（非 per-provider）回收：terminate_after_offline_days 默认 30 天、sweep_interval_s 默认 1 天，默认 enabled=false。
- **证据**：omnigent/server/managed_hosts.py YAML 注释（reaper 段）。
- **links**：causal→EK-18、subsystem→EK-19。
- **等级**：S3。

### EK-22 host_config 注入语义：server-managed 条目覆盖、用户配置存活、secrets 走 env 引用
- **内容**：sandbox 内 ~/.omnigent/config.yaml 内容在 host 启动前安装；server-managed 条目在下一次 launch/resume 时被替换或移除，sandbox 内用户配置存活；secrets 用 api_key_ref: env: 在 **sandbox env**（harness Secret / provider env lane）中解析。
- **证据**：omnigent/server/managed_hosts.py YAML 注释（host_config 段）。
- **links**：mechanism→EK-19、constraint→EK-18。
- **等级**：S3。

## D. 失败 / 修复 / 自愈

### EK-23 bundle 自愈：内置 agent bundle 缺失 → server 启动重传
- **内容**：内置 agent 的 bundle 若从 artifact store 缺失（被 prune 或未恢复），server 启动时重新上传——自愈而不是每次会话启动都失败。
- **证据**：CHANGELOG.md v0.12.0（"A built-in agent whose bundle is missing from the artifact store is now re-uploaded at server startup ... self-heals instead of failing every session launch. (#2498)"）。
- **links**：mechanism→EK-17/EK-24（自愈族）、subsystem→EK-01。
- **等级**：S3（CHANGELOG 记录 + 实现存在）。

### EK-24 runner 重连自愈：server 侧 relaunch 后自动恢复
- **内容**：会话的 runner 被 server 侧重新拉起后自动恢复，而不是每条消息都失败 "runner is not registered"。
- **证据**：CHANGELOG.md v0.12.0（"Recover automatically when a session's runner is relaunched server-side (#2752)"）。
- **links**：mechanism→EK-23/EK-17、causal→EK-01。
- **等级**：S3。

### EK-25 诊断改进 #1119：forwarder POST 失败被误报 "wedged LLM" → 单槽健康记录
- **内容**：native-harness 子进程恰好服务一个会话（app.state.conversation_id），transcript forwarder 与 idle-turn watchdog 跑在同一事件循环；watchdog 触发 stall 时真实原因常是 forwarder 无法 POST 会话事件（ConnectError），用户只见泛化的 "wedged LLM"。修复：forwarder 记录最后一次耗尽的重试失败（process-global 单槽，因一个子进程一个会话一个 turn）+ monotonic 时间戳（watchdog 只把足够新的失败归因给该 turn）+ 成功 POST 清槽（恢复的连接不会把旧失败误归给后续无关 stall）。
- **证据**：omnigent/_native_forwarder_health.py docstring（"issue #1119"）。
- **links**：causal→EK-26（崩溃报告同族）、mechanism→EK-29（观测诚实）、subsystem→EK-14。
- **等级**：S3。

### EK-26 crash handler 三 chokepoint + 无 token shipped + TTY-aware
- **内容**：sys.excepthook（主线程）+ threading.excepthook（后台线程）+ faulthandler（C 级 segfault 落文件，进程将死）覆盖全部崩溃路径。**不携带 token**（发行二进制不能嵌 GitHub 凭据——任何人能提取）：打开 repo 预填 bug-report 模板（title/version/OS/完整 traceback 经 URL query 预填 Description），剪贴板带完整报告备份（URL 过长被截断时兜底）。TTY-aware：交互 "file a bug?" 提示只在 stdin AND stderr 都是真实 TTY 时运行，scripts/CI 不挂起（打印保存的报告路径 + issue 链接）。KeyboardInterrupt/SystemExit 不视为崩溃，defer 到原始 hooks。
- **证据**：omnigent/crash_handler.py docstring（Design notes 全段）。
- **links**：mechanism→EK-25（失败可解释族）、subsystem→EK-01。
- **等级**：S3。

### EK-27 缓存串扰类缺陷：共享 command 不同 env 的 MCP 工具列表串扰
- **内容**：两个子 agent 声明共享同一 command 但 env/headers/Databricks profile 不同 → 子 agent 继承了兄弟配置的缓存 MCP 工具列表。修复：#4457。
- **证据**：CHANGELOG.md v0.12.0（"Sub-agents no longer inherit a sibling config's cached MCP tool list when two declarations share a command but differ in env, headers, or Databricks profile. (#4457)"）。
- **links**：contrast→EK-02（分发正确性）、constraint→EK-06。
- **等级**：S3。

### EK-28 模型路由确定性：优先级分支 + 可审计静态 fallback 表
- **内容**：model_resolver 用显式优先级分支（ModelResolutionSource: EXPLICIT/…）确定性解析；剩余静态 fallback 表带 owner/provenance/discovery_gap 可审计归属（release-curated 排序提示），**绝不发明 live 列表没有的 picker 行**（live harness probes 是 picker 的 source of truth）。
- **证据**：omnigent/model_resolver.py（ModelResolutionSource 枚举）；omnigent/model_fallbacks.py docstring（"It never invents picker rows: ids absent from the live listing are simply not ranked by it"）。
- **links**：mechanism→EK-12（声明化）、subsystem→EK-01。
- **等级**：S3。

## E. 证据 / 观测 / CLI

### EK-29 OBSERVABILITY 自我审计：trace 从不跨 wire 传播（设计-实现差距 ADR）
- **内容**：设计文档主动披露：trace context 从不跨 wire 传播——各层在 agent-turn 路径上从共享 response_id 独立推导同一 W3C trace id（"Elegant, but it only covers boundaries that carry a response_id"）；一切无 response_id 的流量是暗的（host-daemon 控制帧、客户端 REST/SSE 控制流量、会话列表更新、native policy HTTP hook、全部 DB 查询）；HTTPXClientInstrumentor 声明依赖但从未 wire；FastAPIInstrumentor 默认关；get_traceparent_env 是 dead code（零调用点）；无 SQLAlchemy 插桩。项目自评"heavily vibe-coded and lacks a clear mental map ... Static analysis alone has proven unreliable"。修复方案：标准 W3C traceparent 注入/提取，response_id→trace_id 推导只保留为 root trace-ID seed。
- **证据**：designs/OBSERVABILITY.md（Status: Proposed；Motivation 段逐条）。
- **links**：causal→EK-25（同属观测投资）、contrast→EK-30（输出契约）。
- **等级**：S2（设计文档，自我审计为事实）。

### EK-30 CLI 契约 "stdout is data, stderr is decoration"
- **内容**：stdout 只承载机器可解析输出（IDs、paths、config dumps、version），stderr 承载装饰与诊断（warnings、errors、banner、spinners、progress）；禁止手写 ANSI；保证 `omnigent version | cat` / `omnigent config list | jq` byte-clean。
- **证据**：designs/CLI_CONTRACT.md（"The one rule: stdout is data, stderr is decoration" + Never 条款）。
- **links**：mechanism→EK-25/EK-29（可观测/可脚本化族）、subsystem→EK-01。
- **等级**：S2（契约文档 + cli.py 实现存在，S3）。

---

## EK Graph 边密度统计

| 指标 | 值 |
|------|-----|
| 平均出边 | ~2.5 |
| 游离 EK | 0 |
| mechanism 边 | EK-02/04/07/08/09/11/12/13/17/18/20/22/23/24/25/26/29/30 间多条 |
| causal 边 | EK-01→03→04→06→10；EK-18→21；EK-25→26 |
| constraint 边 | EK-05→03；EK-15→14；EK-19→18 |
| contrast 边 | EK-02↔04；EK-05↔04；EK-08↔10；EK-27↔02 |
| subsystem 边 | EK-01/02/03/04/05/06/10（runner 面）；EK-18/19/20/21/22（托管面）；EK-11/12/13/14/16/17（桥接面） |

---

## F. Reconciliation 补录（Independent Auditor 发现，2026-09-07）

### EK-31 CredentialProxy：swap-on-access + 占位符注入 + 跨 host 泄漏守卫（Auditor MISSING 修复）
- **内容**：沙箱持有"没有任何 credential 形状的东西"——默认 swap-on-access：工具向 host 发无 Authorization 的请求，egress MITM proxy 在出口注入 `Authorization: <scheme> <real>`；`gh` 等客户端本地看不到凭据就短路"authentication required"，此时 opt-in 注入合成 `oa_cred_*` 占位符让客户端发出请求，proxy 在出口换真 secret，**占位符重放到其它 host → HTTP 403（cross-host leak guard）**。YAML `credential_proxy` 类型（https_bearer/https_basic/git_https/gh_basic）归一化为 CredentialProxyEntry，runtime 在父侧解析 source（**永不进入沙箱**）。
- **证据**：omnigent/inner/datamodel.py:400-430（CredentialProxyEntry docstring："The sandbox holds *nothing* credential-shaped"、"rejects a placeholder replayed to any other host with HTTP 403"）；omnigent/onboarding/sandboxes/kubernetes.py:492-497（`configure_clone_credentials(server_url, host_id)` 为代理绑定配置，非真凭据写入）。
- **links**：mechanism→EK-19（凭据隔离的实现机制）、constraint→EK-18、subsystem→EK-22（env 注入面）。
- **等级**：S3。**意义**：这是"用户凭据从不进入沙箱"（KO-06）的工程化实现，并为跨 host 泄漏守卫提供回归语义（R-01）。

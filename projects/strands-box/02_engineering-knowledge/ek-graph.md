# 02 Engineering Knowledge — strands-box（ARCH-2026-10-23-001）

> EK Graph：每条 EK 带证据（文件:行/锚）+ links（6 类边）。前缀 B（Box），与 Aegis G / Archon B 区分。

## EK-B01 · 三个 enforcement points = 一个 authority
- **机制**: Box 构建三适配器共享一个 `Arc<PolicyEngine>`：EgressPolicyInterceptor（net:*）/ ShellPolicyInterceptor（shell:exec/spawn + fs:*）/ ScriptPolicyInterceptor（Python fs:*）——"Three views of one authority, never three authorities"；命令在解析后判定一次（shell:exec vs shell:spawn 二选一）
- **证据**: crates/box/AGENTS.md（"The authored policy is the only authorizer, at all three enforcement points"）；crates/policy/src/adapters/{egress,shell,script}.rs
- **links**: mechanism↔B06（Request 统一事件）；dependency→B02（fail-closed 语义）

## EK-B02 · fail-closed 四拒绝语义
- **机制**: DenyReason = Forbidden（显式 forbid 命中）/ NoMatch（无 permit 匹配）/ **PolicyPending（完整策略包未安装→拒绝）**/ InternalFault（构造或评估失败）；README "denied by default: a matching permit must allow, a matching forbid overrides"
- **证据**: crates/policy/src/decision.rs:48-80；README Overview；adapters/egress.rs:182（PolicyPending 拒绝路径）
- **links**: causal→B10（决策→回执）；constraint→B03（pin 是契约一部分）

## EK-B03 · 引擎精确 pin（安全 pin 契约）
- **机制**: cedar-policy `=4.11.0`（"a *security* pin: schema-directed context parsing differs across 4.x minors, so the pin is part of the fail-closed contract"）；dogwood-language `=1.0.0`（"The policy engine. The ONLY decision engine."）；dogwood-local-engine `=1.0.0`（temporal）；Cargo.toml 注释原文
- **证据**: crates/policy/Cargo.toml [dependencies] 注释
- **links**: constraint→全部 policy 面（引擎不可换）；contrast↔B16（治理机制同域）

## EK-B04 · SELF_DEFENDED_FILES + refuse_removed_keys
- **机制**: `SELF_DEFENDED_FILES = ["box.toml", "policy.dw"]`（策略自身文件受保护）；`refuse_removed_keys` 在真实解析前运行——文件带已移除旧 key 时按名拒绝（"rather than an unknown-field error"），`name_is_required_and_every_removed_key_is_refused_by_name` 测试钉死
- **证据**: crates/policy/src/policy.rs:20；crates/box/AGENTS.md（"A key that moved is refused by name"）
- **links**: constraint→B01（配置是策略面）；mechanism↔B18（测试钉死行为）

## EK-B05 · ApprovedPath 类型化路径（raw spelling 编译不过）
- **机制**: Request::Fs 的 path 是 `ApprovedPath`（"The type carries the claim"），无公开构造器——"a raw spelling now fails to compile instead of passing a check that was never run"（曾有 caller 未 canonicalize 的缺陷被类型消灭）；resource() 是标签非权威，"Nothing may resolve it back into a path and act on it"
- **证据**: crates/policy/src/request.rs:130-155 注释 + 190-210
- **links**: contrast↔B06（标签 vs 权威）；constraint→B02

## EK-B06 · Request 统一事件语义 + 跨工具归一
- **机制**: PEP 递统一 Request enum；Shell 读与 Python 读同报 fs:read、curl 与 Python HTTP 同报 http:request；`resource()` 返回 reported 拼写（"a record from one machine reads the same as the rule that decided it"）——支持跨工具规则（读了 customer-data 后禁出站）
- **证据**: crates/policy/src/request.rs:142-260（resource() 实现）；docs/design/mcp.md（MCP 同门）
- **links**: mechanism↔B07（MCP 双门）；causal→B08（temporal 依赖历史）

## EK-B07 · MCP 双门决策（gate 2 非默认拒绝）
- **机制**: gate 1 = 粗粒度 mcp:call（per server，permit 覆盖该 server 所有 method/tool）；gate 2 = refine_tool_call 生成的 per-tool action 精化——"Refinement is not default-deny: if the per-tool rule does not match, gate 1's allow stands"；粗授权宽泛是设计使然，精确 operator 对危险工具写 forbid
- **证据**: docs/design/decisions.md#mcp-authorization-is-two-gates（line 816-825）；crates/policy/src/policy.rs:306（refine_tool_call）
- **links**: contrast↔B02（粗 gate 非默认拒绝 vs 整体 fail-closed——范围边界）；subsystem↔B06

## EK-B08 · temporal 策略（durable history + monitor 恢复）
- **机制**: dogwood-local-engine =1.0.0（"Its durable assembly appends each event before evaluation and recovers monitor state from the box's private history file"）——temporal 算子（since/formerly/once/count）依赖记录历史；decide 调用时先记录后评估
- **证据**: crates/policy/Cargo.toml 注释；crates/policy/tests/fail_closed_guards.rs 头部（"the enforced temporal surface (decisions.md#temporal-rules-are-enforced-against-recorded-history)"）
- **links**: dependency→B06（统一事件是历史语义前提）；causal→B09（时间性拒绝可解释）

## EK-B09 · 拒绝回执 403 + rule id/description（agent 可自纠）
- **机制**: PolicyAttribution 携带 authored @id + @description；拒绝以可读形式回执（decision.to_string() 走 Err 路径），agent 能读"哪条规则为何拒绝"并自纠（等待而非盲重试）
- **证据**: crates/policy/src/decision.rs:86-110（PolicyAttribution）；adapters/egress.rs:174-180（Err(decision.to_string())）
- **links**: causal→B10（Vault 注入在允许后）；contrast↔aegis justification_gap（解释性回执两种形态）

## EK-B10 · 凭证外挂网关（agent never holds the secret）
- **机制**: credentials Vault 在出站边界解析真实密钥并 attach（5 种注入：Bearer/Custom header/Basic/Query param/AWS SigV4，sign_request 在 source 层）；workload 只见 placeholder；"resolves each real secret and attaches it to an outbound request at the network boundary, so the workload never holds it"
- **证据**: crates/credentials/Cargo.toml description + src/{vault,attach,sources}.rs；docs/design/credentials.md；tenets #2
- **links**: mechanism↔B11（box 是凭证边界）；causal→B12（FORBIDDEN_PATHS 兜底）

## EK-B11 · box 是凭证边界 + 无 per-process selection
- **机制**: agent 拿到每个声明工具/每个 MCP alias/每条 egress route；**无 per-process selection**——"the gateway and the broker cannot tell which process in a box calls, so a selection would bound nothing. A process that must not get a credential goes in another box"
- **证据**: crates/box/AGENTS.md（"The box is the credential boundary. No process table selects what it reaches."）
- **links**: constraint→B10；contrast↔aegis（进程内 credentials vs 边界外挂）

## EK-B12 · FORBIDDEN_PATHS + validate_grant（凭据路径永不授予）
- **机制**: containment 持有 FORBIDDEN_PATHS（~/.aws、~/.ssh、keychain、browser profile 等）；每项 grant 经 validate_grant 判定；"Two lists in two crates was the defect"（box 曾只筛内置路径，record [agent] 条目绕过凭据检查——已修）；store 前缀/父级仍拒绝，仅 operator 精确命名的 store 单点放行
- **证据**: crates/box/AGENTS.md（FORBIDDEN_PATHS 段 + a_credential_store_is_granted_only_when_named_exactly 测试）
- **links**: causal→B10；constraint→B14（containment 执行面）

## EK-B13 · egress capability fold + MutationCollision fail-closed
- **机制**: 每能力出 CapabilityOutcome（Applied{edits}/Unavailable{fault}）；decide_request 折叠：机制失败（first_fault）→ Deny{Credential}（"a mechanism failure, not a policy verdict: the boundary could not perform the edit, so nothing egresses"）；finish_allow 检测跨能力 mutation 冲突 → Deny{MutationCollision}
- **证据**: crates/egress-gateway/src/capability/decider.rs:1-60
- **links**: mechanism↔B02（fail-closed 族）；dependency→B10

## EK-B14 · containment 双后端 + authority 降权
- **机制**: macOS Seatbelt（seatbelt.rs 2750 行：macho.rs 解析二进制）+ Linux namespace（authority.rs：close_inherited_descriptors/verify_closed/drop_all_capabilities/set_no_new_privileges）；contain_trampoline bin 衔接
- **证据**: crates/containment/src/backend/{macos/seatbelt.rs,linux/namespace/*};tests/backend_seatbelt.rs 等 12 测试
- **links**: dependency→B12（FORBIDDEN_PATHS 执行于 grant 之下）；contrast↔B01（OS 层 vs 策略层双层）

## EK-B15 · ShellSpawn 边界（host binary 无 fs:* 决策）
- **机制**: ShellSpawn（host binary）"Permitting this grants more than it appears to. The binary runs against the kernel rather than the Shell's VFS, so no fs:* decision fires…its children inherit containment while raising no decision of their own"；program_path 按 identity 判定（"never against this value and never against a re-resolution through PATH"）
- **证据**: crates/policy/src/request.rs:200-240（ShellSpawn 注释）
- **links**: constraint→B01（spawn 需特别谨慎）；contrast↔B06（exec 有 fs:* 决策 vs spawn 无）

## EK-B16 · 治理即接口冻结（interface-frozen + ADR 降格）
- **机制**: AGENTS.md STOP 段：接口变更 + 基础前提变更需用户显式批准（agent 不得自批）；feature-critic 视 BLOCKER 级；decisions.md 自称 historical record——"The code is the only authority. Where this page and the code disagree, the code is right and the entry is stale"
- **证据**: AGENTS.md STOP 段；docs/design/decisions.md 头部（145 anchors）
- **links**: mechanism↔B18（测试钉死）；contrast↔aegis AGEF/AGP 自述标准（文档权威性两种态度）

## EK-B17 · DecisionObserver 非阻塞契约
- **机制**: observed 在 decide 内运行——"must not block, await, or perform input or output. decide is infallible and free of both"；observer 收到 action/resource/decision 而非 Request（无法重读被判定对象）与 principal（"passing it would invite an observer to record a value that did not decide anything"）
- **证据**: crates/policy/src/observe.rs:1-60
- **links**: dependency→B01（三拦截点都经 observer）；constraint→B13（egress 不受观察影响）

## EK-B18 · reload live authority（热重载 + 调用方认证）
- **机制**: PolicyEngine.reload：authority 内建锁串行化与在飞决策；调用方拥有"谁可触发"问题——"Nothing here authenticates a request: the contained agent must never reach a reload, and the only thing that keeps it out is where the trigger lives"
- **证据**: crates/policy/src/policy.rs:228-245（reload + 注释）；HEAD 8c46dac（#122）
- **links**: causal→B02（reload 后语义保持 fail-closed）；subsystem↔B08（历史状态一致性）

## EK-B19 · 黑盒测试套件 + fail-open 回归钉死
- **机制**: test-workload 20 套件驱动 **built** binary（黑盒确定性）；test-integ 显式 exclude（"excluded deliberately, not overlooked: it drives the BUILT strands-box binary as a subprocess"）；fail_closed_guards.rs：加载必须失败——"a guard that lets one of these through has reopened the fail-open class"；verdict-hermetic CI
- **证据**: Cargo.toml exclude 注释；crates/policy/tests/fail_closed_guards.rs:1-30；.github/workflows/
- **links**: mechanism↔B18（行为回归钉死）；constraint→B16（冻结面由测试背书）

## EK-B20 · per-request Shell + 内嵌 Lua 5.4 真实依赖面
- **机制**: "A Shell is cheap, which is what makes per-request affordable. ~110-350 microseconds…632 bytes over an Arc<Mutex<Vfs>>"；但内嵌 Lua 5.4（mlua lua54 async vendored 非可选）——"Never describe the Shell as a small unsafe-free parser…the interpreter is in the dependency graph"
- **证据**: crates/box/AGENTS.md（Shell cheap 段 + Five residuals 段）；crates/box/Cargo.toml
- **links**: constraint→B15（spawn 语义）；contrast↔aegis（插桩 vs 解释器沙箱）

## EK Graph 质量门
- 20 条 EK；links 覆盖率 100%；平均出边 ~1.7；游离 0%；全部可回溯仓库实盘（HEAD 8c46dac）

## EK-B21 · telemetry DecisionRecord 结构 [reconciled via R-4]
- **机制**: DecisionRecord（record.rs:338）+ DecisionCause（record.rs:132）；Collector 一 lane/box，决策记录含 action/resource/decision（DecisionObserver 契约喂入）；写入 private/telemetry/records.jsonl（"The box writes no diagnostic log file"——唯一正式记录流）
- **证据**: crates/telemetry/src/record.rs:132,338；crates/box/AGENTS.md（log file 禁则）
- **links**: mechanism↔EK-B17（observer 非阻塞契约）；dependency→EK-B01（三拦截点决策都入记录）

# Engineering Knowledge Layer（工程知识层 · 宽底座）

> **v3 核心交付物。** 本层保存 Codex 具体如何解决工程问题的无损底座——核心机制、关键实现、决策、失败、配置、边界。
> **"不够抽象"绝不等于"不重要"。** 这些是未来 Agent 推理的原材料：抽象错了可以回到底层重新推导。
> 每条遵循 knowledge-schema v3：`knowledge_layer: engineering`，锚定真实符号 + 说明为什么 + 标注边界。
> 编号 EK-01 ~ EK-52，Generalized KO 的 `derivation.facts` 指回本层编号。

---

## 一、核心机制（Core Mechanisms）——关键服务如何工作

### EK-01：ExecApprovalRequirement 三态审批枚举

- **claim**：每个工具调用的审批需求是三态而非二元——Skip（直接执行）/ NeedsApproval（需审批）/ Forbidden（禁止），且前两态携带策略修正案建议。
- **source**：`codex-rs/core/src/tools/sandboxing.rs:152-171`
- **detail**：`Skip { bypass_sandbox, proposed_execpolicy_amendment }`、`NeedsApproval { reason, proposed_execpolicy_amendment }`、`Forbidden { reason }`。`proposed_execpolicy_amendment()` 方法（:174-186）从 Skip 和 NeedsApproval 提取修正案，Forbidden 不产生。
- **why**：审批决策本身就是策略学习的输入——系统在每次审批时同时计算"是否应该把这个命令固化为免审规则"，形成"执行→审批→固化→未来免审"闭环。Forbidden 不携带修正案是安全不变量：拒绝的命令永远不会被自动固化。
- **scope**：适用于所有工具运行时（shell / apply_patch / unified_exec）。不适用于无需审批的 Full Access 模式。

### EK-02：ExecPolicyManager 策略持久化与热更新闭环

- **claim**：用户审批"允许这个命令"后，系统通过 Semaphore 串行 + spawn_blocking 落盘 + 幂等检查 + ArcSwap 原子替换，将命令前缀固化为免审规则，跨 session 持久化且不重启即生效。
- **source**：`codex-rs/core/src/exec_policy.rs:276-279, 464-512`
- **detail**：`append_amendment_and_update()` 执行：① `update_lock.acquire()`（Semaphore=1 串行化）② `spawn_blocking(blocking_append_allow_prefix_rule(default.rules))` 落盘 ③ 幂等检查（当前策略已 Allow 则跳过）④ `clone → add_prefix_rule → policy.store(Arc::new(updated_policy))` 原子热更新。
- **why**：读路径高频（每次工具调用查策略）需无锁；写路径低频（用户"允许并记住"才触发）需串行防并发 append 丢规则。ArcSwap 保证读线程要么看旧策略要么看新策略，不会看到中间态。
- **scope**：命令前缀规则和网络域名规则共享此机制（`append_network_rule_and_update` :514-558 同构）。crash 窗口内落盘成功但 store 前崩溃→重启时从磁盘全量加载保证最终一致（非强一致）。

### EK-03：Guardian 自动审批审查器——独立 session + fail-closed + 熔断器

- **claim**：用另一个 AI 审批 AI 的命令时，审批器运行在独立 session（approval_policy=never、不继承父 exec-policy、只读沙箱），超时/出错默认拒绝，连续拒绝触发熔断中断 turn。
- **source**：`codex-rs/core/src/guardian/mod.rs:62-77, 169-246`，`guardian/review.rs:321-343, 655-689, 977-990`
- **detail**：路由条件 `routes_approval_policy_to_guardian()` = approval_policy ∈ {OnRequest, Granular} ∧ approvals_reviewer == AutoReview。Full Access 短路直接 Approved。独立 review session 可复用（trunk idle 追加同一 conversation 保 cache key；trunk busy fork ephemeral session）。90s 超时，最多 3 次重试（仅 transient 错误）。
- **why**：隔离防止 Guardian 被父 session 的免审规则或恶意 prompt 影响；fail-closed 因为审批器不可用时"不执行"比"可能执行危险命令"安全；熔断防止 Guardian 持续拒绝导致 agent 反复尝试消耗 token。
- **scope**：仅 OnRequest/Granular + AutoReview 配置激活。Never 直接执行，UnlessTrusted 必须人工审批，均不走 Guardian。

### EK-04：网络审批闭环——代理层拦截 + 运行时回调 + 策略固化

- **claim**：命令执行时的网络请求经过 codex-network-proxy，allowlist 命中放行，miss 时通过 NetworkPolicyDecider trait 回调 core 动态决策（Allow/Deny/Ask），审批通过的域名固化为网络规则。
- **source**：`codex-rs/core/src/tools/network_approval.rs:1014-1030, 1032-1158`，`network_policy_decision.rs:74-102`
- **detail**：`build_network_policy_decider()` 返回 `Arc<dyn NetworkPolicyDecider>`，在 allowlist-miss 时调用。`begin_network_approval()` 在工具执行前构建 execution-scoped proxy，注入 fallback_policy_decider。`ActiveNetworkApproval { registration_id, cancellation_token, execution_proxy }` 跟踪活跃审批。被拒绝请求通过 `build_blocked_request_observer`（:1002-1012）记录为 sandbox violation + blocked request。
- **why**：网络访问无法在命令解析阶段静态判定（命令运行时才发起请求），必须在代理层运行时拦截。回调到 core 可利用 session 上下文做动态决策而非静态 allowlist。
- **scope**：仅 managed_network 激活时生效。spec 为 None 或 managed_network_active=false 时无网络治理。代理生命周期与 session 绑定，session 销毁时 execution_proxy 随之取消。

### EK-05：RolloutBudget 跨 agent 树共享 token 预算

- **claim**：root agent 创建时初始化 RolloutBudget，通过 Arc 共享给整个 agent 树（root + 所有 subagent），按加权模型计费 token 消耗，预算耗尽后 latch，阈值穿越时每线程去重提醒。
- **source**：`codex-rs/core/src/rollout_budget.rs:11-127`，`agent/control.rs:131, 148-167`
- **detail**：`RolloutBudget { state: OnceLock<Mutex<RolloutBudgetState>> }`，state 含 `config / weighted_tokens_used / deliveries: HashMap<ThreadId, ThreadBudgetDelivery>`。`record_usage()` 优先用服务端 `codex_rollout_budget_units` 精确计费，fallback 到本地加权（output × sampling_weight + non_cached_input × prefill_weight）。`pending_reminder()` 数已穿越阈值数作 reminder_index，每线程 delivery 去重。`rearm_reminder()` 清除记录强制重申。
- **why**：多 agent 树的 token 消耗需统一预算防止 subagent 失控消耗。加权计费因为 output 比 input 贵。每线程去重防止同一阈值在多线程间重复提醒。
- **scope**：共享配额是设计意图——subagent 确实可以消耗 root 预算（非漏洞）。如需隔离应使用 per-agent 配额（Codex 未提供）。

### EK-06：PermissionProfileState——Session 级权限 SSOT

- **claim**：Session 级权限由 PermissionProfileState 单一事实源管理，将 constrained profile、active profile id、profile-defined workspace roots 三字段封装，通过方法更新保证一致性，切换时保留 deny-read 限制。
- **source**：`codex-rs/core/src/session/session.rs:96-99, 164-201, 360-502, 504-538`
- **detail**：注释明确要求"constrained profile, active profile id, and profile-defined workspace roots in sync by using the methods below instead of mutating the fields independently"。`permission_profile()` 从 state 快照。`effective_permission_profile()` 优先用选中 environment 的 config，fallback 到 materialized。`apply()` 更新时通过 `set_permission_profile_projection()` 保持三字段同步，并调用 `preserve_deny_read_restrictions_from()` 保留 deny-read。
- **why**：三个字段强相关（active id 决定用哪个 profile，profile 决定 workspace roots），独立修改会导致不一致。deny-read 保留防止 profile 切换时意外放宽读取限制。
- **scope**：session 级权限。environment 级 config 可覆盖 session 级 materialized profile。

### EK-07：AgentControl——多 agent 控制面单例 + 共享状态

- **claim**：AgentControl 在 root thread/session tree 中只创建一次，clone 给所有 subagent，持有 Weak<ThreadManagerState>（断环）、AgentExecutionLimiter（并发限制）、Arc<RolloutBudget>（共享预算）、Arc<ArcSwapOption<String>>（共享 service tier）。
- **source**：`codex-rs/core/src/agent/control.rs:111-134, 146-173`
- **detail**：`manager: Weak<ThreadManagerState>` 避免 `ThreadManagerState → CodexThread → Session → SessionServices → ThreadManagerState` 循环。`agent_execution_limiter: Arc<AgentExecutionLimiter>` 在 `with_session_id` 中用 `effective_agent_max_threads` 初始化。`rollout_budget: Arc<RolloutBudget>` 和 `root_service_tier: Arc<ArcSwapOption<String>>` 跨 agent 树共享。
- **why**：多 agent 协作需要全局协调（并发限制、资源预算、service tier），但全局状态与 session 存在循环引用风险，用 Weak 断环。单例 clone 保证 registry scoped to root thread。
- **scope**：root thread 级控制面。subagent 获得 clone 但不拥有 manager 的 Arc（仅 Weak）。

### EK-08：上下文片段系统——50+ 结构化 ContextualUserFragment

- **claim**：所有注入模型上下文的信息必须定义为 `core/context/` 下的 struct 并实现 `ContextualUserFragment` trait，当前有 50+ 个 fragment（guardian_policy、permissions_instructions、rollout_budget、token_budget_context 等），每个是结构化上下文注入单元。
- **source**：`AGENTS.md:91-100`，`codex-rs/core/src/context/` 目录
- **detail**：AGENTS.md 第 6 条强制要求 trait 实现（编译期强制）。fragment 涵盖权限指令、Guardian 策略、预算提醒、world_state 渲染等。每个 fragment 有界大小和硬 cap。
- **why**：类型安全的上下文治理——trait 确保所有注入项有结构化接口（渲染/大小/优先级），防止任意字符串注入导致上下文不可控。50+ fragment 说明上下文是 Codex 的核心治理面。
- **scope**：主会话上下文。Guardian review session 有独立的 5 层预算（见 EK-22），不直接复用主 context fragment。

---

## 二、关键实现（Key Implementations）——具体实现 pattern

### EK-09：ArcSwap 读无锁 + Semaphore 写串行的并发模型

- **claim**：读多写少的策略状态用 `ArcSwap<Policy>` 无锁读取（`load_full()`），写用 `Semaphore::new(1)` 串行化，写时 clone→modify→`store(Arc::new())` 原子替换。
- **source**：`codex-rs/core/src/exec_policy.rs:276-279, 298, 311-313, 510`
- **detail**：`ExecPolicyManager { policy: ArcSwap<Policy>, update_lock: Semaphore }`。读路径 `current()` = `policy.load_full()` 无锁。写路径先 `update_lock.acquire()`，再 `let mut updated = current.as_ref().clone(); updated.add_prefix_rule(...); self.policy.store(Arc::new(updated));`。
- **why**：ArcSwap 是 Rust 读多写少场景的成熟模式（arc-swap crate 推荐）。Semaphore(1) 防止并发写导致规则丢失或乱序。clone→modify→store 保证读线程不看到中间态。
- **scope**：适用于读多写少、需热更新的配置/策略系统。写操作是低频的（用户审批触发），Semaphore 等待时间 = 文件 append 毫秒级，非瓶颈。

### EK-10：Weak<Session> 代理回调——避免循环引用 + 生命周期感知

- **claim**：网络代理回调通过 `Arc<RwLock<std::sync::Weak<Session>>>` 持有 session 引用，避免 Session↔NetworkApprovalService 循环引用；调用时 `upgrade()`，失败则保守降级。
- **source**：`codex-rs/core/src/tools/network_approval.rs:1014-1030`
- **detail**：`build_network_policy_decider(network_approval: Arc<NetworkApprovalService>, network_policy_decider_session: Arc<RwLock<Weak<Session>>>)`。回调闭包中 `let Some(session) = session.read().await.upgrade() else { return NetworkDecision::ask("not_allowed"); };`。
- **why**：Session 持有 NetworkApprovalService（通过 SessionServices），NetworkApprovalService 的回调又需要引用 Session——直接用 Arc 会循环引用导致内存泄漏。Weak 不增加引用计数，session 释放后 Weak 自动失效。
- **scope**：Rust 特定实现。通用原则是"回调持有方可能已释放，需保守降级"，适用于任何有生命周期的回调系统（API Gateway ext_authz、sidecar 代理等）。

### EK-11：Weak<ThreadManagerState> 控制面断环

- **claim**：AgentControl 通过 `Weak<ThreadManagerState>` 持有全局线程管理器，避免 `ThreadManagerState → CodexThread → Session → SessionServices → ThreadManagerState` 的引用循环。
- **source**：`codex-rs/core/src/agent/control.rs:121-124`
- **detail**：注释明确说明循环路径。`upgrade()` 失败时返回错误（不 panic）。AgentControl 被 clone 给所有 subagent，但每个 clone 都持有同一个 Weak。
- **why**：多 agent 控制面需要回调全局状态（spawn 新 thread、查询 registry），但全局状态拥有整个 thread/session 树，直接 Arc 引用会循环。Weak 是 Rust 父子双向引用的标准断环模式。
- **scope**：Rust 语言级模式。适用于任何有父子双向引用的系统（DOM 事件委托、组件树等）。

### EK-12：spawn_blocking 落盘——避免阻塞 async runtime

- **claim**：策略文件写入（blocking IO）在 `spawn_blocking` 中执行，不阻塞 async runtime 线程池。
- **source**：`codex-rs/core/src/exec_policy.rs:464-512`
- **detail**：`append_amendment_and_update` 中 `spawn_blocking({ let prefix = amendment.command.clone(); move || blocking_append_allow_prefix_rule(&policy_path, &prefix) }).await...`。网络规则同理（:514-558）。
- **why**：Codex 核心是 async 系统（tokio runtime），文件 IO 是阻塞操作。若在 async 线程中直接写文件，会占用 runtime 线程导致其他并发任务（如其他工具执行、网络请求）饥饿。spawn_blocking 将阻塞操作移到专用线程池。
- **scope**：所有文件持久化操作。策略文件路径 = `codex_home/rules/default.rules`（:867-869，常量 `RULES_DIR_NAME="rules"`、`DEFAULT_POLICY_FILE="default.rules"` :54-56）。

### EK-13：OnceLock<Mutex> 延迟初始化

- **claim**：RolloutBudget 使用 `OnceLock<Mutex<RolloutBudgetState>>` 延迟初始化——未配置时所有方法 no-op，配置后一次性初始化。
- **source**：`codex-rs/core/src/rollout_budget.rs:18-20, 33-44`
- **detail**：`configure()` 在 `AgentControl::new` 中调用（agent/control.rs:163-165）。未配置时 `lock()` 返回 None，`record_usage()` / `pending_reminder()` 均返回默认值（false / None）。OnceLock 保证只初始化一次，Mutex 保护内部状态。
- **why**：RolloutBudget 是可选功能（rollout 配置可能不存在），延迟初始化避免在无配置时分配状态或报错。OnceLock 比 `Option<Mutex<...>>` 更安全（一旦初始化不可变回 None）。
- **scope**：可选共享资源的标准模式。适用于任何"配置可能不存在但存在时需共享"的场景。

### EK-14：trunk session 复用 + ephemeral fork 并行审批

- **claim**：Guardian review session 可复用——cached trunk session idle 时后续审批追加到同一 conversation（preserve prompt-cache key），trunk busy 时从 last committed rollout fork ephemeral session 实现并行审批不互相阻塞。
- **source**：`codex-rs/core/src/guardian/review.rs:977-990`（doc comment）
- **detail**：doc comment 说明："cached trunk session idle 时后续审批追加到同一 conversation（preserve prompt-cache key）；trunk busy 时 fork ephemeral session"。Guardian review session 运行在 approval_policy=never + 无继承 exec-policy + 只读沙箱。
- **why**：复用 trunk session 保留 prompt cache 降低延迟和成本。ephemeral fork 保证并行审批不排队（一个长审批不阻塞后续审批）。这是"缓存复用 + 并行隔离"的权衡。
- **scope**：Guardian 自动审批场景。人工审批不使用此机制。

### EK-15：ActiveNetworkApproval——cancellation_token 生命周期绑定

- **claim**：活跃网络审批由 `ActiveNetworkApproval { registration_id, cancellation_token, execution_proxy }` 跟踪，session 释放时 cancellation_token 触发，代理 execution_proxy 随之取消。
- **source**：`codex-rs/core/src/tools/network_approval.rs:1138-1157`
- **detail**：`register_call()` 记录活跃网络审批调用（:1138-1151），返回 ActiveNetworkApproval。`cancellation_token` 在 session/turn 结束时触发，取消进行中的网络代理。`execution_proxy` 是 execution-scoped 的，随工具执行结束而销毁。
- **why**：网络代理是执行时构建的，必须与执行生命周期绑定，防止 session 释放后代理仍在运行或回调已释放的 session。cancellation_token 是 Rust async 生态的标准取消机制。
- **scope**：工具执行期间的网络治理。工具执行结束后 ActiveNetworkApproval drop，代理销毁。

### EK-16：set_permission_profile_projection——三字段原子同步

- **claim**：更新 permission_profile 时通过 `set_permission_profile_projection()` 原子同步 constrained profile、active profile id、workspace roots 三字段，并保留 deny-read 限制。
- **source**：`codex-rs/core/src/session/session.rs:504-538`
- **detail**：`apply()`（:360-502）在更新时调用此方法。`preserve_deny_read_restrictions_from()` 确保从旧 profile 继承 deny-read 限制。三字段通过 projection 一次性写入，不允许独立修改。
- **why**：三字段强相关，独立修改会导致"active id 指向 A 但 constrained profile 是 B"的不一致。deny-read 保留是安全不变量——profile 切换不能作为绕过读取限制的手段。
- **scope**：session 级权限更新。environment 级 config 变更走不同路径（effective_permission_profile 优先 environment）。

### EK-17：root_service_tier——ArcSwapOption 共享可变服务层级

- **claim**：AgentControl 持有 `root_service_tier: Arc<ArcSwapOption<String>>`，跨 agent 树共享可变的 service tier 配置，支持热更新。
- **source**：`codex-rs/core/src/agent/control.rs:133`
- **detail**：ArcSwapOption 是 ArcSwap 的 Option 特化，允许 store(None) 清空。所有 subagent 通过 clone 的 Arc 看到同一个 service tier。更新时 `store(Arc::new(Some("tier-name".into())))`。
- **why**：service tier 决定模型路由和功能可用性，可能在运行时变更（如灰度切换），需要跨 agent 树一致且支持热更新。ArcSwapOption 提供无锁读 + 原子写。
- **scope**：root agent 级配置。subagent 不独立设置 service tier。

---

## 三、关键决策（Key Decisions）——为什么这样设计

### EK-18：修正案仅在 AllowPrefixRules::Honor 模式生成

- **claim**：`proposed_execpolicy_amendment` 仅在 `AllowPrefixRules::Honor` 模式下计算生成，其他模式不产生修正案。
- **source**：`codex-rs/core/src/exec_policy.rs:358`（`auto_amendment_allowed`）
- **detail**：`create_exec_approval_requirement_for_parsed_commands`（:337-462）中，三态决策（Forbidden/Prompt→NeedsApproval/Allow→Skip）的修正案计算受 `auto_amendment_allowed` 门控。三个 derive 函数（`derive_requested_execpolicy_amendment_from_prefix_rule` :958-997、`try_derive_execpolicy_amendment_for_prompt_rules` :916-935、`try_derive_execpolicy_amendment_for_allow_rules` :940-956）仅在此模式激活。
- **why**：不是所有部署模式都希望"审批即学习"。managed/enterprise 环境可能要求固定策略集，不允许运行时自动追加规则。Honor 模式是"允许策略自演化"的显式开关。
- **scope**：策略修正案生成的全局门控。即使 Honor 模式，修正案仍需经过 BANNED 过滤等三道关卡（见 EK-20）。

### EK-19：Session 销毁时保守降级为 ask("not_allowed")

- **claim**：网络代理回调中 session 已释放（Weak upgrade 失败）时，返回 `NetworkDecision::ask("not_allowed")` 而非 Allow 或 Deny——保守降级为"需要人工审批"。
- **source**：`codex-rs/core/src/tools/network_approval.rs:1022-1024`
- **detail**：`let Some(session) = ...upgrade() else { return NetworkDecision::ask("not_allowed"); };`。ask 意味着将决策推给用户，not_allowed 是原因标签。
- **why**：控制面（session）生命周期结束时，执行面（代理）无法做动态决策。Allow 太危险（可能放行未授权请求），Deny 太激进（可能误杀合法请求），ask 是最保守的——将决策权交还给用户。这是"不可用时保守降级"原则的实例。
- **scope**：网络代理回调。Guardian 不可用时用更严格的 fail-closed（Deny/TimedOut，见 EK-25），因为 Guardian 是审批器本身不可用，而网络回调是控制面 session 不可用。

### EK-20：策略修正案三道过滤——BANNED / is_policy_match / simulate

- **claim**：自动固化的策略修正案在持久化前经过三道过滤：① BANNED_PREFIX_SUGGESTIONS（88 个危险前缀）② 已有 policy rule 匹配则不追加 ③ `prefix_rule_would_approve_all_commands()` 模拟验证所有命令段都变 Allow。
- **source**：`codex-rs/core/src/exec_policy.rs:958-997`
- **detail**：`derive_requested_execpolicy_amendment_from_prefix_rule` 依次执行：BANNED 检查（:970）→ `is_policy_match` 检查（:981）→ `simulate` 模拟添加后验证（:986）。任一道不通过则返回 None。
- **why**：自动固化是高风险操作——一条错误的免审规则可能永久放行危险命令。三道过滤分别防御：危险前缀（已知攻击向量）、规则冲突（避免重复/矛盾规则）、过度授权（防止一条前缀豁免所有命令）。
- **scope**：用户传入 prefix_rule 时的修正案推导。heuristics 推导（prompt_rules / allow_rules）走不同路径但同样受 BANNED 约束。

### EK-21：Granular 配置下的静默 Forbidden

- **claim**：当 `AskForApproval::Granular` 且 `allows_sandbox_approval()==false` 时，需要沙箱审批的命令直接 Forbidden，不产生用户提示——审批策略配置本身成为"静默拒绝器"。
- **source**：`codex-rs/core/src/tools/sandboxing.rs:209-218`
- **detail**：`default_exec_approval_requirement`（:194-230）将 AskForApproval（4 态：Never/OnRequest/Granular/UnlessTrusted）× FileSystemSandboxPolicy.kind（3 态）映射到三态。Granular + sandbox_approval disabled → Forbidden，不提示用户。
- **why**：用户显式配置了"不允许沙箱审批"，意味着这类命令在策略层面就被禁止，无需每次询问。这是"配置即策略"的实例——配置可以静默拒绝，减少用户噪音。
- **scope**：Granular 审批模式。Never 直接执行，OnRequest 提示用户，UnlessTrusted 必须人工审批，均不触发此路径。

### EK-22：Guardian 5 层上下文预算硬约束

- **claim**：Guardian 审查的输入上下文有 5 层硬上限：总 message transcript 20K tokens、总 tool transcript 10K、单条 message 5K、单条 tool result 1K、最近 40 条 entry。
- **source**：`codex-rs/core/src/guardian/mod.rs:71-77`
- **detail**：`GUARDIAN_MAX_MESSAGE_TRANSCRIPT_TOKENS = 20_000`、`GUARDIAN_MAX_TOOL_TRANSCRIPT_TOKENS = 10_000`、`GUARDIAN_MAX_MESSAGE_ENTRY_TOKENS = 5_000`、`GUARDIAN_MAX_TOOL_ENTRY_TOKENS = 1_000`、`GUARDIAN_RECENT_ENTRY_LIMIT = 40`。
- **why**：Guardian 审查需要足够上下文判断命令风险，但不能无限增长（成本 + 延迟 + prompt injection 攻击面）。5 层预算从总量到单条到最近条目逐级限制，平衡信息量和可控性。单条 tool result 1K 限制也限制了 prompt injection 载荷大小。
- **scope**：Guardian review session 输入。主会话上下文受 AGENTS.md 6 条规则约束（见 EK-08、EK-36），不同预算体系。

### EK-23：加权 token 计费——output 权重高于 input

- **claim**：RolloutBudget 本地 fallback 计费采用加权模型：`output_tokens × sampling_token_weight + non_cached_input × prefill_token_weight`，output 权重通常高于 input。
- **source**：`codex-rs/core/src/rollout_budget.rs:46-65`
- **detail**：优先使用服务端返回的 `codex_rollout_budget_units`（精确计费），缺失时 fallback 到本地加权。`sampling_token_weight` 和 `prefill_token_weight` 来自 RolloutBudgetConfig。non_cached_input 排除了缓存命中的 input（缓存 input 成本低）。
- **why**：LLM 计费中 output token 比 input token 贵（采样计算量更大），加权计费反映真实成本。non_cached_input 排除缓存因为缓存命中几乎无成本。服务端精确 units 优先因为服务端有更准确的计费数据。
- **scope**：RolloutBudget 资源配额。适用于任何按 token 计费的 LLM 资源配额系统。

### EK-24：no history rewrite——上下文增量构建

- **claim**：AGENTS.md 规定上下文必须增量构建，禁止历史重写（no history rewrite）。
- **source**：`AGENTS.md:91-100`（第 1 条）
- **detail**："No history rewrite - the context must be built up incrementally." 这是 6 条硬约束的第 1 条。
- **why**：历史重写会导致模型看到的上下文与实际对话历史不一致，破坏推理链的可追溯性。增量构建保证上下文是对话历史的严格前缀/超集，可审计可复现。
- **scope**：主会话上下文治理。Guardian review session 是独立构建的审查上下文，不受此规则约束（但有自己的 5 层预算）。

### EK-25：Guardian fail-closed——超时/解析/会话错误默认 Deny

- **claim**：Guardian 审查出错时默认拒绝：Timeout → ReviewDecision::TimedOut（不执行）；PromptBuild/Session/Parse 错误 → 构造 risk_level=High, outcome=Deny 的 assessment。
- **source**：`codex-rs/core/src/guardian/review.rs:616, 655-689`
- **detail**：`GuardianReviewError::PromptBuild | Session | Parse => (GuardianAssessment { risk_level: High, outcome: Deny, ... }, false)`——false 表示不计入熔断器（因为不是 Guardian 的判断拒绝，是系统错误）。Timeout → `ReviewDecision::TimedOut`。仅 transient 错误（ServerOverloaded/HttpConnectionFailed/ResponseStreamConnectionFailed/InternalServerError/ResponseStreamDisconnected/Parse）重试，最多 3 次，90s deadline 内。
- **why**：审批器不可用时，"不执行"比"可能执行危险命令"安全。fail-closed 是安全系统的标准设计。transient 错误重试因为可能是临时网络问题，non-transient 错误直接 Deny 因为重试也无用。
- **scope**：Guardian 自动审批。人工审批不存在此问题（用户总是可以选择）。网络回调用 ask("not_allowed") 而非 Deny（见 EK-19），因为网络回调是控制面不可用而非审批器不可用。

---

## 四、失败与修复（Failures & Fixes）——踩坑/修复/回退/异常路径

### EK-26：PendingApprovalDecision::Deny——drop 默认拒绝

- **claim**：待处理的审批决策（PendingApprovalDecision）在 drop 时默认 Deny——如果审批流程未完成就被丢弃，命令不执行。
- **source**：`codex-rs/core/src/tools/network_approval.rs:300`（`PendingApprovalDecision::Deny` drop 默认）
- **detail**：PendingApprovalDecision 代表一个尚未完成的审批决策。其 Drop 实现默认 Deny（而非 Allow 或挂起）。这意味着如果 session 结束、turn 中断、或审批流程被取消，进行中的审批默认拒绝。
- **why**：审批未完成 = 未获得授权 = 不应执行。drop 默认 Deny 是 RAII 风格的安全保证——无论什么原因导致审批流程中断，都不会意外放行。这是"未授权即拒绝"的失败安全设计。
- **scope**：所有待处理审批决策。适用于任何有异步审批流程的系统——审批对象的生命周期必须绑定决策结果，drop 时默认拒绝。

### EK-27：策略固化幂等检查——防止重复 store

- **claim**：`append_amendment_and_update` 在落盘后、store 前检查当前内存策略是否已 Allow该前缀，已 Allow 则跳过 store——幂等性保证。
- **source**：`codex-rs/core/src/exec_policy.rs:491-506`
- **detail**：落盘后 `let existing_evaluation = current_policy.check_multiple_with_options([&amendment.command], &|_| Decision::Forbidden, &match_options); if already_allowed { return Ok(()); }`。只有当前策略未 Allow 时才 clone→modify→store。
- **why**：并发场景下两个审批可能同时通过同一命令的修正案，Semaphore 串行化了写操作但第一个完成后第二个仍会执行。幂等检查防止第二个重复 store（虽然 store 本身是幂等的，但跳过可减少不必要的 Arc 替换和缓存失效）。
- **scope**：命令前缀规则固化。网络规则固化（append_network_rule_and_update）同理。

### EK-28：crash 窗口——落盘成功但 store 前崩溃的最终一致性

- **claim**：`append_amendment_and_update` 存在 crash 窗口：落盘成功但 `policy.store()` 前进程崩溃，导致内存策略与磁盘不一致。重启时从磁盘全量加载保证最终一致。
- **source**：`codex-rs/core/src/exec_policy.rs:303-309, 464-512`
- **detail**：`load_exec_policy`（:303-309）在启动时从磁盘全量加载。crash 窗口内的新规则在重启后从磁盘恢复。这是最终一致而非强一致——crash 后到重启前，内存中缺少新规则（但进程已崩溃，无影响）。
- **why**：强一致需要 fsync + 原子 rename + 版本号等复杂机制，对于"用户审批后追加一条规则"的低频操作，最终一致可接受。重启全量加载是简单可靠的恢复机制。
- **scope**：策略持久化。适用于任何"写后读"的配置系统，低频写可接受最终一致。

### EK-29：requirements overlay 静默覆盖——缺少冲突可见性

- **claim**：`requirements().exec_policy` 通过 `merge_overlay` 叠加在 user/project 规则之上，同前缀规则静默覆盖（requirements 优先级更高），无显式冲突检测日志。
- **source**：`codex-rs/core/src/exec_policy.rs:662-716, 711-715`
- **detail**：`load_exec_policy` 遍历 config_stack.layers_low_to_high() 加载，最后 `merge_overlay(requirements_policy)`。overlay 语义是"覆盖"——requirements 层同前缀规则优先级更高。若 user 层有 `allow python` 而 requirements 有 `forbid python`，后者静默覆盖前者，用户可能不知道。
- **why**：managed/enterprise 环境需要强制策略覆盖用户配置，静默覆盖是有意设计（防止用户绕过企业策略）。但缺少冲突可见性是已知局限——用户可能困惑"为什么我的规则不生效"。
- **scope**：多层策略叠加。`ignore_user_and_project_exec_policy_rules()`（:670-677）可完全跳过 User/Project 层，是更强的强制策略机制。

### EK-30：前缀匹配粒度——固化 `python build.py` 后 `python build.py --deploy-production` 被豁免

- **claim**：策略修正案固化的是精确前缀匹配（prefix rule），`python build.py` 固化后不会豁免 `python -c "..."`（在 BANNED 列表），但会豁免 `python build.py --deploy-production`（同前缀）。
- **source**：`codex-rs/core/src/exec_policy.rs:958-997`（prefix rule 语义）
- **detail**：`prefix_rule_would_approve_all_commands()` 模拟添加后验证所有解析出的命令段都变 Allow。前缀匹配意味着同前缀的所有后续参数都被豁免。这是 prefix rule 的固有局限。
- **why**：前缀匹配是简单高效的策略匹配方式，适合"信任这个命令的所有用法"场景。但对于"信任这个命令的特定参数"场景，前缀匹配粒度过粗。BANNED_PREFIX_SUGGESTIONS 部分缓解了最危险的前缀（shell 包装器、解释器 -c/-e）。
- **scope**：命令前缀规则。网络规则是 host + protocol 匹配（见 EK-31），不同匹配语义。

### EK-31：网络规则匹配语义边界——host + protocol，子域名匹配未显式说明

- **claim**：网络规则匹配粒度是 host + protocol（NetworkApprovalProtocol: Http/Https/Socks5Tcp/Socks5Udp），不是子域名通配。`allow api.example.com:443` 不会放行 `api.example.com:8080`。但 `allow example.com` 是否匹配 `api.example.com` 取决于 codex-execpolicy 的匹配语义，代码中未显式说明。
- **source**：`codex-rs/core/src/network_policy_decision.rs:79-84`
- **detail**：`NetworkApprovalProtocol` 枚举 4 种协议。`execpolicy_network_rule_amendment`（:74-102）将 NetworkPolicyAmendment 转换为 ExecPolicyNetworkRuleAmendment。子域名匹配语义在 codex-execpolicy crate 中，core 代码未展开。
- **why**：host + protocol 精确匹配是安全的默认——不会意外放行子域名或不同端口。但子域名匹配语义的不明确是一个边界模糊点，可能导致用户对规则生效范围的误解。
- **scope**：网络域名规则。需查阅 codex-execpolicy crate 确认子域名匹配语义。

---

## 五、测试揭示的行为（Test-revealed Behavior）

### EK-32：Full Access 短路——has_full_access 直接 Approved 跳过 Guardian

- **claim**：当 session 处于 Full Access 状态（approval_policy==Never ∧ 无沙箱限制）时，Guardian 审查完全跳过，直接返回 Approved。
- **source**：`codex-rs/core/src/guardian/review.rs:330-343`
- **detail**：`if context.environments().has_full_access(context.approval_policy, &turn.config.permissions.effective_permission_profile()) { return if external_cancel...is_cancelled() { Abort } else { Approved }; }`
- **why**：Full Access 是用户显式选择的最高权限模式——所有命令直接执行，本来就不经过任何审批（不只是 Guardian）。Guardian 短路是一致性设计：既然不审批，就不浪费资源做审查。
- **scope**：Full Access 配置。这是显式用户选择，不是运行时可被诱导进入的状态。Full Access 下所有命令直接执行。

### EK-33：denied_read_active 时沙箱升级恒不可能——NoOverride

- **claim**：当存在 denied-read 限制时，沙箱升级（escalation 到无沙箱执行）恒不可能——必须保留沙箱以执行 deny-read 限制。
- **source**：`codex-rs/core/src/tools/sandboxing.rs:238-267`
- **detail**：`sandbox_override_for_first_attempt()` 中：`denied_read_active? → NoOverride`。`Skip{bypass_sandbox:true} → BypassSandboxFirstAttempt`。`sandbox_permissions.requires_escalated_permissions() → BypassSandboxFirstAttempt`。denied-read 优先于所有升级路径。
- **why**：deny-read 限制通过沙箱的文件系统过滤实现，跳出沙箱等于绕过 deny-read。因此只要有 deny-read 限制，就必须保留沙箱，即使命令本身被 Skip 或需要升级权限。
- **scope**：沙箱升级决策。deny-read 是安全不变量，优先于执行便利性。

### EK-34：record_non_denial 重置连续拒绝计数

- **claim**：Guardian 熔断器的连续拒绝计数在每次非拒绝（Allow/Ask/Abort）时通过 `record_non_denial()` 重置。
- **source**：`codex-rs/core/src/guardian/mod.rs:234-238`
- **detail**：`record_non_denial()` 重置 consecutive 计数。熔断器使用双阈值：连续拒绝计数 + 滑动窗口（最近 50 次中拒绝数）。连续计数可被单次非拒绝重置，滑动窗口不受单次影响。
- **why**：连续计数检测"持续拒绝"模式（agent 反复尝试同类危险命令），一次成功审批说明 agent 调整了行为，应重置。滑动窗口检测"整体拒绝率"（即使偶尔成功，拒绝率仍高），不被单次成功重置。双阈值互补。
- **scope**：Guardian 熔断器。Standard 模型阈值 3 连续/10 近期，Cyber 模型 1 连续/1 近期（见 EK-35）。

### EK-35：Cyber 模型熔断器阈值更敏感——1 次即中断

- **claim**：Cyber 专用模型（`MODEL_SPECIALTY_CYBER`）的 Guardian 熔断器阈值比 Standard 更敏感：连续 1 次拒绝或最近 50 次中 1 次拒绝即中断 turn。
- **source**：`codex-rs/core/src/guardian/mod.rs:64-68`，`review.rs:258-262`
- **detail**：`MAX_CONSECUTIVE_CYBER_GUARDIAN_DENIALS_PER_TURN = 1`、`MAX_RECENT_CYBER_AUTO_REVIEW_DENIALS_PER_TURN = 1`。Standard 为 3/10。Cyber 模型触发条件 `model_specialty==cyber`。
- **why**：Cyber 专用模型的拒绝意味着更高的安全风险（该模型专门用于安全敏感场景，其拒绝信号权重更高）。更敏感的阈值防止在高安全场景下反复尝试被拒绝的命令。
- **scope**：Cyber 模型配置。非 Cyber 模型使用 Standard 阈值。

### EK-36：>1K tokens 新 context item 标记 P0 需人工审查

- **claim**：AGENTS.md 规定新增的 context item 若可能超过 1K tokens，标记为 P0 优先级，需要额外人工审查。
- **source**：`AGENTS.md:91-100`（第 5 条）
- **detail**："Highlight new individual items that can cross >1k tokens as P0. These need an additional manual review." 这是 6 条硬约束的第 5 条。第 4 条规定 no items larger than 10K tokens。
- **why**：大 context item 可能包含意外内容（敏感信息、prompt injection、冗余数据），且对成本和缓存命中率影响大。P0 标记 + 人工审查确保大注入项经过严格评估。这是 review 期强制（非编译期）。
- **scope**：主会话上下文注入。Guardian 上下文有独立的 1K per-tool 硬限制（见 EK-22），是编译期/运行期强制。

---

## 六、重要配置（Important Config）——关键配置项

### EK-37：BANNED_PREFIX_SUGGESTIONS——88 个禁止自动固化的危险前缀

- **claim**：约 88 个前缀被永久禁止自动固化为免审规则，包括所有 shell 包装器（`bash -c`、`sh -lc`、`powershell -Command`）、所有解释器（`python -c`、`node -e`、`perl -e`）、`sudo`、`rm`、`git` 等。
- **source**：`codex-rs/core/src/exec_policy.rs:57-146`
- **detail**：静态数组 `BANNED_PREFIX_SUGGESTIONS`。这些前缀的命令每次都必须重新审批，用户永远不能通过"允许这一次"来自动固化。这是三道过滤的第一道（:970）。
- **why**：shell 包装器和解释器的 `-c`/`-e` 形式是"前缀看似安全实际任意执行"的最常见攻击向量——`bash -c "任意命令"` 固化后等于放行任意命令。`sudo`、`rm`、`git` 是高风险操作。永久禁止自动固化是关键安全不变量。
- **scope**：自动固化修正案。用户仍可手动编辑 default.rules 添加这些前缀（但不推荐）。BANNED 仅限制"审批时自动提议固化"。

### EK-38：AskForApproval 四态配置

- **claim**：审批策略配置 `AskForApproval` 是四态：Never（从不审批直接执行）、OnRequest（按需审批）、Granular（细粒度，带 sandbox_approval + rules 子开关）、UnlessTrusted（非信任命令必须审批）。
- **source**：`codex-rs/core/src/tools/sandboxing.rs:189-207`，`exec_policy.rs:49-53`
- **detail**：Granular 携带子配置（sandbox_approval 开关 + 自定义 rules）。四态与 FileSystemSandboxPolicy.kind（Restricted/Unrestricted/ExternalSandbox）组合映射到 ExecApprovalRequirement 三态。
- **why**：不同用户/场景需要不同审批严格度。Never 适合信任环境，OnRequest 适合常规，Granular 适合需要精细控制的，UnlessTrusted 适合高安全场景。四态比二元"审批/不审批"提供更细的控制粒度。
- **scope**：session 级审批配置。Guardian 路由仅 OnRequest/Granular + AutoReview（见 EK-03）。

### EK-39：Guardian 超时与重试配置

- **claim**：Guardian 审查配置：`GUARDIAN_REVIEW_TIMEOUT = 90s`、`GUARDIAN_REVIEW_MAX_ATTEMPTS = 3`，仅 transient 错误重试，且在 90s deadline 内。
- **source**：`codex-rs/core/src/guardian/mod.rs:62`，`review.rs:83, 1098, 1145-1161`
- **detail**：transient 错误列表：ServerOverloaded、HttpConnectionFailed、ResponseStreamConnectionFailed、InternalServerError、ResponseStreamDisconnected、Parse。重试在 90s deadline 内（总超时不超过 90s）。non-transient 错误直接 Deny。
- **why**：90s 是用户可接受的审批等待上限（超过则用户体验差）。3 次重试平衡了 transient 错误恢复和总延迟。仅 transient 错误重试因为 non-transient 错误（如 PromptBuild 失败）重试也无用。
- **scope**：Guardian 自动审批。人工审批无超时配置（用户决定何时响应）。

### EK-40：多层 config stack——builtin < user < project < requirements overlay

- **claim**：策略加载遍历 `config_stack.layers_low_to_high()`，从 builtin → user → project 层加载，最后叠加 requirements overlay。高层覆盖低层。
- **source**：`codex-rs/core/src/exec_policy.rs:662-716`
- **detail**：每层从 `config_folder/rules/*.rules` 加载（:668-683）。低到高顺序 append，后加载的规则优先级更高。`ignore_user_and_project_exec_policy_rules()` 可跳过 User/Project 层（:670-677）。`merge_overlay(requirements_policy)` 最后叠加（:711-715）。
- **why**：多层配置允许不同范围的策略共存——builtin 是出厂默认，user 是用户个人配置，project 是项目级配置，requirements 是托管/企业强制策略。高层覆盖低层实现"越具体越优先"。
- **scope**：exec_policy 策略加载。配置层叠是 Codex 全局配置系统的通用模式（不限于 exec_policy）。

### EK-41：AgentExecutionLimiter——effective_agent_max_threads 并发限制

- **claim**：AgentControl 持有 `agent_execution_limiter: Arc<AgentExecutionLimiter>`，在 `with_session_id` 中用 `effective_agent_max_threads` 初始化，限制整个 agent 树的并发 agent 数。
- **source**：`codex-rs/core/src/agent/control.rs:129, 169-173`
- **detail**：`effective_agent_max_threads` 来自配置（可能是用户配置或 environment 配置）。AgentExecutionLimiter 是信号量模式，超过并发数的 subagent spawn 等待。
- **why**：无限制的 subagent 并发会导致资源耗尽（token 预算、文件描述符、网络连接）。并发限制是多 agent 系统的基本资源治理。effective 前缀说明实际值可能受 environment/profile 覆盖。
- **scope**：root agent 树并发。与 RolloutBudget（token 预算）互补——一个限制并发数，一个限制总 token 消耗。

### EK-42：策略文件路径——codex_home/rules/default.rules

- **claim**：策略持久化文件路径为 `codex_home/rules/default.rules`，由常量 `RULES_DIR_NAME="rules"` 和 `DEFAULT_POLICY_FILE="default.rules"` 定义。
- **source**：`codex-rs/core/src/exec_policy.rs:54-56, 867-869`
- **detail**：`default_policy_path(codex_home)` 返回 `codex_home.join(RULES_DIR_NAME).join(DEFAULT_POLICY_FILE)`。命令前缀规则和网络域名规则都写入此文件。
- **why**：统一策略文件简化管理——用户可手动编辑此文件自定义规则。rules 子目录与其他配置（config.toml 等）分离。default.rules 是默认策略文件，未来可能支持多策略文件。
- **scope**：exec_policy 持久化。codex_home 是 Codex 数据目录（通常 ~/.codex/）。

---

## 七、边界与例外（Boundaries & Exceptions）——适用/不适用条件

### EK-43：ignore_user_and_project_exec_policy_rules——托管环境强制策略

- **claim**：`ignore_user_and_project_exec_policy_rules()` 可跳过 User 和 Project 层的规则，仅 builtin + requirements 生效，用于 managed/enterprise 环境的强制策略。
- **source**：`codex-rs/core/src/exec_policy.rs:670-677`
- **detail**：在 `load_exec_policy` 中检查此标志，若设置则跳过 User/Project 层的 `config_folder/rules/*.rules` 加载。这比 overlay 覆盖更强——不是"覆盖"而是"完全忽略"用户/项目规则。
- **why**：企业/托管环境需要确保用户无法通过个人或项目配置绕过企业安全策略。overlay 覆盖仍允许用户规则存在（只是被覆盖），ignore 完全移除用户规则的影响面。
- **scope**：managed/enterprise 部署。普通用户部署不设置此标志，User/Project 层规则正常生效。

### EK-44：修正案无法跳过显式策略规则——prompt_rules derive 返回 None

- **claim**：`try_derive_execpolicy_amendment_for_prompt_rules` 从 heuristics Prompt 推导修正案，但如果有显式 policy rule Prompt 则返回 None——修正案无法跳过显式策略规则。
- **source**：`codex-rs/core/src/exec_policy.rs:916-935`
- **detail**：三个 derive 函数之一。如果 exec_policy 中已有显式的 Prompt 规则（用户/管理员配置的规则），则 heuristics 推导的修正案不生成——因为显式规则优先级更高，修正案不应覆盖它。
- **why**：显式策略规则是用户/管理员的明确意图，自动推导的修正案不应与之冲突或覆盖。返回 None 意味着"这个命令的审批由显式规则决定，不自动提议固化"。
- **scope**：heuristics Prompt 推导。`try_derive_execpolicy_amendment_for_allow_rules`（:940-956）同理，仅在沙箱失败后提示用户跳出沙箱时使用。

### EK-45：网络治理仅 managed_network 激活时生效

- **claim**：网络审批闭环仅在 `managed_network_active == true` 且 spec 为 Some 时生效。spec 为 None 或 managed_network 未激活时，`begin_network_approval` 返回 None，无网络治理。
- **source**：`codex-rs/core/src/tools/network_approval.rs:1032-1158`
- **detail**：`begin_network_approval(session, turn, managed_network_active, spec)` 开头检查，若条件不满足返回 None。工具执行时若 ActiveNetworkApproval 为 None，则不注入网络代理，命令直接访问网络（受系统网络配置约束）。
- **why**：网络代理是有开销的（延迟、兼容性），不是所有环境都需要。managed_network 是显式配置的网络治理模式，未配置时不启用。这是"功能按需激活"的设计。
- **scope**：网络审批闭环。即使无网络治理，exec_policy 命令级审批仍生效（网络访问是命令执行的副作用）。

### EK-46：RolloutBudget 未配置时 no-op

- **claim**：RolloutBudget 未配置（OnceLock 未初始化）时，`record_usage()` 返回 false（未耗尽），`pending_reminder()` 返回 None，所有方法 no-op。
- **source**：`codex-rs/core/src/rollout_budget.rs:33-44, 67-118`
- **detail**：`lock()` 返回 None 时，`record_usage` 返回 `Ok(false)`，`pending_reminder` 返回 `Ok(None)`。这意味着无 rollout 配置时，预算系统完全透明——不限制、不提醒、不报错。
- **why**：RolloutBudget 是可选功能（灰度发布/资源配额场景），不是所有部署都需要。no-op 设计保证无配置时不影响正常功能，也不要求调用方检查"是否已配置"。
- **scope**：RolloutBudget 资源配额。配置来自 rollout_budget_config（AgentControl::new 参数）。

### EK-47：Guardian 仅 OnRequest/Granular + AutoReview 激活

- **claim**：Guardian 自动审批仅在 `approval_policy ∈ {OnRequest, Granular}` 且 `approvals_reviewer == AutoReview` 时激活。Never 直接执行，UnlessTrusted 必须人工审批，均不走 Guardian。
- **source**：`codex-rs/core/src/guardian/review.rs:209-217`
- **detail**：`routes_approval_policy_to_guardian()` 函数。Never 模式下所有命令直接执行（无需审批），UnlessTrusted 模式下非信任命令必须人工审批（不自动审查）。只有 OnRequest（按需审批）和 Granular（细粒度审批）且配置了 AutoReview（自动审查器）才走 Guardian。
- **why**：Guardian 是"用 AI 审批 AI"的自动审查器，仅在用户配置了"需要审批且允许自动审查"时才有意义。Never 不需要审批，UnlessTrusted 需要人工审批（不信任自动审查器）。
- **scope**：Guardian 路由。Full Access 短路（见 EK-32）是在此路由之后的进一步短路。

### EK-48：PromptBuild/Session/Parse 错误不计入熔断器

- **claim**：Guardian 的 PromptBuild/Session/Parse 错误导致的 Deny 不计入熔断器（`false` 参数），因为这是系统错误而非 Guardian 的判断拒绝。
- **source**：`codex-rs/core/src/guardian/review.rs:655-689`
- **detail**：错误处理返回 `(GuardianAssessment { outcome: Deny, ... }, false)`——第二个字段 `false` 表示不计入熔断器。只有 Guardian 明确判断的 Deny（risk assessment 结果）才计入熔断器。
- **why**：熔断器检测"Guardian 持续拒绝命令"的模式（agent 反复尝试危险命令）。系统错误导致的 Deny 不是 Guardian 的判断，计入熔断器会导致"系统不稳定时误触发熔断"。区分"判断拒绝"和"错误拒绝"是精确的熔断器设计。
- **scope**：Guardian 熔断器。Timeout → TimedOut 也不计入熔断器（类似逻辑）。

### EK-49：网络规则与命令规则共享持久化但匹配语义不同

- **claim**：网络域名规则和命令前缀规则共享同一套持久化机制（update_lock + spawn_blocking + blocking_append + ArcSwap 热更新），都写入 default.rules，但匹配语义不同——命令是前缀匹配，网络是 host + protocol 匹配。
- **source**：`codex-rs/core/src/exec_policy.rs:514-558`（网络规则），`:464-512`（命令规则）
- **detail**：`append_network_rule_and_update` 与 `append_amendment_and_update` 结构同构。但 `blocking_append_network_rule` 写入的是网络规则格式（host + protocol + decision），与命令前缀规则格式不同。codex-execpolicy parser 区分两种规则。
- **why**：共享持久化机制减少代码重复（同一套并发安全 + 落盘 + 热更新）。但匹配语义必须不同——命令前缀匹配适合"信任这个命令"，网络 host+protocol 匹配适合"信任这个域名的这种协议"。
- **scope**：exec_policy 持久化。两种规则在 default.rules 中可能共存，parser 按格式区分。

### EK-50：Guardian review session 不继承父 exec-policy 规则

- **claim**：Guardian 运行在独立 review session 中，`approval_policy=never` 且**不继承父 session 的 exec-policy 规则**，防止父 session 的免审规则影响 Guardian 判断。
- **source**：`codex-rs/core/src/guardian/review.rs:977-990`（doc comment）
- **detail**：doc comment："it intentionally runs without inherited exec-policy rules." Guardian review session 可复用父 session 的 managed-network allowlist（只读检查），但不继承 exec-policy。approval_policy=never 意味着 Guardian 自己执行命令时不需要审批（它是只读审查，不执行命令）。
- **why**：如果 Guardian 继承父 session 的 exec-policy，父 session 已固化的免审规则会让 Guardian 对同类命令直接放行，失去审查意义。独立 exec-policy 保证 Guardian 对每个命令做独立判断，不受父 session 审批历史影响。
- **scope**：Guardian review session。这是 Guardian 隔离设计的核心部分（与独立 session、只读沙箱并列）。

### EK-51：主上下文与 Guardian 上下文预算体系分离

- **claim**：主会话上下文受 AGENTS.md 6 条规则约束（no rewrite / cache friendly / bounded / 10K cap / 1K P0 / trait），Guardian review session 受独立 5 层预算约束（20K/10K/5K/1K/40 entries），两套体系分离。
- **source**：`AGENTS.md:91-100`，`codex-rs/core/src/guardian/mod.rs:71-77`
- **detail**：主上下文是增量构建的对话历史 + 50+ context fragment，预算是原则性规则（review 期强制）。Guardian 上下文是为单次审查构建的压缩 transcript，预算是硬编码常量（运行期强制）。两者目的不同——主上下文支持长期对话，Guardian 上下文支持单次风险判断。
- **why**：不同场景需要不同上下文治理策略。主上下文需要保留历史连贯性，Guardian 上下文需要控制单次审查的信息量和攻击面。分离避免一套预算约束两种不同需求。
- **scope**：上下文治理。未来可能有其他子系统的独立上下文预算（如 subagent session）。

### EK-52：Forbidden 不产生修正案——拒绝不可学习

- **claim**：`ExecApprovalRequirement::Forbidden` 不携带 `proposed_execpolicy_amendment`——被禁止的命令永远不会被自动固化为免审规则。
- **source**：`codex-rs/core/src/tools/sandboxing.rs:152-171, 174-186`
- **detail**：三态枚举中 Forbidden 只有 `reason: String` 字段，无 `proposed_execpolicy_amendment`。`proposed_execpolicy_amendment()` 方法对 Forbidden 返回 None。
- **why**：被禁止的命令意味着策略层明确拒绝（显式 Forbidden 规则或 Granular 静默拒绝），自动固化为免审规则等于绕过禁止。这是安全不变量——"拒绝不可学习"，与 Skip/NeedsApproval 的"审批可学习"形成对比。
- **scope**：exec_policy 审批闭环。Forbidden 是最终决策，不进入策略学习路径。

---

## 附录：Engineering Knowledge 索引与分类统计

| 类别 | 编号范围 | 数量 |
|------|---------|------|
| 核心机制 | EK-01 ~ EK-08 | 8 |
| 关键实现 | EK-09 ~ EK-17 | 9 |
| 关键决策 | EK-18 ~ EK-25 | 8 |
| 失败与修复 | EK-26 ~ EK-31 | 6 |
| 测试揭示的行为 | EK-32 ~ EK-36 | 5 |
| 重要配置 | EK-37 ~ EK-42 | 6 |
| 边界与例外 | EK-43 ~ EK-52 | 10 |
| **总计** | | **52** |

### 关键实现细节保留检查（v3 验收点）

| 实现细节 | 对应 EK | 状态 |
|---------|---------|------|
| Weak\<Session\> 防止循环引用 | EK-10 | ✅ |
| cancellation_token 生命周期绑定 | EK-15 | ✅ |
| drop 默认拒绝（PendingApprovalDecision::Deny） | EK-26 | ✅ |
| Semaphore(1) 串行化策略更新 | EK-02, EK-09 | ✅ |
| ArcSwap 热更新（无锁读 + 原子替换） | EK-02, EK-09, EK-17 | ✅ |
| spawn_blocking 落盘 | EK-12 | ✅ |
| BANNED_PREFIX_SUGGESTIONS 88 个 | EK-20, EK-37 | ✅ |
| 三道过滤（BANNED / is_policy_match / simulate） | EK-20 | ✅ |
| Guardian fail-closed（Timeout/Parse/Session → Deny） | EK-25, EK-48 | ✅ |
| 熔断器双阈值（连续 + 滑动窗口） | EK-03, EK-34, EK-35 | ✅ |
| 加权 token 计费（output > input） | EK-23 | ✅ |
| OnceLock 延迟初始化 | EK-13 | ✅ |
| trunk 复用 + ephemeral fork | EK-14 | ✅ |
| deny-read 保留（profile 切换） | EK-06, EK-16 | ✅ |
| 幂等检查（防重复 store） | EK-27 | ✅ |
| 多层 config stack + overlay | EK-40, EK-29, EK-43 | ✅ |
| no history rewrite | EK-24 | ✅ |
| ContextualUserFragment trait 类型安全 | EK-08 | ✅ |
| Full Access 短路 | EK-32 | ✅ |
| denied_read_active → NoOverride | EK-33 | ✅ |

**结论**：52 条 Engineering Knowledge 覆盖了 v2 证据包的全部核心机制，保留了所有关键实现细节（Weak ref / cancellation_token / drop / Semaphore / ArcSwap / spawn_blocking / BANNED / 三道过滤 / fail-closed / 熔断器 / 加权计费 / OnceLock / trunk+fork / deny-read / 幂等 / 多层 stack / no rewrite / trait / Full Access 短路 / NoOverride），作为未来推理的无损底座。

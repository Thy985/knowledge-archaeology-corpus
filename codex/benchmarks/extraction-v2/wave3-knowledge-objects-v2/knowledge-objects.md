# Wave 3 — 知识对象 (v2)

> 每个 KO 包含 L1→L5 五层阶梯、晋升论证块、反例攻击（L3+ ≥3 条）、降级记录、flow_traceability。epistemic_status 诚实标注。

---

## KO-1：三态审批枚举 + 修正案携带模式

**KO ID**: ko-tri-state-approval-with-amendment
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（代码直接验证，多文件交叉确认）
**flow_traceability**: Policy Flow Phase 1 → Phase 4

### L1 事实（带 flow_traceability）

| 事实 | 锚点 | Flow Edge |
|------|------|-----------|
| ExecApprovalRequirement 是三态枚举：Skip{bypass_sandbox, proposed_execpolicy_amendment}、NeedsApproval{reason, proposed_execpolicy_amendment}、Forbidden{reason} | sandboxing.rs:152-171 | Policy Flow Phase 1 输出 |
| Skip 和 NeedsApproval 都携带 proposed_execpolicy_amendment，Forbidden 不携带 | sandboxing.rs:174-186 | Policy Flow Phase 1→4 衔接 |
| default_exec_approval_requirement 将 AskForApproval（4 态）× FileSystemSandboxKind（3 态）映射到三态 | sandboxing.rs:194-230 | Policy Flow Phase 1 |
| Granular 且 allows_sandbox_approval()==false 时直接 Forbidden，不提示用户 | sandboxing.rs:209-218 | Policy Flow Phase 1 静默拒绝路径 |
| exec_policy 规则匹配产生 Decision::{Forbidden, Prompt, Allow}，再映射到 ExecApprovalRequirement | exec_policy.rs:394-461 | Policy Flow Phase 1 内部 |
| Allow 时 bypass_sandbox 仅当所有解析出的命令段都有显式 policy Allow | exec_policy.rs:440-454 | Policy Flow Phase 1→3 衔接 |

### L2 知识

审批结果不是简单的"允许/拒绝"二元决策，而是一个携带元数据的三态结构。核心洞察：**审批决策本身就是策略学习的输入**——Skip 和 NeedsApproval 都携带"是否应该把这个命令固化为免审规则"的修正案建议，Forbidden 不产生修正案（拒绝不可学习）。这形成了一个"执行→审批→固化→未来免审"的闭环。

Granular 配置下的静默 Forbidden 是一个易被忽略的路径：当用户配置了 `sandbox_approval=false` 时，需要沙箱审批的命令直接被拒绝，不产生用户提示。这意味着审批策略配置本身可以成为一个"静默拒绝器"。

### L3 模式（晋升论证）

**模式名称**：三态审批 + 修正案携带（Tri-State Approval with Amendment Carrying）

**晋升论证 L2→L3**：
- **可迁移性论证**：此模式不依赖 Codex 特定技术栈。任何需要"运行时审批 + 审批结果可学习固化"的系统都适用：CI/CD 审批门、云资源申请审批、API 速率限制豁免审批。
- **跨实例验证**：三态结构（Skip/NeedsApproval/Forbidden）在多个工具运行时中统一使用（shell、apply_patch、unified_exec），不是单个工具的特例。
- **不变量论证**：Forbidden 不携带修正案是一个安全不变量——拒绝的命令永远不会被自动固化为免审规则，防止"拒绝一次后误固化"。

**模式描述**：
审批系统的输出应是三态（直接执行/需要审批/禁止），且前两态应携带"是否建议固化为免审规则"的修正案。禁止态不携带修正案。修正案在持久化前经过危险前缀过滤、冲突检测和模拟验证三道关卡。

### 反例攻击（3 条）

1. **攻击：修正案可能被攻击者构造为"看似安全但实际危险"的前缀。**
   - 反驳：BANNED_PREFIX_SUGGESTIONS（88 个）覆盖了所有 shell 包装器和解释器的 `-c`/`-e` 形式（exec_policy.rs:57-146），这些是最常见的"前缀看似安全实际任意执行"向量。但自定义二进制的危险子命令（如 `docker run --privileged`）不在 BANNED 列表中，依赖用户在审批时判断。→ **部分缓解，非完全防御**。

2. **攻击：NeedsApproval 被用户拒绝后，proposed_execpolicy_amendment 仍可能被错误持久化。**
   - 反驳：append_amendment_and_update 仅在审批通过后调用（Policy Flow Phase 4 入口是审批通过路径），拒绝路径不触发固化。代码中 `append_amendment_and_update` 的调用点在用户选择"允许并记住"之后。→ **不成立**。

3. **攻击：Forbidden 不携带修正案，但攻击者可以通过"先执行一个类似但安全的命令获得 Skip + 修正案，固化后再执行危险变体"来绕过。**
   - 反驳：修正案固化的是精确前缀匹配（prefix rule），`python script.py` 固化后不会豁免 `python -c "import os; os.system('rm -rf /')"`，因为后者前缀是 `python -c`，在 BANNED 列表中。但 `python build.py` 固化后，`python build.py --deploy-production` 会被豁免（前缀匹配）。→ **部分成立：前缀匹配的粒度是此模式的固有局限**。

### 降级记录

- **尝试 L4（认知模型）**：曾尝试将此模式抽象为"审批即学习"的通用认知模型，但发现固化逻辑（BANNED 过滤、幂等检查、ArcSwap 热更新）与 Codex 的 ExecPolicyManager 强耦合，不具备跨系统的模型级通用性。→ **降级为 L3 Pattern**。
- **L5 方法论不适用**：此模式不构成方法论级别的指导原则，仅是一个可复用的设计模式。

---

## KO-2：策略持久化热更新模型

**KO ID**: ko-policy-persistence-hot-reload
**类型**: Model (L4)
**epistemic_status**: Validated Model（代码直接验证，含并发安全分析）
**flow_traceability**: Policy Flow Phase 0 + Phase 4

### L1 事实

| 事实 | 锚点 | Flow Edge |
|------|------|-----------|
| ExecPolicyManager 持有 ArcSwap<Policy> + Semaphore(1) | exec_policy.rs:276-279 | Phase 0 初始化 |
| 读路径 current() = policy.load_full()（无锁） | exec_policy.rs:311-313 | Phase 1 每次审批查询 |
| 写路径 append_amendment_and_update：acquire Semaphore → spawn_blocking(blocking_append_allow_prefix_rule) → 幂等检查 → clone→add_prefix_rule→store(Arc::new) | exec_policy.rs:464-512 | Phase 4 固化 |
| 策略文件路径 = codex_home/rules/default.rules | exec_policy.rs:867-869 | Phase 4 落盘 |
| 策略加载遍历 config_stack.layers_low_to_high()，高层覆盖低层 | exec_policy.rs:668-683 | Phase 0 加载 |
| ignore_user_and_project_exec_policy_rules() 可跳过 User/Project layer | exec_policy.rs:670-677 | Phase 0 强制策略路径 |
| requirements().exec_policy 通过 merge_overlay 叠加 | exec_policy.rs:711-715 | Phase 0 强制 overlay |
| 网络规则使用同一套机制（append_network_rule_and_update） | exec_policy.rs:514-558 | Phase 4 网络固化 |
| BANNED_PREFIX_SUGGESTIONS 含 88 个禁止自动固化的前缀 | exec_policy.rs:57-146 | Phase 4 过滤 |

### L2 知识

策略持久化采用"读无锁、写串行、原子替换"的并发模型。ArcSwap 保证读路径的无锁高性能（每次工具调用都要查策略），Semaphore(1) 保证写操作的串行化（防止并发 append 导致规则丢失），store(Arc::new) 保证替换的原子性（读线程要么看到旧策略要么看到新策略，不会看到中间态）。

策略加载是多层叠加的：builtin < user < project < requirements(overlay)。`ignore_user_and_project_exec_policy_rules` 提供了一个"托管环境强制策略"开关——开启后 User 和 Project 层的规则被忽略，只有 builtin + requirements 生效。这是 enterprise/managed 部署的关键治理机制。

### L3 模式

**模式名称**：读无锁写串行的策略热更新（Lock-free Read + Serialized Write Strategy Hot-Reload）

**晋升论证 L2→L3**：ArcSwap+Semaphore 组合在 Rust 生态中是成熟模式（arc-swap crate 文档推荐用法），但 Codex 的特定贡献是将其与"spawn_blocking 落盘 + 幂等检查 + 内存热更新"组合成完整的持久化闭环。此模式可迁移到任何需要热更新的策略/配置系统（feature flag、rate limit 规则、安全策略）。

### L4 模型（晋升论证）

**模型名称**：分层策略叠加 + 运行时修正案持久化模型（Layered Policy Overlay with Runtime Amendment Persistence）

**晋升论证 L3→L4**：
- **预测力论证**：此模型可以预测系统行为：给定 config layer 组合和审批历史，可以精确推导当前生效的策略集和未来审批的结果。
- **解释力论证**：模型解释了为什么"用户在 project 层加了规则但不生效"——因为 `ignore_user_and_project_exec_policy_rules` 被开启（enterprise 环境）。
- **边界论证**：模型的适用边界是"策略规则是前缀匹配 + 决策树"的系统。对于需要复杂表达式的策略系统（如 OPA/Rego），此模型的修正案机制不直接适用。
- **多实例验证**：命令前缀规则和网络规则共享同一模型（append_amendment_and_update 和 append_network_rule_and_update 结构同构），验证了模型的泛化性。

**模型描述**：
策略系统由四层叠加构成（builtin → user → project → requirements overlay），运行时通过 ArcSwap 无锁读取、Semaphore 串行写入。审批通过的命令前缀/网络域名通过 blocking IO 落盘到 default.rules，经幂等检查后原子更新内存策略。88 个危险前缀被永久禁止自动固化。托管环境可通过 ignore_user_and_project_exec_policy_rules 忽略用户/项目层规则。

### 反例攻击（3 条）

1. **攻击：Semaphore(1) 成为写瓶颈，高并发审批场景下策略固化排队。**
   - 反驳：策略固化是低频操作（用户每次"允许并记住"才触发一次），Semaphore(1) 的等待时间 = spawn_blocking 的文件 append 时间（毫秒级），不是系统瓶颈。读路径完全无锁。→ **不成立（在预期负载下）**。

2. **攻击：ArcSwap 的 store(Arc::new) 不是持久化原子的——落盘成功但 store 前 crash，重启后内存策略与磁盘不一致。**
   - 反驳：确实存在这个窗口。但重启时会重新从磁盘 load_exec_policy（exec_policy.rs:303-309），所以最终一致性由启动时的全量加载保证。crash 窗口内的新规则在重启后从磁盘恢复。→ **最终一致，非强一致，但可接受**。

3. **攻击：requirements overlay 的 merge_overlay 可能与 user/project 规则产生冲突，且没有冲突解决机制的文档。**
   - 反驳：merge_overlay 的语义是"overlay 覆盖"（exec_policy.rs:715），即 requirements 层的同前缀规则优先级更高。这是标准的层叠策略语义。但确实没有显式的冲突检测日志——如果 user 层有 `allow python` 而 requirements 有 `forbid python`，后者静默覆盖前者，用户可能不知道。→ **部分成立：缺少冲突可见性**。

### 降级记录

- **尝试 L5（方法论）**：曾考虑将"分层叠加 + 运行时固化"提升为"自适应策略系统方法论"，但发现缺少"策略效果评估→自动调优"的闭环（Codex 没有根据审批拒绝率自动调整策略的机制），不构成完整方法论。→ **停留在 L4 Model**。

---

## KO-3：Guardian 独立审查 + fail-closed + 熔断器模型

**KO ID**: ko-guardian-independent-review-failclosed-circuitbreaker
**类型**: Model (L4)
**epistemic_status**: Validated Model（代码直接验证，含错误路径分析）
**flow_traceability**: Guardian Review Flow 全链路

### L1 事实

| 事实 | 锚点 | Flow Edge |
|------|------|-----------|
| Guardian 路由条件：approval_policy ∈ {OnRequest, Granular} ∧ approvals_reviewer == AutoReview | review.rs:209-217 | Guardian Flow 路由 |
| Full Access 短路：has_full_access → 直接 Approved | review.rs:330-343 | Guardian Flow 短路 |
| Guardian 运行在独立 review session，approval_policy=never，不继承 exec-policy 规则 | review.rs:977-990 | Guardian Flow 隔离 |
| GUARDIAN_REVIEW_TIMEOUT = 90s | mod.rs:62 | Guardian Flow 超时 |
| GUARDIAN_REVIEW_MAX_ATTEMPTS = 3 | review.rs:83 | Guardian Flow 重试 |
| 仅 transient 错误重试（ServerOverloaded/HttpConnectionFailed/ResponseStreamConnectionFailed/InternalServerError/ResponseStreamDisconnected/Parse） | review.rs:1145-1161 | Guardian Flow 重试策略 |
| Timeout → ReviewDecision::TimedOut（不执行） | review.rs:616 | fail-closed |
| PromptBuild/Session/Parse 错误 → 构造 risk=High, outcome=Deny 的 assessment | review.rs:655-689 | fail-closed |
| 熔断器阈值 Standard：连续 3 次 ∨ 最近 50 次中 10 次 | mod.rs:64-68 | 熔断器 |
| 熔断器阈值 Cyber：连续 1 次 ∨ 最近 50 次中 1 次 | mod.rs:64-67 | 熔断器（Cyber 更敏感） |
| 熔断器触发 → InterruptTurn + GuardianWarning | review.rs:269-307 | 熔断器动作 |
| 上下文预算 5 层：20K message / 10K tool / 5K per-message / 1K per-tool / 40 entries | mod.rs:71-77 | Guardian Flow 输入限制 |

### L2 知识

Guardian 是一个"用 AI 审批 AI"的子系统，其核心设计原则是**隔离 + fail-closed + 熔断**。隔离体现在：独立 session（approval_policy=never，防止 Guardian 自己触发审批循环）、不继承父 session 的 exec-policy 规则（防止父 session 的免审规则影响 Guardian 判断）、只读沙箱（防止 Guardian  mutate 状态）。fail-closed 体现在：超时→TimedOut（不执行）、解析/会话/prompt 错误→Deny（默认拒绝）。熔断体现在：连续/近期拒绝超过阈值→中断整个 turn，防止 Guardian 持续拒绝导致 agent 反复尝试消耗资源。

Cyber 模型的熔断器阈值更敏感（1 次 vs 3 次），因为 cyber 专用模型的拒绝意味着更高的安全风险。

### L3 模式

**模式名称**：隔离审查器 + fail-closed + 滑动窗口熔断（Isolated Reviewer with Fail-Closed and Sliding-Window Circuit Breaker）

**晋升论证 L2→L3**：此模式可迁移到任何"自动审批器"场景：CI/CD 自动合并审批、云资源自动审批、代码自动审查合并。隔离（独立运行环境）、fail-closed（出错默认拒绝）、熔断（连续失败时中断）是自动审批系统的三个通用安全属性。

### L4 模型（晋升论证）

**模型名称**：AI 审批器可靠性模型（AI Approver Reliability Model）

**晋升论证 L3→L4**：
- **预测力论证**：模型可以预测给定错误类型下的系统行为（transient→retry up to 3，non-transient→immediate deny，timeout→TimedOut）。
- **解释力论证**：模型解释了为什么"Guardian 偶尔超时后命令不执行"——因为 fail-closed 设计，超时等同于拒绝。
- **边界论证**：模型假设审批器是无状态的（每次审查独立），对于有状态审批器（如累积上下文的安全分析器）不直接适用。
- **多实例验证**：Guardian 的 fail-closed 模式与网络代理的 `session 释放→ask("not_allowed")`（保守降级）同构，验证了"不可用时保守降级"原则的泛化性。

**模型描述**：
AI 审批器运行在隔离环境中（独立 session、无继承策略、只读沙箱），输入受 5 层预算限制。审查在 90s 内完成，最多重试 3 次（仅 transient 错误）。超时→TimedOut，非 transient 错误→Deny（fail-closed）。审批结果触发熔断器更新：Standard 模型连续 3 次或 50 次中 10 次拒绝→中断 turn；Cyber 模型阈值为 1 次。Full Access 短路所有审查。

### 反例攻击（3 条）

1. **攻击：Guardian 审查 prompt 被恶意构造的工具输出注入，导致 Guardian 错误地 Allow 危险命令。**
   - 反驳：Guardian 的输入经过 5 层预算截断（mod.rs:71-77），单条 tool result 最多 1K tokens，限制了注入载荷大小。但 prompt injection 无法通过预算完全防御——1K tokens 足够构造有效的注入。Guardian 依赖模型自身的抗注入能力。→ **部分缓解，非完全防御（这是所有 LLM 审批器的固有局限）**。

2. **攻击：熔断器触发后，agent 可以通过"等待 turn 结束后新 turn"重置熔断器，继续尝试危险命令。**
   - 反驳：熔断器是 per-turn 的（mod.rs:175-179 的 GuardianRejectionCircuitBreakerTurn 以 turn_id 为 key），新 turn 确实会重置。但每次新 turn 的第一次尝试仍需经过 Guardian 审查，且 Guardian 的判断是基于命令内容的，不依赖历史——同一个危险命令在新 turn 仍会被拒绝。熔断器的目的是防止"同一 turn 内反复尝试消耗 token"，不是防止跨 turn 尝试。→ **不成立（熔断器的设计意图就是 per-turn 限流）**。

3. **攻击：Full Access 短路被滥用——如果攻击者能让 session 进入 Full Access 状态，Guardian 完全失效。**
   - 反驳：Full Access 的条件是 `approval_policy==Never ∧ 无沙箱限制`（has_full_access 的实现，review.rs:330-333），这是用户在配置中显式选择的最高权限模式，不是运行时可被诱导进入的状态。且 Full Access 下所有命令都直接执行，本来就不经过任何审批（不只是 Guardian）。→ **不成立（Full Access 是显式配置，不是漏洞）**。

### 降级记录

- **尝试 L5（方法论）**：曾考虑将"隔离+fail-closed+熔断"提升为"AI 安全审批方法论"，但发现缺少"审批器自身的定期审计/校准"环节（Codex 没有 Guardian 准确率的持续评估机制），不构成完整方法论。→ **停留在 L4 Model**。

---

## KO-4：网络代理回调治理模式

**KO ID**: ko-network-proxy-callback-governance
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（代码直接验证）
**flow_traceability**: Network Approval Flow 全链路

### L1 事实

| 事实 | 锚点 | Flow Edge |
|------|------|-----------|
| build_network_policy_decider 通过 Weak<Session> 持有 session 引用 | network_approval.rs:1014-1030 | Network Flow 回调构造 |
| session 已释放时返回 NetworkDecision::ask("not_allowed") | network_approval.rs:1022-1024 | Network Flow 保守降级 |
| begin_network_approval 构建 execution-scoped proxy，注入 fallback_policy_decider | network_approval.rs:1032-1158 | Network Flow 代理构建 |
| 网络规则持久化使用 append_network_rule_and_update → blocking_append_network_rule | exec_policy.rs:514-558 | Network Flow 固化 |
| 被拒绝网络请求通过 build_blocked_request_observer 记录 | network_approval.rs:1002-1012 | Network Flow 审计 |
| denied_network_policy_message 区分 5 种拒绝原因 | network_policy_decision.rs:46-72 | Network Flow 用户反馈 |

### L2 知识

网络治理采用"代理层拦截 + 运行时回调"模式：命令执行时的网络请求经过 codex-network-proxy，allowlist 命中则放行，miss 时通过 NetworkPolicyDecider trait 回调 core 决策。回调通过 Weak<Session> 持有 session，避免循环引用；session 释放时保守降级为 ask("not_allowed")。审批通过的网络域名通过与命令前缀规则相同的持久化机制写入 default.rules。

### L3 模式（晋升论证）

**模式名称**：代理层回调 + Weak ref 生命周期管理（Proxy-Level Callback with Weak-Reference Lifecycle）

**晋升论证 L2→L3**：
- **可迁移性论证**：此模式适用于任何需要运行时动态决策的代理/中间件：API 网关的 ext_authz、service mesh 的 sidecar 代理、反向代理的动态路由。Weak ref 是 Rust 特定实现，但"回调持有方可能已释放，需保守降级"的原则是通用的。
- **跨实例验证**：网络回调和 Guardian 审查都采用"不可用时保守降级/拒绝"原则（网络→ask，Guardian→TimedOut），验证了"不可用即保守"的设计一致性。

**模式描述**：在代理层拦截请求，allowlist 外的请求回调到控制面决策。控制面引用通过 Weak 持有，避免循环引用。控制面不可用时保守降级（ask/deny）。决策结果可持久化为 allowlist 规则。

### 反例攻击（3 条）

1. **攻击：session 释放后回调返回 ask("not_allowed")，但代理可能缓存了之前的 Allow 决策，导致 session 释放后仍放行。**
   - 反驳：NetworkApprovalService 的 register_call 跟踪活跃调用（network_approval.rs:1138-1151），session 释放时关联的 ActiveNetworkApproval 被取消（cancellation_token），代理的 execution_proxy 随 session 销毁。→ **不成立（代理生命周期与 session 绑定）**。

2. **攻击：网络域名固化为 Allow 后，子域名或不同端口的请求被错误放行。**
   - 反驳：网络规则的匹配粒度是 host + protocol（NetworkApprovalProtocol: Http/Https/Socks5Tcp/Socks5Udp，network_policy_decision.rs:79-84），不是子域名通配。`allow api.example.com:443` 不会放行 `api.example.com:8080`。但如果规则是 `allow example.com`，是否匹配 `api.example.com` 取决于 codex-execpolicy 的匹配语义，代码中未显式说明。→ **需确认匹配语义，可能存在边界模糊**。

3. **攻击：攻击者通过 DNS rebinding 让 allowlist 中的域名在首次解析后指向恶意 IP。**
   - 反驳：这是网络层攻击，超出 Codex 应用层治理范围。Codex 的网络代理在 TCP 层拦截，DNS 解析由代理或系统完成。如果代理在每次请求时重新解析 DNS，rebinding 可能绕过；如果代理缓存 DNS，则有 TTL 内的风险窗口。→ **超出 Codex 治理边界，属于基础设施安全问题**。

### 降级记录

- **尝试 L4（模型）**：曾尝试将此模式抽象为"动态决策代理模型"，但发现网络回调的决策逻辑（handle_inline_policy_request）在 NetworkApprovalService 中，其内部状态机未在本次考古中完全展开，缺少足够证据支撑模型级预测力。→ **降级为 L3 Pattern**。

---

## KO-5：上下文注入硬预算治理模式

**KO ID**: ko-context-injection-hard-budget
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（AGENTS.md 显式规则 + Guardian 实现验证）
**flow_traceability**: Context Flow（AGENTS.md 规则 → core/context/ 实现 → Guardian 预算）

### L1 事实

| 事实 | 锚点 | Flow Edge |
|------|------|-----------|
| AGENTS.md 规定 6 条上下文硬约束：no history rewrite / avoid cache misses / no unbounded items / no items >10K / >1K items P0 review / 必须实现 ContextualUserFragment trait | AGENTS.md:91-100 | Context Flow 规则源 |
| Guardian 上下文 5 层预算：20K message / 10K tool / 5K per-message / 1K per-tool / 40 entries | guardian/mod.rs:71-77 | Context Flow 实例化 |
| core/context/ 下有 50+ 个 context fragment struct | 目录枚举 | Context Flow 实现 |
| RolloutBudget 提醒通过 context/rollout_budget.rs 注入 | 文件存在 | Context Flow 实例 |
| Guardian policy 通过 context/guardian_policy.rs 注入 | 文件存在 | Context Flow 实例 |

### L2 知识

上下文治理采用"规则文档 + 类型安全 trait + 硬预算常量"三层机制。AGENTS.md 是规则源（6 条硬约束），ContextualUserFragment trait 是类型安全机制（所有注入项必须实现），各子系统的预算常量是执行机制（如 Guardian 的 5 层限制）。核心原则是：**上下文不是无限的笔记本，而是有严格预算的通信通道**。

### L3 模式（晋升论证）

**模式名称**：规则+trait+预算的三层上下文治理（Rule + Trait + Budget Three-Layer Context Governance）

**晋升论证 L2→L3**：
- **可迁移性论证**：任何 LLM 应用都面临上下文膨胀问题，此模式的三层机制（规则文档定原则、trait 定接口、常量定预算）可直接迁移。
- **跨实例验证**：Guardian（5 层预算）和主 context（AGENTS.md 6 条规则）是同一模式的两个实例，验证了模式的一致性。

**模式描述**：上下文注入系统由三层构成：规则层（文档化的硬约束，如单条不超过 10K、>1K 需审查）、类型层（统一 trait 确保所有注入项有结构化接口）、预算层（各子系统的具体 token 预算常量）。注入项的大小和频率受预算约束，超预算项需额外审查。

### 反例攻击（3 条）

1. **攻击：AGENTS.md 的 6 条规则是"建议"而非强制，开发者可以不遵守。**
   - 反驳：第 5 条明确">1K tokens 的新 context item 标记为 P0，需要额外人工审查"，这在 PR review 流程中是强制的（CI 或 reviewer 检查）。第 6 条的 trait 要求在编译期强制（不实现 trait 无法注入）。→ **部分强制：trait 是编译期强制，预算是 review 期强制**。

2. **攻击：Guardian 的 5 层预算是硬编码常量，无法根据模型上下文窗口动态调整。**
   - 反驳：确实是硬编码（mod.rs:71-77 的 const）。但 Guardian 的输入是"压缩后的 transcript"，其预算设计是基于"足够做出审批决策"的最小信息量，不是基于模型上下文窗口。对于上下文窗口极小的模型，Guardian review session 可能需要自己的压缩，但这是 Guardian session 内部的问题。→ **成立：预算是静态的，未适配模型上下文窗口**。

3. **攻击：context fragment 可以通过"分散注入"绕过单条 10K 限制——将一个 30K 的信息拆成 3 个 9.9K 的 fragment。**
   - 反驳：AGENTS.md 第 3 条"no unbounded items"和第 2 条"avoid frequent changes that cause cache misses"联合限制了分散注入——频繁注入小 fragment 会导致缓存失效，且 Guardian 的总预算（20K message + 10K tool）是总量限制，分散注入仍受总量约束。→ **总量预算提供了兜底防御**。

### 降级记录

- **不尝试 L4+**：此模式是工程实践级别的设计模式，不具备预测力或解释力的模型级抽象。→ **停留在 L3 Pattern**。

---

## KO-6：跨 agent 树共享资源预算模式

**KO ID**: ko-cross-agent-shared-resource-budget
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（代码直接验证）
**flow_traceability**: RolloutBudget Flow 全链路

### L1 事实

| 事实 | 锚点 | Flow Edge |
|------|------|-----------|
| AgentControl.rollout_budget: Arc<RolloutBudget> 共享给整个 agent 树 | agent/control.rs:131 | RolloutBudget Flow 共享 |
| RolloutBudget 使用 OnceLock<Mutex<RolloutBudgetState>> 延迟初始化 | rollout_budget.rs:18-20 | RolloutBudget Flow 初始化 |
| 加权计费：output_tokens * sampling_weight + non_cached_input * prefill_weight | rollout_budget.rs:60-62 | RolloutBudget Flow 计费 |
| 优先使用服务端 codex_rollout_budget_units 精确计费 | rollout_budget.rs:50-58 | RolloutBudget Flow 计费 |
| 提醒机制：阈值穿越 + 每线程 delivery 去重 | rollout_budget.rs:67-91 | RolloutBudget Flow 提醒 |
| rearm_reminder 清除线程 delivery 记录，强制重申 | rollout_budget.rs:112-118 | RolloutBudget Flow 重申 |

### L2 知识

RolloutBudget 是一个跨 agent 树的共享 token 预算系统。root agent 创建时初始化（AgentControl::new 中 configure），通过 Arc 共享给所有 spawn 的 subagent。计费采用加权模型（output 权重通常高于 input，因为 output 更贵），优先使用服务端返回的精确 units。预算耗尽后 latch（持续返回 exhausted）。提醒机制采用"阈值穿越计数 + 每线程去重"，确保每个线程在穿越新阈值时收到提醒，但不重复提醒同一阈值。

### L3 模式（晋升论证）

**模式名称**：层级执行体共享配额 + 加权计费 + 阈值提醒（Hierarchical Executor Shared Quota with Weighted Billing and Threshold Alerting）

**晋升论证 L2→L3**：
- **可迁移性论证**：此模式适用于任何有层级执行体的资源配额系统：K8s namespace ResourceQuota、AWS OU SCP、组织级 API 速率限制。Arc 是 Rust 特定实现，但"共享配额 + 加权计费 + 阈值提醒"是通用模式。
- **跨实例验证**：RolloutBudget（token 预算）和 Guardian 熔断器（拒绝率预算）都是"共享计数器 + 阈值触发动作"模式的实例。

**模式描述**：在层级执行体树的根节点创建共享资源预算，通过引用计数共享给所有子节点。资源消耗按权重计费（不同资源类型权重不同），优先使用精确计量。预算耗尽后 latch。阈值穿越时触发提醒，每节点去重，支持手动 rearm。

### 反例攻击（3 条）

1. **攻击：subagent 可以通过"快速消耗预算"让 root agent 后续无预算可用（拒绝服务）。**
   - 反驳：这是共享配额系统的固有特性——配额是共享的，子节点确实可以消耗父节点的预算。但这是设计意图（整个 agent 树有统一预算），不是漏洞。如果需要隔离，应使用 per-agent 配额（Codex 未提供此选项）。→ **设计意图，非漏洞**。

2. **攻击：Mutex<RolloutBudgetState> 在高并发 agent 场景下成为锁竞争瓶颈。**
   - 反驳：record_usage 仅在每次 response.completed 时调用（低频，每次模型响应一次），Mutex 持有时间极短（f64 加法 + 比较），不是瓶颈。且 OnceLock 确保只初始化一次。→ **不成立（预期负载下无竞争问题）**。

3. **攻击：服务端返回的 codex_rollout_budget_units 可以被篡改（如果中间人攻击），导致计费不准确。**
   - 反驳：codex_rollout_budget_units 来自 Responses API 的 response.completed 事件，通过 HTTPS 传输，中间人攻击需要 TLS 降级或根证书篡改，超出应用层治理范围。且本地 fallback 计费（output*weight + input*weight）在服务端 units 缺失时提供兜底。→ **超出应用层边界**。

### 降级记录

- **不尝试 L4+**：此模式是资源配额的工程实现，不具备模型级预测力。→ **停留在 L3 Pattern**。

---

## KO-7：Weak ref 断环 + PermissionProfile SSOT 知识

**KO ID**: ko-weak-ref-cycle-breaking-permissionprofile-ssot
**类型**: Knowledge (L2)
**epistemic_status**: Validated Knowledge（代码直接验证）
**flow_traceability**: Policy Flow Phase 0（PermissionProfile 加载）

### L1 事实

| 事实 | 锚点 |
|------|------|
| AgentControl.manager: Weak<ThreadManagerState>，注释说明避免 ThreadManagerState→CodexThread→Session→SessionServices→ThreadManagerState 循环 | agent/control.rs:121-124 |
| network_policy_decider_session: Arc<RwLock<Weak<Session>>>，session 释放时 upgrade() 失败→ask("not_allowed") | network_approval.rs:1016, 1022-1024 |
| SessionConfiguration.permission_profile_state 是 SSOT，注释要求三者同步（constrained profile + active profile id + workspace roots） | session/session.rs:96-99 |
| set_permission_profile_projection 在更新时保持三者同步并保留 deny-read | session/session.rs:504-538 |

### L2 知识

Codex 的对象图存在天然的循环引用风险：ThreadManagerState 持有 CodexThread，CodexThread 持有 Session，Session 通过 SessionServices 持有 AgentControl，而 AgentControl 需要回调 ThreadManagerState。解决方案是 AgentControl 通过 Weak<ThreadManagerState> 持有全局状态，upgrade() 失败时返回错误。同样的模式用于网络代理回调（Weak<Session>）。

PermissionProfileState 是 session 级权限的单一事实源，将三个相关字段（constrained profile、active profile id、profile-defined workspace roots）封装在一个 state 中，通过方法更新保证一致性。更新时保留 deny-read 限制（preserve_deny_read_restrictions_from），防止 profile 切换时意外放宽读取限制。

### 不晋升 L3 的原因

Weak ref 断环是 Rust 语言级别的通用模式（标准做法），不是 Codex 特有的可迁移设计模式。PermissionProfile SSOT 是单一职责原则的直接应用，也不具备模式级别的新颖性。→ **停留在 L2 Knowledge**。

---

## KO 汇总与覆盖矩阵

| KO ID | 类型 | 层级 | 反例数 | 核心锚点 | v1 覆盖 |
|-------|------|------|--------|---------|---------|
| KO-1 | 三态审批+修正案 | L3 Pattern | 3 | sandboxing.rs:152-230, exec_policy.rs:394-461 | 部分 |
| KO-2 | 策略持久化热更新 | L4 Model | 3 | exec_policy.rs:276-558, 662-716 | ❌ 新增 |
| KO-3 | Guardian 独立审查+熔断 | L4 Model | 3 | guardian/mod.rs:62-246, review.rs:321-689 | ❌ 新增 |
| KO-4 | 网络代理回调治理 | L3 Pattern | 3 | network_approval.rs:1002-1158, network_policy_decision.rs | ❌ 新增 |
| KO-5 | 上下文注入硬预算 | L3 Pattern | 3 | AGENTS.md:91-100, guardian/mod.rs:71-77 | ❌ 新增 |
| KO-6 | 跨 agent 共享预算 | L3 Pattern | 3 | rollout_budget.rs:11-127, agent/control.rs:131 | ❌ 新增 |
| KO-7 | Weak ref 断环+SSOT | L2 Knowledge | - | agent/control.rs:121-124, session/session.rs:96-99 | 部分 |

**总计**：7 个 KO（2 个 L4 Model + 4 个 L3 Pattern + 1 个 L2 Knowledge），其中 5 个为 v1 遗漏新增，2 个为 v1 部分覆盖的补全。所有 L3+ KO 均含 ≥3 条反例攻击和晋升论证块，2 个 L4 尝试 L5 后降级并记录。

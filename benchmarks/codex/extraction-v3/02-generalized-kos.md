# Generalized Knowledge Layer（广义知识层 · 窄尖顶）

> **v3 三层架构的窄尖顶。** L3 Pattern / L4 Cognitive Model / L5 Methodology——可迁移的跨项目认知。
> **每个 KO 必须能回溯到 Engineering Knowledge 底座**（`derivation.facts` 指回 EK 编号），禁止无源 KO。
> 升维是特例不是默认——仅 8 个 KO 从 52 条工程知识中晋升而来。
> 编号 KO-01 ~ KO-08。

---

## KO-01：三态审批 + 修正案携带模式

**KO ID**: ko-tri-state-approval-with-amendment
**knowledge_layer**: generalized
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（代码直接验证，多文件交叉确认）
**value**: A

### derivation.facts（回溯 Engineering 底座）

- EK-01（ExecApprovalRequirement 三态枚举）
- EK-18（修正案仅在 Honor 模式生成）
- EK-21（Granular 静默 Forbidden）
- EK-52（Forbidden 不产生修正案——拒绝不可学习）

### L1 事实

| 事实 | 锚点 | 回溯 EK |
|------|------|---------|
| ExecApprovalRequirement 是三态：Skip{bypass_sandbox, amendment}、NeedsApproval{reason, amendment}、Forbidden{reason} | sandboxing.rs:152-171 | EK-01 |
| Skip 和 NeedsApproval 都携带 proposed_execpolicy_amendment，Forbidden 不携带 | sandboxing.rs:174-186 | EK-52 |
| AskForApproval 是四态（Never/OnRequest/Granular/UnlessTrusted），与沙箱策略组合映射到三态 | sandboxing.rs:194-230 | EK-38 |
| Granular + sandbox_approval=false → 静默 Forbidden，不提示用户 | sandboxing.rs:209-218 | EK-21 |

### L2 知识

审批结果不是简单的"允许/拒绝"二元决策，而是携带元数据的三态结构。核心洞察：**审批决策本身就是策略学习的输入**——Skip 和 NeedsApproval 都携带"是否应该把这个命令固化为免审规则"的修正案建议，Forbidden 不产生修正案（拒绝不可学习）。这形成了"执行→审批→固化→未来免审"的闭环。

### L3 模式（晋升论证 L2→L3）

**模式名称**：三态审批 + 修正案携带（Tri-State Approval with Amendment Carrying）

- **可迁移性**：不依赖 Codex 技术栈。任何需要"运行时审批 + 审批结果可学习固化"的系统都适用：CI/CD 审批门、云资源申请审批、API 速率限制豁免审批。
- **跨实例验证**：三态结构在多个工具运行时统一使用（shell、apply_patch、unified_exec），不是单个工具特例。
- **不变量论证**：Forbidden 不携带修正案是安全不变量——拒绝的命令永远不会被自动固化为免审规则。

**模式描述**：审批系统的输出应是三态（直接执行/需要审批/禁止），且前两态应携带"是否建议固化为免审规则"的修正案。禁止态不携带修正案。修正案在持久化前经过危险前缀过滤、冲突检测和模拟验证三道关卡。

### 反例攻击（3 条）

1. **修正案可能被构造为"看似安全实际危险"的前缀** → BANNED_PREFIX_SUGGESTIONS（88 个）覆盖 shell 包装器和解释器 -c/-e 形式（EK-37），但自定义二进制危险子命令（如 `docker run --privileged`）不在列表，依赖用户审批判断。**部分缓解**。
2. **NeedsApproval 被拒绝后修正案仍可能被错误持久化** → append_amendment_and_update 仅在审批通过后调用（EK-02），拒绝路径不触发固化。**不成立**。
3. **通过"先执行安全命令获得 Skip+修正案固化，再执行危险变体"绕过** → 修正案固化精确前缀匹配，`python script.py` 固化后不豁免 `python -c "..."`（在 BANNED 列表）。但 `python build.py` 固化后 `python build.py --deploy-production` 被豁免（EK-30）。**部分成立：前缀匹配粒度是固有局限**。

### 降级记录

- 尝试 L4（"审批即学习"认知模型）→ 固化逻辑与 ExecPolicyManager 强耦合，不具备跨系统模型级通用性。**降级为 L3 Pattern**。

---

## KO-02：分层策略叠加 + 运行时修正案持久化模型

**KO ID**: ko-policy-persistence-hot-reload
**knowledge_layer**: generalized
**类型**: Model (L4)
**epistemic_status**: Validated Model（代码直接验证，含并发安全分析）
**value**: A

### derivation.facts

- EK-02（ExecPolicyManager 持久化与热更新闭环）
- EK-09（ArcSwap 读无锁 + Semaphore 写串行）
- EK-12（spawn_blocking 落盘）
- EK-20（修正案三道过滤）
- EK-27（幂等检查）
- EK-28（crash 窗口最终一致性）
- EK-37（BANNED_PREFIX_SUGGESTIONS 88 个）
- EK-40（多层 config stack）
- EK-49（网络规则与命令规则共享持久化）

### L1 事实

| 事实 | 锚点 | 回溯 EK |
|------|------|---------|
| ExecPolicyManager { policy: ArcSwap\<Policy\>, update_lock: Semaphore(1) } | exec_policy.rs:276-279 | EK-09 |
| 读路径 current() = load_full() 无锁；写路径 acquire→spawn_blocking→幂等→clone→store | exec_policy.rs:464-512 | EK-02, EK-12 |
| 策略文件 = codex_home/rules/default.rules | exec_policy.rs:867-869 | EK-42 |
| 多层加载 builtin→user→project，最后 merge_overlay(requirements) | exec_policy.rs:662-716 | EK-40 |
| ignore_user_and_project_exec_policy_rules 可跳过 User/Project 层 | exec_policy.rs:670-677 | EK-43 |
| 网络规则共享同一持久化机制 | exec_policy.rs:514-558 | EK-49 |
| BANNED_PREFIX_SUGGESTIONS 88 个危险前缀 | exec_policy.rs:57-146 | EK-37 |
| 三道过滤：BANNED / is_policy_match / simulate | exec_policy.rs:958-997 | EK-20 |

### L2 知识

策略持久化采用"读无锁、写串行、原子替换"的并发模型。策略加载是多层叠加的（builtin < user < project < requirements overlay），托管环境可通过 ignore 开关完全跳过用户/项目层规则。审批通过的命令前缀/网络域名通过 blocking IO 落盘，经幂等检查后原子更新内存策略。

### L3 模式

**模式名称**：读无锁写串行的策略热更新（Lock-free Read + Serialized Write Strategy Hot-Reload）

ArcSwap+Semaphore 是 Rust 成熟模式，Codex 的贡献是将其与 spawn_blocking 落盘 + 幂等检查 + 内存热更新组合成完整持久化闭环。可迁移到任何需要热更新的策略/配置系统（feature flag、rate limit 规则、安全策略）。

### L4 模型（晋升论证 L3→L4）

**模型名称**：分层策略叠加 + 运行时修正案持久化模型（Layered Policy Overlay with Runtime Amendment Persistence）

- **预测力**：给定 config layer 组合和审批历史，可精确推导当前生效策略集和未来审批结果。
- **解释力**：解释了"用户在 project 层加了规则但不生效"——因为 ignore_user_and_project 被开启（enterprise 环境），或 requirements overlay 静默覆盖（EK-29）。
- **边界**：适用于"策略规则是前缀匹配 + 决策树"的系统。对于需要复杂表达式的策略系统（OPA/Rego），修正案机制不直接适用。
- **多实例验证**：命令前缀规则和网络规则共享同一模型（append_amendment_and_update 和 append_network_rule_and_update 同构，EK-49）。

**模型描述**：策略系统由四层叠加构成（builtin→user→project→requirements overlay），运行时通过 ArcSwap 无锁读取、Semaphore 串行写入。审批通过的命令前缀/网络域名通过 blocking IO 落盘到 default.rules，经幂等检查后原子更新内存策略。88 个危险前缀被永久禁止自动固化。托管环境可通过 ignore 开关忽略用户/项目层规则。

### 反例攻击（3 条）

1. **Semaphore(1) 成为写瓶颈** → 策略固化是低频操作（用户每次"允许并记住"才触发），等待时间 = 文件 append 毫秒级，非瓶颈。读路径完全无锁。**不成立（预期负载下）**。
2. **落盘成功但 store 前 crash，内存与磁盘不一致** → 确实存在窗口（EK-28）。但重启时从磁盘全量加载保证最终一致。crash 窗口内新规则在重启后恢复。**最终一致，非强一致，可接受**。
3. **requirements overlay 与 user/project 规则冲突无可见性** → merge_overlay 语义是"覆盖"（EK-29），requirements 同前缀优先级更高。但无显式冲突检测日志——user 层 allow python 而 requirements forbid python，后者静默覆盖。**部分成立：缺少冲突可见性**。

### 降级记录

- 尝试 L5（"自适应策略系统方法论"）→ 缺少"策略效果评估→自动调优"闭环（Codex 没有根据审批拒绝率自动调整策略的机制）。**停留在 L4 Model**。

---

## KO-03：AI 审批器可靠性模型——隔离 + fail-closed + 熔断

**KO ID**: ko-guardian-independent-review-failclosed-circuitbreaker
**knowledge_layer**: generalized
**类型**: Model (L4)
**epistemic_status**: Validated Model（代码直接验证，含错误路径分析）
**value**: A

### derivation.facts

- EK-03（Guardian 自动审批审查器）
- EK-14（trunk 复用 + ephemeral fork）
- EK-22（Guardian 5 层上下文预算）
- EK-25（fail-closed 错误处理）
- EK-34（record_non_denial 重置计数）
- EK-35（Cyber 模型更敏感阈值）
- EK-39（超时与重试配置）
- EK-48（系统错误不计入熔断器）
- EK-50（不继承父 exec-policy 规则）

### L1 事实

| 事实 | 锚点 | 回溯 EK |
|------|------|---------|
| 路由条件：OnRequest\|Granular + AutoReview；Never/UnlessTrusted 不走 Guardian | review.rs:209-217 | EK-47 |
| Full Access 短路直接 Approved | review.rs:330-343 | EK-32 |
| 独立 review session：approval_policy=never、不继承父 exec-policy、只读沙箱 | review.rs:977-990 | EK-50 |
| trunk idle 复用（保 cache key），trunk busy fork ephemeral（并行不阻塞） | review.rs:977-990 | EK-14 |
| 90s 超时，最多 3 次重试（仅 transient），90s deadline 内 | mod.rs:62, review.rs:83, 1098 | EK-39 |
| Timeout→TimedOut；PromptBuild/Session/Parse→Deny（fail-closed） | review.rs:616, 655-689 | EK-25 |
| 熔断器 Standard：连续 3 次 ∨ 最近 50 次中 10 次；Cyber：1/1 | mod.rs:64-68 | EK-35 |
| 5 层上下文预算：20K/10K/5K/1K/40 entries | mod.rs:71-77 | EK-22 |
| 系统错误（PromptBuild/Session/Parse）导致的 Deny 不计入熔断器 | review.rs:655-689 | EK-48 |

### L2 知识

Guardian 是"用 AI 审批 AI"的子系统，核心设计原则是**隔离 + fail-closed + 熔断**。隔离体现在独立 session（approval_policy=never 防审批循环、不继承父 exec-policy 防免审规则影响判断、只读沙箱防 mutate 状态）。fail-closed 体现在超时→TimedOut、解析/会话/prompt 错误→Deny。熔断体现在连续/近期拒绝超阈值→中断 turn，防止持续拒绝导致 agent 反复尝试消耗资源。

### L3 模式

**模式名称**：隔离审查器 + fail-closed + 滑动窗口熔断（Isolated Reviewer with Fail-Closed and Sliding-Window Circuit Breaker）

可迁移到任何"自动审批器"场景：CI/CD 自动合并审批、云资源自动审批、代码自动审查合并。隔离（独立运行环境）、fail-closed（出错默认拒绝）、熔断（连续失败时中断）是自动审批系统的三个通用安全属性。

### L4 模型（晋升论证 L3→L4）

**模型名称**：AI 审批器可靠性模型（AI Approver Reliability Model）

- **预测力**：给定错误类型可预测系统行为（transient→retry up to 3，non-transient→immediate deny，timeout→TimedOut）。
- **解释力**：解释了"Guardian 偶尔超时后命令不执行"——因为 fail-closed 设计，超时等同于拒绝。
- **边界**：假设审批器是无状态的（每次审查独立），对于有状态审批器（如累积上下文的安全分析器）不直接适用。
- **多实例验证**：Guardian fail-closed 与网络代理 session 释放→ask("not_allowed")（EK-19）同构，验证了"不可用时保守降级"原则的泛化性。

**模型描述**：AI 审批器运行在隔离环境中（独立 session、无继承策略、只读沙箱），输入受 5 层预算限制。审查在 90s 内完成，最多重试 3 次（仅 transient 错误）。超时→TimedOut，非 transient 错误→Deny（fail-closed）。审批结果触发熔断器更新：Standard 连续 3 次或 50 次中 10 次拒绝→中断 turn；Cyber 阈值为 1 次。系统错误导致的 Deny 不计入熔断器。Full Access 短路所有审查。

### 反例攻击（3 条）

1. **Guardian prompt 被恶意工具输出注入导致错误 Allow** → 输入经 5 层预算截断（EK-22），单条 tool result 最多 1K tokens，限制注入载荷。但 1K tokens 足够构造有效注入，Guardian 依赖模型自身抗注入能力。**部分缓解，非完全防御（所有 LLM 审批器固有局限）**。
2. **熔断器触发后 agent 通过"新 turn"重置继续尝试** → 熔断器是 per-turn 的（EK-34），新 turn 确实重置。但每次新 turn 第一次尝试仍需 Guardian 审查，同一危险命令仍被拒绝。熔断器目的是防"同一 turn 内反复尝试消耗 token"，不是防跨 turn。**不成立（设计意图就是 per-turn 限流）**。
3. **Full Access 短路被滥用** → Full Access 条件是 approval_policy==Never ∧ 无沙箱限制（EK-32），是用户显式选择的最高权限模式，不是运行时可诱导状态。且 Full Access 下所有命令直接执行，本来就不经过任何审批。**不成立**。

### 降级记录

- 尝试 L5（"AI 安全审批方法论"）→ 缺少"审批器自身定期审计/校准"环节（Codex 没有 Guardian 准确率持续评估机制）。**停留在 L4 Model**。

---

## KO-04：代理层回调 + Weak ref 生命周期管理模式

**KO ID**: ko-network-proxy-callback-governance
**knowledge_layer**: generalized
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（代码直接验证）
**value**: A

### derivation.facts

- EK-04（网络审批闭环）
- EK-10（Weak\<Session\> 代理回调）
- EK-15（ActiveNetworkApproval cancellation_token）
- EK-19（Session 销毁保守降级 ask("not_allowed")）
- EK-31（网络规则匹配语义边界）
- EK-45（仅 managed_network 激活）

### L1 事实

| 事实 | 锚点 | 回溯 EK |
|------|------|---------|
| build_network_policy_decider 通过 Weak\<Session\> 持有 session 引用 | network_approval.rs:1014-1030 | EK-10 |
| session 已释放时返回 NetworkDecision::ask("not_allowed") | network_approval.rs:1022-1024 | EK-19 |
| begin_network_approval 构建 execution-scoped proxy，注入 fallback_policy_decider | network_approval.rs:1032-1158 | EK-04 |
| ActiveNetworkApproval { registration_id, cancellation_token, execution_proxy } | network_approval.rs:1153-1157 | EK-15 |
| 被拒绝请求通过 build_blocked_request_observer 记录 | network_approval.rs:1002-1012 | EK-04 |
| 网络规则持久化使用 append_network_rule_and_update | exec_policy.rs:514-558 | EK-49 |

### L2 知识

网络治理采用"代理层拦截 + 运行时回调"模式：命令执行时的网络请求经过 codex-network-proxy，allowlist 命中放行，miss 时通过 NetworkPolicyDecider trait 回调 core 决策。回调通过 Weak\<Session\> 持有 session 避免循环引用；session 释放时保守降级为 ask("not_allowed")。审批通过的网络域名通过与命令前缀规则相同的持久化机制写入 default.rules。

### L3 模式（晋升论证 L2→L3）

**模式名称**：代理层回调 + Weak ref 生命周期管理（Proxy-Level Callback with Weak-Reference Lifecycle）

- **可迁移性**：适用于任何需要运行时动态决策的代理/中间件：API 网关 ext_authz、service mesh sidecar 代理、反向代理动态路由。Weak ref 是 Rust 特定实现，但"回调持有方可能已释放，需保守降级"原则通用。
- **跨实例验证**：网络回调和 Guardian 审查都采用"不可用时保守降级/拒绝"原则（网络→ask，Guardian→TimedOut/Deny，EK-25），验证了设计一致性。

**模式描述**：在代理层拦截请求，allowlist 外的请求回调到控制面决策。控制面引用通过 Weak 持有避免循环引用。控制面不可用时保守降级（ask/deny）。决策结果可持久化为 allowlist 规则。代理生命周期与执行上下文绑定（cancellation_token），上下文结束时代理自动取消。

### 反例攻击（3 条）

1. **session 释放后代理缓存之前的 Allow 决策导致仍放行** → register_call 跟踪活跃调用（EK-04），session 释放时 ActiveNetworkApproval 被 cancellation_token 取消（EK-15），execution_proxy 随 session 销毁。**不成立（代理生命周期与 session 绑定）**。
2. **网络域名固化 Allow 后子域名或不同端口被错误放行** → 匹配粒度是 host + protocol（EK-31），不是子域名通配。但 `allow example.com` 是否匹配 `api.example.com` 取决于 codex-execpolicy 匹配语义，代码未显式说明。**需确认匹配语义，可能存在边界模糊**。
3. **DNS rebinding 让 allowlist 域名首次解析后指向恶意 IP** → 网络层攻击，超出 Codex 应用层治理范围。**超出治理边界，属基础设施安全问题**。

### 降级记录

- 尝试 L4（"动态决策代理模型"）→ 网络回调决策逻辑（handle_inline_policy_request）在 NetworkApprovalService 中，内部状态机未完全展开，缺少足够证据支撑模型级预测力。**降级为 L3 Pattern**。

---

## KO-05：规则 + trait + 预算三层上下文治理模式

**KO ID**: ko-context-injection-hard-budget
**knowledge_layer**: generalized
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（AGENTS.md 显式规则 + Guardian 实现验证）
**value**: B

### derivation.facts

- EK-08（上下文片段系统 50+ ContextualUserFragment）
- EK-22（Guardian 5 层上下文预算）
- EK-24（no history rewrite 增量构建）
- EK-36（>1K tokens P0 审查）
- EK-51（主上下文与 Guardian 上下文预算体系分离）

### L1 事实

| 事实 | 锚点 | 回溯 EK |
|------|------|---------|
| AGENTS.md 6 条硬约束：no rewrite / cache friendly / bounded / 10K cap / 1K P0 / trait | AGENTS.md:91-100 | EK-24, EK-36 |
| Guardian 5 层预算：20K message / 10K tool / 5K per-message / 1K per-tool / 40 entries | guardian/mod.rs:71-77 | EK-22 |
| core/context/ 下 50+ 个 context fragment struct，均实现 ContextualUserFragment trait | core/context/ 目录 | EK-08 |
| 主上下文与 Guardian 上下文预算体系分离 | AGENTS.md + mod.rs | EK-51 |

### L2 知识

上下文治理采用"规则文档 + 类型安全 trait + 硬预算常量"三层机制。AGENTS.md 是规则源（6 条硬约束），ContextualUserFragment trait 是类型安全机制（编译期强制），各子系统预算常量是执行机制（如 Guardian 5 层限制）。核心原则：**上下文不是无限的笔记本，而是有严格预算的通信通道**。

### L3 模式（晋升论证 L2→L3）

**模式名称**：规则 + trait + 预算三层上下文治理（Rule + Trait + Budget Three-Layer Context Governance）

- **可迁移性**：任何 LLM 应用都面临上下文膨胀问题，三层机制（规则文档定原则、trait 定接口、常量定预算）可直接迁移。
- **跨实例验证**：Guardian（5 层预算）和主 context（AGENTS.md 6 条规则）是同一模式的两个实例，验证了一致性。

**模式描述**：上下文注入系统由三层构成：规则层（文档化硬约束，如单条≤10K、>1K 需审查）、类型层（统一 trait 确保所有注入项有结构化接口）、预算层（各子系统具体 token 预算常量）。注入项的大小和频率受预算约束，超预算项需额外审查。不同子系统可有独立预算体系（如主会话 vs 审查器）。

### 反例攻击（3 条）

1. **AGENTS.md 6 条规则是"建议"非强制** → 第 5 条 >1K P0 审查在 PR review 流程中强制；第 6 条 trait 要求编译期强制（EK-08）。**部分强制：trait 编译期，预算 review 期**。
2. **Guardian 5 层预算是硬编码常量，无法根据模型上下文窗口动态调整** → 确实是硬编码（EK-22）。但 Guardian 输入是"压缩后的 transcript"，预算基于"足够做审批决策的最小信息量"，不是基于模型窗口。**成立：预算是静态的，未适配模型上下文窗口**。
3. **通过"分散注入"绕过单条 10K 限制** → AGENTS.md 第 3 条 no unbounded + 第 2 条 avoid cache misses 联合限制（EK-24），且 Guardian 总预算（20K+10K）是总量限制（EK-22）。**总量预算提供兜底防御**。

### 不尝试 L4+

此模式是工程实践级设计模式，不具备预测力或解释力的模型级抽象。**停留在 L3 Pattern**。

---

## KO-06：层级执行体共享配额 + 加权计费 + 阈值提醒模式

**KO ID**: ko-cross-agent-shared-resource-budget
**knowledge_layer**: generalized
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（代码直接验证）
**value**: B

### derivation.facts

- EK-05（RolloutBudget 跨 agent 树共享 token 预算）
- EK-13（OnceLock\<Mutex\> 延迟初始化）
- EK-23（加权 token 计费 output > input）
- EK-41（AgentExecutionLimiter 并发限制）
- EK-46（未配置时 no-op）

### L1 事实

| 事实 | 锚点 | 回溯 EK |
|------|------|---------|
| AgentControl.rollout_budget: Arc\<RolloutBudget\> 共享给整个 agent 树 | agent/control.rs:131 | EK-05 |
| OnceLock\<Mutex\<RolloutBudgetState\>\> 延迟初始化，未配置时 no-op | rollout_budget.rs:18-20, 33-44 | EK-13, EK-46 |
| 加权计费：output × sampling_weight + non_cached_input × prefill_weight | rollout_budget.rs:46-65 | EK-23 |
| 优先服务端 codex_rollout_budget_units 精确计费 | rollout_budget.rs:50-58 | EK-23 |
| 阈值穿越 + 每线程 delivery 去重提醒 | rollout_budget.rs:67-91 | EK-05 |
| rearm_reminder 清除记录强制重申 | rollout_budget.rs:112-118 | EK-05 |
| AgentExecutionLimiter 限制并发 agent 数（effective_agent_max_threads） | agent/control.rs:129, 169-173 | EK-41 |

### L2 知识

RolloutBudget 是跨 agent 树的共享 token 预算系统。root agent 创建时初始化，通过 Arc 共享给所有 subagent。计费采用加权模型（output 权重高于 input），优先服务端精确 units。预算耗尽后 latch（持续返回 exhausted）。提醒机制采用"阈值穿越计数 + 每线程去重"，确保每个线程穿越新阈值时收到提醒但不重复。与 AgentExecutionLimiter（并发数限制）互补——一个限制总 token 消耗，一个限制并发数。

### L3 模式（晋升论证 L2→L3）

**模式名称**：层级执行体共享配额 + 加权计费 + 阈值提醒（Hierarchical Executor Shared Quota with Weighted Billing and Threshold Alerting）

- **可迁移性**：适用于任何有层级执行体的资源配额系统：K8s namespace ResourceQuota、AWS OU SCP、组织级 API 速率限制。Arc 是 Rust 特定实现，但"共享配额 + 加权计费 + 阈值提醒"通用。
- **跨实例验证**：RolloutBudget（token 预算）和 Guardian 熔断器（拒绝率预算，EK-03）都是"共享计数器 + 阈值触发动作"模式的实例。

**模式描述**：在层级执行体树的根节点创建共享资源预算，通过引用计数共享给所有子节点。资源消耗按权重计费（不同资源类型权重不同），优先使用精确计量。预算耗尽后 latch。阈值穿越时触发提醒，每节点去重，支持手动 rearm。可选功能未配置时 no-op。与并发数限制互补。

### 反例攻击（3 条）

1. **subagent 快速消耗预算让 root 后续无预算可用（DoS）** → 共享配额系统的固有特性——配额是共享的，子节点确实可以消耗父节点预算。但这是设计意图（整个 agent 树有统一预算），不是漏洞。如需隔离应使用 per-agent 配额（Codex 未提供）。**设计意图，非漏洞**。
2. **Mutex\<RolloutBudgetState\> 高并发下成为锁竞争瓶颈** → record_usage 仅在每次 response.completed 时调用（低频），Mutex 持有时间极短（f64 加法+比较）。**不成立（预期负载下无竞争问题）**。
3. **服务端 codex_rollout_budget_units 被篡改导致计费不准确** → 来自 Responses API 的 response.completed 事件，HTTPS 传输，中间人攻击需 TLS 降级或根证书篡改，超出应用层治理范围。本地 fallback 计费在服务端 units 缺失时兜底（EK-23）。**超出应用层边界**。

### 不尝试 L4+

此模式是资源配额的工程实现，不具备模型级预测力。**停留在 L3 Pattern**。

---

## KO-07：控制面生命周期保守降级模式（v3 新增）

**KO ID**: ko-control-plane-lifecycle-conservative-degradation
**knowledge_layer**: generalized
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（多子系统交叉验证：网络回调 + Guardian + PendingApprovalDecision + AgentControl）
**value**: A

### derivation.facts

- EK-10（Weak\<Session\> 代理回调避免循环引用）
- EK-11（Weak\<ThreadManagerState\> 控制面断环）
- EK-19（Session 销毁保守降级 ask("not_allowed")）
- EK-25（Guardian fail-closed 超时/错误→Deny）
- EK-26（PendingApprovalDecision::Deny drop 默认拒绝）

### L1 事实

| 事实 | 锚点 | 回溯 EK |
|------|------|---------|
| 网络代理回调通过 Weak\<Session\> 持有 session，upgrade() 失败→ask("not_allowed") | network_approval.rs:1014-1030 | EK-10, EK-19 |
| AgentControl 通过 Weak\<ThreadManagerState\> 持有全局状态，避免循环引用 | agent/control.rs:121-124 | EK-11 |
| Guardian 超时→TimedOut，PromptBuild/Session/Parse 错误→Deny（fail-closed） | review.rs:616, 655-689 | EK-25 |
| PendingApprovalDecision drop 时默认 Deny（未完成审批=未授权=不执行） | network_approval.rs:300 | EK-26 |

### L2 知识

Codex 中存在一个贯穿多个子系统的设计原则：**当控制面生命周期结束或不可用时，执行面必须保守降级，绝不默认放行**。这个原则有四个独立实例：

1. **网络代理回调**：控制面 session 释放（Weak upgrade 失败）→ 执行面代理保守降级为 ask("not_allowed")（将决策推给用户），而非 Allow 或 Deny。
2. **Guardian 审批器**：审批器自身不可用（超时/解析错误/会话错误）→ fail-closed Deny（直接拒绝），因为审批器不可用时"不执行"比"可能执行危险命令"安全。
3. **待处理审批决策**：审批流程未完成就被丢弃（drop）→ 默认 Deny，因为"未授权即拒绝"。
4. **控制面引用**：AgentControl 通过 Weak\<ThreadManagerState\> 持有全局状态，避免循环引用导致控制面无法释放；upgrade 失败时返回错误而非 panic。

这四个实例的共同模式是：**执行面不持有控制面的强引用（用 Weak/RAII），控制面不可用时执行面选择最安全的降级路径（ask/deny/error），绝不默认 allow**。

### L3 模式（晋升论证 L2→L3）

**模式名称**：控制面生命周期保守降级（Control-Plane Lifecycle Conservative Degradation）

- **可迁移性**：适用于任何有"控制面 + 执行面"分离的系统：API Gateway（控制面=配置中心，执行面=网关代理）、Sidecar 代理（控制面=管理平面，执行面=data plane）、Callback Runtime（控制面=回调注册中心，执行面=回调执行器）、Workflow Engine（控制面=编排器，执行面=worker）。
- **跨实例验证**：四个独立子系统（网络代理、Guardian、审批决策、AgentControl）都采用此原则，不是单个组件的特例。网络用 ask（推给用户），Guardian 用 Deny（直接拒绝），PendingApproval 用 Deny（RAII），Weak 用 error（返回错误）——降级路径的严格程度取决于"控制面不可用时是否还有人能做决策"。
- **不变量论证**：执行面绝不默认 allow 是安全不变量——控制面不可用时，任何"默认放行"都可能是安全漏洞。

**模式描述**：在控制面/执行面分离的系统中，执行面通过弱引用（Weak）或 RAII 持有控制面引用，避免循环引用和悬挂回调。当控制面生命周期结束或不可用时，执行面选择最安全的降级路径：
- 若有人类决策者可用 → ask（将决策推给用户）
- 若无人决策且执行有风险 → deny（直接拒绝）
- 若审批未完成 → RAII drop 默认 deny
- 若仅是引用失效 → 返回 error（不 panic）
**核心不变量：执行面绝不默认 allow。**

### 反例攻击（3 条）

1. **保守降级导致用户体验差（频繁 ask 或 deny）** → 这是安全与体验的权衡。Codex 的设计是"安全优先"——控制面不可用时，保守降级是正确选择。用户可通过配置 Full Access（EK-32）或 Never 审批模式来减少审批，但这是显式选择，不是默认行为。**设计意图，非缺陷**。
2. **Weak 引用的 upgrade() 存在竞态——upgrade 成功后 session 立即释放** → upgrade() 返回 Arc\<Session\>，持有 Arc 期间 session 不会释放（Arc 强引用阻止 drop）。竞态窗口是"upgrade 检查到返回 Arc 之间"，但 Rust 的 Arc::upgrade 是原子操作，要么成功（持有强引用）要么失败（已释放），不存在中间态。**不成立（Rust Arc 原子性保证）**。
3. **fail-closed 被滥用——所有错误都 Deny 导致系统不可用** → Codex 区分了 transient 和 non-transient 错误（EK-25）：transient 错误（网络波动、服务过载）重试最多 3 次，non-transient 错误（prompt 构建失败、会话创建失败）直接 Deny。这避免了"临时故障导致永久拒绝"。**部分缓解：transient 重试机制防止误拒**。

### 为什么是 L3 而非 L4

此模式描述的是"控制面不可用时执行面如何降级"的可复用设计模板，具备明确的结构（弱引用 + 降级路径选择）和可迁移性（API Gateway/Sidecar/Callback Runtime）。但它不具备 L4 认知模型所需的"预测力"——无法预测给定系统的具体降级行为（取决于系统是否有人类决策者、执行风险等级等外部因素）。因此停留在 L3 Pattern。

### 与 KO-04 的区别

KO-04（代理层回调 + Weak ref 生命周期）聚焦于**网络代理这一个具体场景**的回调机制。KO-07 是从网络回调、Guardian、PendingApprovalDecision、AgentControl 四个子系统中归纳出的**跨子系统通用模式**——控制面生命周期保守降级。KO-04 是 KO-07 的一个实例（网络代理场景），KO-07 的解释范围更大。

---

## KO-08：一致性保护的 SSOT 模式（v3 新增）

**KO ID**: ko-consistency-protected-ssot
**knowledge_layer**: generalized
**类型**: Pattern (L3)
**epistemic_status**: Validated Pattern（代码直接验证：PermissionProfileState + deny-read 保留 + sandbox NoOverride）
**value**: B

### derivation.facts

- EK-06（PermissionProfileState Session 级权限 SSOT）
- EK-16（set_permission_profile_projection 三字段原子同步）
- EK-33（denied_read_active 时沙箱升级 NoOverride）

### L1 事实

| 事实 | 锚点 | 回溯 EK |
|------|------|---------|
| PermissionProfileState 封装 constrained profile + active profile id + workspace roots，注释要求三字段同步 | session/session.rs:96-99 | EK-06 |
| set_permission_profile_projection 在更新时原子同步三字段 | session/session.rs:504-538 | EK-16 |
| preserve_deny_read_restrictions_from 在 profile 切换时保留 deny-read 限制 | session/session.rs:360-502 | EK-06, EK-16 |
| denied_read_active 时沙箱升级恒不可能（NoOverride），必须保留沙箱执行 deny-read | sandboxing.rs:238-267 | EK-33 |

### L2 知识

Codex 的权限系统采用"单一事实源 + 一致性保护"模式。PermissionProfileState 将三个强相关字段（constrained profile、active profile id、profile-defined workspace roots）封装为 SSOT，通过方法更新保证一致性，不允许独立修改。更关键的是**一致性保护**：更新时通过 `preserve_deny_read_restrictions_from` 保留旧 profile 的 deny-read 限制，防止 profile 切换作为绕过读取限制的手段。这一保护在执行层也有对应：`denied_read_active` 时沙箱升级恒不可能（NoOverride），因为跳出沙箱等于绕过 deny-read。

这不是简单的 SSOT 模式（单一职责原则的直接应用），而是**带安全不变量保护的 SSOT**——SSOT 不仅保证数据一致性，还保证"安全限制在状态变更时不被意外放宽"。

### L3 模式（晋升论证 L2→L3）

**模式名称**：一致性保护的 SSOT（Consistency-Protected Single Source of Truth）

- **可迁移性**：适用于任何有"多字段强相关 + 安全限制"的状态管理系统：用户权限系统（角色 + 权限 + 资源范围，切换角色时保留安全限制）、订阅系统（计划 + 功能集 + 配额，降级时保留数据访问限制）、多租户系统（tenant + 角色 + 数据范围，切换 tenant 时保留隔离限制）。
- **跨实例验证**：PermissionProfileState（三字段同步 + deny-read 保留，EK-06）和沙箱升级决策（deny-read active → NoOverride，EK-33）是同一模式在数据层和执行层的两个实例——数据层保证"切换时不放宽限制"，执行层保证"运行时不绕过限制"。
- **不变量论证**：安全限制在状态变更时不被意外放宽是安全不变量——状态变更（profile 切换、角色变更、计划变更）是常见的安全绕过向量，必须显式保护。

**模式描述**：将多个强相关字段封装为单一事实源（SSOT），通过方法更新保证一致性（不允许独立修改）。关键是**一致性保护**：在状态变更（切换/更新/迁移）时，显式保留安全限制（如 deny-read、访问控制、配额限制），防止状态变更作为绕过安全限制的手段。保护应在数据层（更新时保留）和执行层（运行时强制）双重实现。

### 反例攻击（3 条）

1. **preserve_deny_read_restrictions_from 可能保留了过于严格的限制，导致新 profile 本应允许的访问被拒绝** → 这是安全与功能的权衡。deny-read 是显式配置的安全限制，保留它是正确的。如果新 profile 应允许更多访问，用户应显式移除 deny-read 限制，而非通过 profile 切换隐式放宽。**设计意图，非缺陷**。
2. **SSOT 封装导致字段访问需要通过方法，增加了间接层和性能开销** → PermissionProfileState 的方法是简单的字段读取/设置，性能开销可忽略。一致性保证的价值远大于间接层的微小开销。**不成立（性能影响可忽略）**。
3. **执行层 NoOverride（deny-read active 时不跳出沙箱）与数据层 deny-read 保留可能不一致——数据层保留了但执行层未检查** → Codex 在两层都实现了保护：数据层 `preserve_deny_read_restrictions_from`（EK-16）+ 执行层 `denied_read_active? → NoOverride`（EK-33）。双重实现保证了一致性。但如果未来新增执行路径（如新的工具运行时）未检查 deny-read，可能出现不一致。**部分成立：需确保所有执行路径都检查 deny-read**。

### 为什么是 L3 而非 L2

v2 KO-7 将 PermissionProfile SSOT 判定为 L2（"单一职责原则的直接应用，不具备模式级新颖性"）。v3 重新评估后晋升为 L3，因为：
1. 它不是简单的 SSOT，而是**带安全不变量保护的 SSOT**——"状态变更时保留安全限制"是超出单一职责原则的设计决策。
2. 它在数据层和执行层有双重实现（PermissionProfileState + sandbox NoOverride），验证了模式的一致性。
3. 它可迁移到任何有"多字段强相关 + 安全限制"的系统（用户权限、订阅、多租户），具备模式级可迁移性。

---

## KO 汇总与三层回溯矩阵

| KO ID | 类型 | 层级 | 反例数 | derivation.facts（回溯 EK） | v2 来源 |
|-------|------|------|--------|---------------------------|---------|
| KO-01 | 三态审批+修正案 | L3 Pattern | 3 | EK-01, EK-18, EK-21, EK-52 | v2 KO-1 |
| KO-02 | 策略持久化热更新 | L4 Model | 3 | EK-02, EK-09, EK-12, EK-20, EK-27, EK-28, EK-37, EK-40, EK-49 | v2 KO-2 |
| KO-03 | Guardian 独立审查+熔断 | L4 Model | 3 | EK-03, EK-14, EK-22, EK-25, EK-34, EK-35, EK-39, EK-48, EK-50 | v2 KO-3 |
| KO-04 | 网络代理回调治理 | L3 Pattern | 3 | EK-04, EK-10, EK-15, EK-19, EK-31, EK-45 | v2 KO-4 |
| KO-05 | 上下文注入硬预算 | L3 Pattern | 3 | EK-08, EK-22, EK-24, EK-36, EK-51 | v2 KO-5 |
| KO-06 | 跨 agent 共享预算 | L3 Pattern | 3 | EK-05, EK-13, EK-23, EK-41, EK-46 | v2 KO-6 |
| KO-07 | 控制面生命周期保守降级 | L3 Pattern | 3 | EK-10, EK-11, EK-19, EK-25, EK-26 | **v3 新增**（从 v2 KO-7 Weak ref 部分 + Guardian + PendingApproval 归纳） |
| KO-08 | 一致性保护的 SSOT | L3 Pattern | 3 | EK-06, EK-16, EK-33 | **v3 新增**（从 v2 KO-7 PermissionProfile 部分晋升） |

**总计**：8 个 KO（2 个 L4 Model + 6 个 L3 Pattern），其中 6 个复用 v2（已独立核验），2 个为 v3 新增（从 v2 KO-7 的两个 L2 主题分别晋升/归纳）。所有 KO 均含 ≥3 条反例攻击和晋升论证块，所有 KO 的 `derivation.facts` 均指回 Engineering Knowledge 层的 EK 编号（**底座可回溯率 100%**，无悬空升维）。

### 层级分布

| 层级 | 数量 | KO |
|------|------|-----|
| L4 Cognitive Model | 2 | KO-02, KO-03 |
| L3 Pattern | 6 | KO-01, KO-04, KO-05, KO-06, KO-07, KO-08 |
| L5 Methodology | 0 | （无 KO 达到方法论级——Codex 缺少"策略效果评估→自动调优"和"审批器定期审计"闭环，KO-02/KO-03 尝试 L5 后降级） |

**窄尖顶验证**：8 个 KO 从 52 条工程知识中晋升，晋升率 15.4%（8/52），符合"升维是特例不是默认"的原则。L5 为空是诚实标注——Codex 的机制还不足以支撑方法论级认知（缺少自我评估/调优闭环）。

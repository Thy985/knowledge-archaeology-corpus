# Wave 2 — 挖掘图 (v2)

> 包含 Problem Graph、Decision Graph、Pattern Graph、Flow Atlas（含 v2 新增 Policy Flow）。所有节点锚定 Wave1 证据包。

---

## 一、Problem Graph（问题图谱）

### P1：不可信执行体的命令审批边界
- **根问题**：AI agent 生成的 shell 命令可能危险，如何在不打断用户的前提下安全执行？
- **子问题**：
  - P1a：审批粒度——逐命令审批太吵，全放行太危险
  - P1b：审批结果复用——同一命令反复审批
  - P1c：沙箱与审批的关系——沙箱能保护时是否还需要审批？
- **锚点**：EP-01（ExecApprovalRequirement 三态）、EP-02（策略固化）

### P2：策略规则的持久化与热更新一致性
- **根问题**：用户审批"允许这个命令"后，如何让这个决定跨 session 持久化，且不重启即可生效？
- **子问题**：
  - P2a：并发 append 导致规则丢失/乱序
  - P2b：内存策略与磁盘规则不一致
  - P2c：哪些命令可以被自动固化（shell 包装器绝对不行）
- **锚点**：EP-02（ArcSwap+Semaphore、BANNED_PREFIX_SUGGESTIONS、append_amendment_and_update）

### P3：自动审批器（Guardian）的可靠性
- **根问题**：用另一个 AI 来审批 AI 的命令，如何保证审批器本身不成为单点故障？
- **子问题**：
  - P3a：审批器超时/出错时怎么办（fail-open vs fail-closed）
  - P3b：审批器被恶意 prompt 注入怎么办（独立 session、不继承 exec-policy）
  - P3c：审批器连续拒绝时如何熔断
- **锚点**：EP-03（Guardian 独立 session、fail-closed、熔断器）

### P4：网络访问的运行时治理
- **根问题**：命令执行时发起网络请求，如何在代理层拦截未授权域名并回调审批？
- **子问题**：
  - P4a：代理 allowlist-miss 时如何回调 core 决策
  - P4b：网络规则如何持久化（与命令规则共享机制）
  - P4c：session 释放后代理回调的生命周期
- **锚点**：EP-04（NetworkPolicyDecider、Weak<Session>、append_network_rule_and_update）

### P5：上下文注入的安全边界
- **根问题**：向模型上下文注入信息时，如何防止注入项过大/频繁变化导致缓存失效或上下文溢出？
- **子问题**：
  - P5a：注入项的大小硬上限
  - P5b：注入项的类型安全（必须实现 trait）
  - P5c：Guardian 等子系统的上下文预算独立
- **锚点**：EP-05（AGENTS.md 6 条硬约束）、EP-03d（Guardian 5 层限流）

### P6：多 agent 树的共享资源治理
- **根问题**：root agent spawn 多个 subagent 后，如何对整个 agent 树的 token 消耗做统一预算？
- **子问题**：
  - P6a：预算状态在 agent 树间共享（Arc）
  - P6b：预算耗尽的检测与提醒
  - P6c：引用循环（ThreadManagerState ↔ Session）
- **锚点**：EP-06（RolloutBudget）、EP-08（AgentControl Weak ref）

### P7：权限配置的单一事实源与一致性
- **根问题**：PermissionProfile、active profile id、workspace roots 三者如何保持同步？
- **子问题**：
  - P7a：更新时的原子性
  - P7b：deny-read 限制在 profile 切换时的保留
- **锚点**：EP-07（PermissionProfileState）

---

## 二、Decision Graph（决策图谱）

### D1：ExecApprovalRequirement 三态决策
- **决策点**：每个工具调用 → Skip / NeedsApproval / Forbidden
- **输入**：AskForApproval（Never/OnRequest/Granular/UnlessTrusted）+ FileSystemSandboxPolicy.kind + exec_policy 规则匹配
- **决策逻辑**（sandboxing.rs:194-230 + exec_policy.rs:394-461）：
  1. exec_policy 规则匹配 → Decision::{Forbidden, Prompt, Allow}
  2. Forbidden → ExecApprovalRequirement::Forbidden
  3. Prompt → 检查 prompt_is_rejected_by_policy → 若被 Granular 配置拒绝则 Forbidden，否则 NeedsApproval
  4. Allow → Skip（bypass_sandbox 取决于是否每段命令都有显式 policy Allow）
- **关键不变量**：Forbidden 不产生 proposed_execpolicy_amendment（无法固化拒绝）

### D2：策略修正案是否可固化
- **决策点**：proposed_execpolicy_amendment 是否 Some
- **三道过滤**（exec_policy.rs:958-997）：
  1. BANNED_PREFIX_SUGGESTIONS：88 个危险前缀直接过滤
  2. is_policy_match：已有 policy rule 匹配时不追加（避免冲突）
  3. prefix_rule_would_approve_all_commands：模拟添加后验证所有解析出的命令段都变 Allow
- **锚点**：EP-02d, EP-02b

### D3：Guardian 路由决策
- **决策点**：NeedsApproval 后走 Guardian 还是用户提示
- **条件**（review.rs:209-217）：approval_policy ∈ {OnRequest, Granular} ∧ approvals_reviewer == AutoReview
- **短路**：has_full_access → 直接 Approved（review.rs:330-343）

### D4：Guardian 失败模式决策
- **决策点**：Guardian review 出错时的行为
- **决策**（review.rs:655-689）：
  - Timeout → ReviewDecision::TimedOut（不执行）
  - PromptBuild/Session/Parse → 构造 risk=High, outcome=Deny 的 assessment（fail-closed）
  - 仅 transient 错误重试（最多 3 次，90s deadline）

### D5：熔断器触发决策
- **决策点**：Guardian 连续/近期拒绝是否中断 turn
- **阈值**（mod.rs:64-68）：
  - Standard：连续 3 次 ∨ 最近 50 次中 10 次
  - Cyber：连续 1 次 ∨ 最近 50 次中 1 次
- **动作**：InterruptTurn + GuardianWarning 事件（review.rs:269-307）

### D6：网络代理回调决策
- **决策点**：代理 allowlist-miss 时
- **决策**（network_approval.rs:1014-1030）：
  - session 存活 → handle_inline_policy_request（Allow/Deny/Ask）
  - session 已释放 → NetworkDecision::ask("not_allowed")（保守降级）

### D7：沙箱升级（escalation）决策
- **决策点**：命令在沙箱中失败后是否升级到无沙箱执行
- **决策**（sandboxing.rs:238-267）：
  - 有 denied-read 限制 → NoOverride（必须保留沙箱以执行 deny-read）
  - Skip{bypass_sandbox:true} → BypassSandboxFirstAttempt
  - sandbox_permissions.requires_escalated_permissions() → BypassSandboxFirstAttempt

---

## 三、Pattern Graph（模式图谱）

### Pattern 1：三态审批枚举 + 修正案携带
- **模式**：审批结果不是二元的，而是 {Skip(携带bypass+修正案), NeedsApproval(携带修正案), Forbidden(携带原因)}
- **可迁移性**：任何需要"审批+学习"的系统都适用——审批结果本身携带"是否应该固化为规则"的建议
- **同类对照**：GitHub CODEOWNERS（二元）、AWS IAM（策略驱动无运行时学习）
- **锚点**：EP-01

### Pattern 2：ArcSwap 读 + Semaphore 写的策略热更新
- **模式**：读路径无锁（ArcSwap::load_full），写路径串行化（Semaphore=1），写时 clone→modify→store(Arc::new)
- **可迁移性**：任何读多写少、需要热更新的配置系统
- **同类对照**：Nginx reload（进程级）、K8s ConfigMap（watch 机制）
- **锚点**：EP-02a, EP-02b

### Pattern 3：独立审查 session + fail-closed
- **模式**：审批器运行在独立 session 中（approval_policy=never、不继承 exec-policy、只读沙箱），超时/出错时默认拒绝
- **可迁移性**：任何用 AI 审查 AI 的系统（代码审查器、安全扫描器）
- **同类对照**：AWS Config Rules（fail-closed）、OPA（策略即代码，无 AI 不确定性）
- **锚点**：EP-03c, EP-03e

### Pattern 4：拒绝熔断器（滑动窗口 + 连续计数）
- **模式**：连续拒绝计数 + 滑动窗口拒绝率，双阈值触发中断；Cyber 模型阈值更敏感
- **可迁移性**：任何自动决策系统的防失控机制
- **同类对照**：Hystrix 熔断器（调用失败率）、circuit_breaker 模式
- **锚点**：EP-03f

### Pattern 5：代理层回调 + Weak ref 生命周期
- **模式**：网络代理在 allowlist-miss 时回调 core 决策，通过 Weak<Session> 持有引用避免循环，session 释放时保守降级
- **可迁移性**：任何需要运行时回调决策的代理/中间件系统
- **同类对照**：Envoy ext_authz（gRPC 回调）、OAuth2 资源服务器 introspection
- **锚点**：EP-04a, EP-04b

### Pattern 6：上下文注入的类型安全 + 硬预算
- **模式**：所有上下文注入项必须实现统一 trait（ContextualUserFragment），并有全局硬上限（10K/项、>1K 需 P0 审查）
- **可迁移性**：任何 LLM 上下文工程系统
- **同类对照**：LangChain document compressors（无硬预算）、Anthropic prompt caching（自动）
- **锚点**：EP-05

### Pattern 7：跨 agent 树的 Arc 共享预算
- **模式**：资源预算（token 消耗）通过 Arc 在 agent 控制面共享，OnceLock 延迟初始化，加权计费，阈值穿越提醒
- **可迁移性**：任何有层级执行体的资源配额系统
- **同类对照**：K8s ResourceQuota（namespace 级）、Linux cgroup（层级配额）
- **锚点**：EP-06, EP-08

### Pattern 8：Weak ref 断环
- **模式**：控制面（AgentControl）通过 Weak<ThreadManagerState> 持有全局状态，避免 ThreadManagerState→CodexThread→Session→SessionServices→ThreadManagerState 循环
- **可迁移性**：任何有父子双向引用的系统
- **同类对照**：DOM 事件委托、Rust 常见的 parent Weak + child Arc 模式
- **锚点**：EP-08

---

## 四、Flow Atlas（流图集）

### Flow 1：Policy Flow（v2 新增 — 策略→审批→执行→固化 完整闭环）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Phase 0: 策略加载 (Session init)                                             │
│                                                                             │
│ config_stack.layers_low_to_high()                                           │
│   → 每层 config_folder/rules/*.rules → PolicyParser.parse()                 │
│   → requirements().exec_policy merge_overlay()                              │
│   → ExecPolicyManager { policy: ArcSwap<Policy>, update_lock: Semaphore(1) }│
│                                                                             │
│ 锚点: exec_policy.rs:662-716, 276-299                                      │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Phase 1: 审批需求计算 (Tool call entry)                                      │
│                                                                             │
│ ToolRuntime::exec_approval_requirement(req)                                 │
│   → ExecPolicyManager::create_exec_approval_requirement_for_command()       │
│     → commands_for_exec_policy_for_platform() (shell 解析)                 │
│     → exec_policy.check_multiple_with_options(commands, fallback, opts)    │
│       → Decision::{Forbidden|Prompt|Allow} + matched_rules                 │
│     → match evaluation.decision:                                             │
│       Forbidden → ExecApprovalRequirement::Forbidden { reason }             │
│       Prompt    → prompt_is_rejected_by_policy()?                           │
│                   → Forbidden (Granular rules/sandbox_approval disabled)   │
│                   → NeedsApproval { reason, proposed_execpolicy_amendment } │
│       Allow     → Skip { bypass_sandbox: all_segments_explicitly_allowed,  │
│                         proposed_execpolicy_amendment }                      │
│                                                                             │
│ 锚点: exec_policy.rs:337-462, sandboxing.rs:194-230                        │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Phase 2: 审批执行 (Guardian or User)                                         │
│                                                                             │
│ if Skip → 直接执行 (Phase 3)                                                │
│ if Forbidden → 拒绝执行，返回 reason                                         │
│ if NeedsApproval:                                                            │
│   → routes_approval_to_guardian(approval_policy, approvals_reviewer)        │
│     → OnRequest|Granular + AutoReview → Guardian review                     │
│       → has_full_access? → 直接 Approved (短路)                             │
│       → 独立 review session (approval_policy=never, 无继承 exec-policy)     │
│       → 90s timeout, 最多 3 次重试 (仅 transient)                           │
│       → fail-closed: Timeout→TimedOut, Parse/Session/PromptBuild→Deny      │
│       → 熔断器: 连续3次/最近50次中10次拒绝 → InterruptTurn                  │
│     → 否则 → 用户提示                                                         │
│                                                                             │
│ 锚点: review.rs:196-217, 321-343, 655-689, mod.rs:64-68, 196-246        │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Phase 3: 执行 (SandboxAttempt)                                               │
│                                                                             │
│ SandboxAttempt { sandbox, sandbox_requested, permissions, ... }             │
│   → sandbox_override_for_first_attempt()                                     │
│     → denied_read_active? → NoOverride (必须保留沙箱执行 deny-read)          │
│     → Skip{bypass_sandbox:true} → BypassSandboxFirstAttempt                 │
│     → requires_escalated_permissions() → BypassSandboxFirstAttempt          │
│   → 执行命令                                                                  │
│   → 沙箱失败 + wants_no_sandbox_approval → 提示用户跳出沙箱                   │
│                                                                             │
│ 锚点: sandboxing.rs:238-295, 386-405                                        │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ Phase 4: 策略固化 (Amendment → Persistence)                                  │
│                                                                             │
│ proposed_execpolicy_amendment (来自 Phase 1)                                 │
│   → 三道过滤:                                                                 │
│     1. BANNED_PREFIX_SUGGESTIONS (88个危险前缀: bash -c, python -c, sudo...) │
│     2. 已有 policy rule 匹配 → 不追加                                        │
│     3. prefix_rule_would_approve_all_commands() 模拟验证                    │
│   → append_amendment_and_update(codex_home, amendment)                       │
│     → update_lock.acquire() (Semaphore=1 串行化)                             │
│     → spawn_blocking(blocking_append_allow_prefix_rule(default.rules))       │
│     → 幂等检查: 当前策略已 Allow 则跳过                                        │
│     → updated_policy.add_prefix_rule(prefix, Allow)                          │
│     → policy.store(Arc::new(updated_policy)) (ArcSwap 原子热更新)            │
│                                                                             │
│ 网络规则同理: append_network_rule_and_update() → blocking_append_network_rule │
│                                                                             │
│ 锚点: exec_policy.rs:464-512, 514-558, 916-997, 57-146                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Flow 2：Guardian Review Flow（v2 细化）

```
Tool call (NeedsApproval)
  │
  ▼
routes_approval_to_guardian()?
  │
  ├─ No → 用户提示 (ReviewDecision from user)
  │
  └─ Yes → has_full_access()?
           │
           ├─ Yes → ReviewDecision::Approved (短路)
           │
           └─ No → build_guardian_review_session_config()
                    │
                    ▼
              GuardianReviewSessionManager::run_review()
                    │
                    ├─ trunk idle → 追加到 cached trunk conversation (preserve cache key)
                    └─ trunk busy → ephemeral fork from last committed rollout
                    │
                    ▼
              Guardian assessment (risk_level, user_authorization, outcome, rationale)
                    │
                    ▼
              parse_guardian_assessment()
                    │
                    ├─ Ok → outcome Allow/Deny → ReviewDecision
                    └─ Err → fail-closed Deny
                    │
                    ▼
              record_guardian_denial/non_denial() → 熔断器更新
```

### Flow 3：Network Approval Flow（v2 新增）

```
Tool execution begins
  │
  ▼
begin_network_approval(session, turn, managed_network_active, spec)
  │
  ├─ spec None / managed_network_active false → None (无网络治理)
  │
  └─ Some → resolve owner_spec (NetworkProxySpec::for_environment)
           │
           ▼
         build execution-scoped NetworkProxy
           │
           ▼
         fallback_policy_decider = build_network_policy_decider(Weak<Session>)
           │
           ▼
         network.for_execution(..., fallback_policy_decider)
           │
           ▼
         ActiveNetworkApproval { registration_id, cancellation_token, execution_proxy }
           │
           ▼
         命令执行中发起网络请求
           │
           ├─ allowlist hit → 放行
           └─ allowlist miss → NetworkPolicyDecider::decide(request)
                    │
                    ├─ session 存活 → handle_inline_policy_request() → Allow/Deny/Ask
                    └─ session 已释放 → NetworkDecision::ask("not_allowed")
                    │
                    ▼
              用户审批 Allow → append_network_rule_and_update() → default.rules 持久化
```

### Flow 4：RolloutBudget Flow（v2 新增）

```
AgentControl::new(rollout_budget_config)
  │
  ▼
RolloutBudget::configure(config) → OnceLock<Mutex<RolloutBudgetState>>
  │
  ▼ (Arc<RolloutBudget> 共享给整个 agent 树)
每个 agent 的每个 response.completed
  │
  ▼
RolloutBudget::record_usage(TokenUsage)
  │
  ├─ codex_rollout_budget_units 存在 → 使用服务端精确 units
  └─ 否则 → output_tokens * sampling_weight + non_cached_input * prefill_weight
  │
  ▼
weighted_tokens_used += units
  │
  ▼
exhausted = (weighted_tokens_used >= limit_tokens)
  │
  ▼ (每次 turn 开始前)
pending_reminder(thread_id, window_id)
  │
  ├─ 已交付过同 index → None (去重)
  └─ 新 threshold 穿越 → Some(RolloutBudgetReminder { remaining_tokens, reminder_index })
  │
  ▼
注入 context → 模型感知剩余预算
```

---

## 五、Flow→KO 交叉校验矩阵（v2 新增机制）

| Flow Edge | 关联 KO | 校验状态 | 说明 |
|-----------|---------|---------|------|
| Policy Flow Phase 1 → ExecApprovalRequirement 三态 | KO-1 | ✅ | 三态枚举与决策逻辑一致 |
| Policy Flow Phase 4 → 策略固化 | KO-2 | ✅ | ArcSwap+Semaphore+BANNED 过滤与代码一致 |
| Guardian Flow → fail-closed + 熔断器 | KO-3 | ✅ | 90s/3次/3-10阈值与代码一致 |
| Network Flow → Weak ref 回调 | KO-4 | ✅ | session 释放→ask("not_allowed") |
| Context Flow → 硬预算 | KO-5 | ✅ | AGENTS.md 6条 + Guardian 5层 |
| RolloutBudget Flow → Arc共享 | KO-6 | ✅ | AgentControl.rollout_budget: Arc |
| PermissionProfile SSOT → 一致性 | KO-7 | ✅ | set_permission_profile_projection |

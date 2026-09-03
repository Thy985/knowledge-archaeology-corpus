# Flow Atlas（流图集 · 中间层）

> **v3 三层架构的中间层。** 七类流（Control/State/Data/Evidence/Authority/Memory/Policy）作为机制的运行时视图，连接 Engineering Knowledge 与 Generalized KO。
> 复用 v2 mining-graphs 中的 Flow Atlas（已独立核验），按 v3 三层架构重新标注与 EK/KO 的关联。

---

## Flow 1：Policy Flow（策略→审批→执行→固化 完整闭环）

> v2 核心新增流。关联 EK-01/02/09/12/18/20/21/27/37/40/42/49/52，KO-01/02。

```
Phase 0: 策略加载 (Session init)
  config_stack.layers_low_to_high()
    → 每层 config_folder/rules/*.rules → PolicyParser.parse()
    → requirements().exec_policy merge_overlay()
    → ExecPolicyManager { policy: ArcSwap<Policy>, update_lock: Semaphore(1) }
  锚点: exec_policy.rs:662-716, 276-299 → EK-40, EK-09

Phase 1: 审批需求计算 (Tool call entry)
  ToolRuntime::exec_approval_requirement(req)
    → ExecPolicyManager::create_exec_approval_requirement_for_command()
      → commands_for_exec_policy_for_platform() (shell 解析)
      → exec_policy.check_multiple_with_options(commands, fallback, opts)
        → Decision::{Forbidden|Prompt|Allow} + matched_rules
      → match evaluation.decision:
        Forbidden → ExecApprovalRequirement::Forbidden { reason }
        Prompt    → prompt_is_rejected_by_policy()?
                    → Forbidden (Granular rules/sandbox_approval disabled)
                    → NeedsApproval { reason, proposed_execpolicy_amendment }
        Allow     → Skip { bypass_sandbox: all_segments_explicitly_allowed,
                            proposed_execpolicy_amendment }
  锚点: exec_policy.rs:337-462, sandboxing.rs:194-230 → EK-01, EK-18, EK-21

Phase 2: 审批执行 (Guardian or User)
  if Skip → 直接执行 (Phase 3)
  if Forbidden → 拒绝执行，返回 reason
  if NeedsApproval:
    → routes_approval_to_guardian(approval_policy, approvals_reviewer)
      → OnRequest|Granular + AutoReview → Guardian review
        → has_full_access? → 直接 Approved (短路)
        → 独立 review session (approval_policy=never, 无继承 exec-policy)
        → 90s timeout, 最多 3 次重试 (仅 transient)
        → fail-closed: Timeout→TimedOut, Parse/Session/PromptBuild→Deny
        → 熔断器: 连续3次/最近50次中10次拒绝 → InterruptTurn
      → 否则 → 用户提示
  锚点: review.rs:196-217, 321-343, 655-689, mod.rs:64-68, 196-246 → EK-03, EK-25, EK-35, EK-47, EK-50

Phase 3: 执行 (SandboxAttempt)
  SandboxAttempt { sandbox, sandbox_requested, permissions, ... }
    → sandbox_override_for_first_attempt()
      → denied_read_active? → NoOverride (必须保留沙箱执行 deny-read)
      → Skip{bypass_sandbox:true} → BypassSandboxFirstAttempt
      → requires_escalated_permissions() → BypassSandboxFirstAttempt
    → 执行命令
    → 沙箱失败 + wants_no_sandbox_approval → 提示用户跳出沙箱
  锚点: sandboxing.rs:238-295, 386-405 → EK-33

Phase 4: 策略固化 (Amendment → Persistence)
  proposed_execpolicy_amendment (来自 Phase 1)
    → 三道过滤:
      1. BANNED_PREFIX_SUGGESTIONS (88个危险前缀: bash -c, python -c, sudo...)
      2. 已有 policy rule 匹配 → 不追加
      3. prefix_rule_would_approve_all_commands() 模拟验证
    → append_amendment_and_update(codex_home, amendment)
      → update_lock.acquire() (Semaphore=1 串行化)
      → spawn_blocking(blocking_append_allow_prefix_rule(default.rules))
      → 幂等检查: 当前策略已 Allow 则跳过
      → updated_policy.add_prefix_rule(prefix, Allow)
      → policy.store(Arc::new(updated_policy)) (ArcSwap 原子热更新)
  网络规则同理: append_network_rule_and_update() → blocking_append_network_rule
  锚点: exec_policy.rs:464-512, 514-558, 916-997, 57-146 → EK-02, EK-12, EK-20, EK-27, EK-37, EK-49
```

---

## Flow 2：Guardian Review Flow（Guardian 审查流）

> 关联 EK-03/14/22/25/34/35/39/48/50，KO-03。

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
           ├─ Yes → ReviewDecision::Approved (短路) → EK-32
           │
           └─ No → build_guardian_review_session_config()
                    │
                    ▼
              GuardianReviewSessionManager::run_review()
                    │
                    ├─ trunk idle → 追加到 cached trunk conversation (preserve cache key) → EK-14
                    └─ trunk busy → ephemeral fork from last committed rollout → EK-14
                    │
                    ▼
              Guardian assessment (risk_level, user_authorization, outcome, rationale)
                    │ 输入受 5 层预算限制: 20K/10K/5K/1K/40 → EK-22
                    │ 90s timeout, 最多 3 次重试(仅 transient) → EK-39
                    ▼
              parse_guardian_assessment()
                    │
                    ├─ Ok → outcome Allow/Deny → ReviewDecision
                    └─ Err → fail-closed Deny → EK-25
                    │
                    ▼
              record_guardian_denial/non_denial() → 熔断器更新
                    │ 系统错误导致的 Deny 不计入熔断器 → EK-48
                    │ Standard: 3连续/10近期, Cyber: 1/1 → EK-35
                    │ non_denial 重置连续计数 → EK-34
                    ▼
              超阈值 → InterruptTurn + GuardianWarning → EK-03
```

---

## Flow 3：Network Approval Flow（网络审批流）

> 关联 EK-04/10/15/19/31/45，KO-04/07。

```
Tool execution begins
  │
  ▼
begin_network_approval(session, turn, managed_network_active, spec)
  │
  ├─ spec None / managed_network_active false → None (无网络治理) → EK-45
  │
  └─ Some → resolve owner_spec (NetworkProxySpec::for_environment)
           │
           ▼
         build execution-scoped NetworkProxy
           │
           ▼
         fallback_policy_decider = build_network_policy_decider(Weak<Session>) → EK-10
           │
           ▼
         network.for_execution(..., fallback_policy_decider)
           │
           ▼
         ActiveNetworkApproval { registration_id, cancellation_token, execution_proxy } → EK-15
           │
           ▼
         命令执行中发起网络请求
           │
           ├─ allowlist hit → 放行
           └─ allowlist miss → NetworkPolicyDecider::decide(request)
                    │
                    ├─ session 存活 → handle_inline_policy_request() → Allow/Deny/Ask
                    └─ session 已释放 → NetworkDecision::ask("not_allowed") → EK-19
                    │
                    ▼
              用户审批 Allow → append_network_rule_and_update() → default.rules 持久化 → EK-49
              用户审批 Deny → build_blocked_request_observer 记录 → EK-04
```

---

## Flow 4：RolloutBudget Flow（资源预算流）

> 关联 EK-05/13/23/41/46，KO-06。

```
AgentControl::new(rollout_budget_config)
  │
  ▼
RolloutBudget::configure(config) → OnceLock<Mutex<RolloutBudgetState>> → EK-13
  │ 未配置时所有方法 no-op → EK-46
  ▼ (Arc<RolloutBudget> 共享给整个 agent 树)
每个 agent 的每个 response.completed
  │
  ▼
RolloutBudget::record_usage(TokenUsage)
  │
  ├─ codex_rollout_budget_units 存在 → 使用服务端精确 units
  └─ 否则 → output_tokens * sampling_weight + non_cached_input * prefill_weight → EK-23
  │
  ▼
weighted_tokens_used += units
  │
  ▼
exhausted = (weighted_tokens_used >= limit_tokens)  (latch 行为)
  │
  ▼ (每次 turn 开始前)
pending_reminder(thread_id, window_id)
  │
  ├─ 已交付过同 index → None (去重)
  └─ 新 threshold 穿越 → Some(RolloutBudgetReminder { remaining_tokens, reminder_index })
  │
  ▼
注入 context → 模型感知剩余预算
  │
  ▼
rearm_reminder(thread_id) → 清除 delivery 记录，强制重申 → EK-05
```

---

## Flow→KO 交叉校验矩阵（v2 机制，v3 延续）

| Flow Edge | 关联 KO | 关联 EK | 校验状态 | 说明 |
|-----------|---------|---------|---------|------|
| Policy Flow Phase 1 → ExecApprovalRequirement 三态 | KO-01 | EK-01, EK-21 | ✅ | 三态枚举与决策逻辑一致 |
| Policy Flow Phase 4 → 策略固化 | KO-02 | EK-02, EK-09, EK-20, EK-37 | ✅ | ArcSwap+Semaphore+BANNED 过滤与代码一致 |
| Guardian Flow → fail-closed + 熔断器 | KO-03 | EK-03, EK-25, EK-35 | ✅ | 90s/3次/3-10阈值与代码一致 |
| Network Flow → Weak ref 回调 | KO-04, KO-07 | EK-10, EK-19 | ✅ | session 释放→ask("not_allowed") |
| Context Flow → 硬预算 | KO-05 | EK-08, EK-22, EK-36 | ✅ | AGENTS.md 6条 + Guardian 5层 |
| RolloutBudget Flow → Arc共享 | KO-06 | EK-05, EK-13, EK-23 | ✅ | AgentControl.rollout_budget: Arc |
| PermissionProfile SSOT → 一致性 | KO-08 | EK-06, EK-16, EK-33 | ✅ | set_permission_profile_projection + deny-read |
| Control Plane Lifecycle → 保守降级 | KO-07 | EK-10, EK-11, EK-19, EK-25, EK-26 | ✅ | Weak/ask/fail-closed/drop Deny 四子系统一致 |

**结论**：8 个 KO 全部通过 Flow→KO 交叉校验，每条 L1 事实均可回溯到 Flow Edge 和 EK 编号。无 KO 与 Flow 矛盾。

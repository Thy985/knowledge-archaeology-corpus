# Wave 1 — 多视角证据包 (v2)

> 范围：codex-rs/core/src（Rust 核心）。本波次覆盖 v1 遗漏的六个高密度子系统：exec_policy 持久化、Guardian 独立 session、网络审批、上下文治理、rollout_budget、AGENTS.md 治理。每条证据锚定真实文件:行号。

---

## EP-01：ExecApprovalRequirement 三态决策枚举（审批语义的核心类型）

**子系统**：权限/审批边界
**锚点**：`codex-rs/core/src/tools/sandboxing.rs:152-171`

```rust
pub(crate) enum ExecApprovalRequirement {
    Skip {
        bypass_sandbox: bool,
        proposed_execpolicy_amendment: Option<ExecPolicyAmendment>,
    },
    NeedsApproval {
        reason: Option<String>,
        proposed_execpolicy_amendment: Option<ExecPolicyAmendment>,
    },
    Forbidden { reason: String },
}
```

**关键事实**：
- 三态不是简单的"允许/拒绝"：`Skip` 携带 `bypass_sandbox`（是否跳过沙箱）和 `proposed_execpolicy_amendment`（建议固化的策略修正案）；`NeedsApproval` 同样携带修正案；`Forbidden` 携带拒绝原因。
- `proposed_execpolicy_amendment` 同时出现在 Skip 和 NeedsApproval 中——这意味着**无论命令是否需要审批，系统都在计算"是否应该把这个命令固化为免审规则"**。这是策略闭环的起点。
- `proposed_execpolicy_amendment()` 方法（sandboxing.rs:174-186）从 Skip 和 NeedsApproval 中提取修正案，Forbidden 不产生修正案。

**派生映射**：`default_exec_approval_requirement`（sandboxing.rs:194-230）将 `AskForApproval`（Never/OnRequest/Granular/UnlessTrusted）与 `FileSystemSandboxPolicy.kind`（Restricted/Unrestricted/ExternalSandbox）组合映射到三态。关键分支：Granular 且 `allows_sandbox_approval()==false` → `Forbidden`（审批被策略配置静默拒绝，而非提示用户）。

---

## EP-02：ExecPolicyManager — 策略持久化与热更新闭环（v1 核心遗漏）

**子系统**：exec_policy 策略引擎
**锚点**：`codex-rs/core/src/exec_policy.rs:276-279, 464-512, 514-558`

### 2a. 数据结构：ArcSwap + Semaphore 双保险

```rust
pub(crate) struct ExecPolicyManager {
    policy: ArcSwap<Policy>,      // 无锁读，写时原子替换
    update_lock: Semaphore,       // 串行化所有写操作（permits=1）
}
```

**关键事实**：
- `ArcSwap<Policy>` 允许读路径无锁（`current()` → `load_full()`，exec_policy.rs:311-313），写路径通过 `store(Arc::new(updated_policy))` 原子替换（exec_policy.rs:510）。
- `Semaphore::new(1)`（exec_policy.rs:298）确保策略更新串行化，防止并发 append 导致规则丢失或乱序。

### 2b. 审批→策略固化：append_amendment_and_update

```rust
pub(crate) async fn append_amendment_and_update(
    &self, codex_home: &Path, amendment: &ExecPolicyAmendment
) -> Result<(), ExecPolicyUpdateError> {
    let _update_guard = self.update_lock.acquire().await...;  // 串行化
    let policy_path = default_policy_path(codex_home);       // ~/.codex/rules/default.rules
    spawn_blocking({
        let prefix = amendment.command.clone();
        move || blocking_append_allow_prefix_rule(&policy_path, &prefix)  // 落盘
    }).await...;
    // 幂等检查：如果当前内存策略已经 Allow，跳过
    let existing_evaluation = current_policy.check_multiple_with_options([&amendment.command], &|_| Decision::Forbidden, &match_options);
    if already_allowed { return Ok(()); }
    let mut updated_policy = current_policy.as_ref().clone();
    updated_policy.add_prefix_rule(&amendment.command, Decision::Allow)?;
    self.policy.store(Arc::new(updated_policy));  // 原子热更新
}
```

**关键事实**：
- **落盘路径**：`default_policy_path(codex_home)` = `codex_home/rules/default.rules`（exec_policy.rs:867-869，常量 `RULES_DIR_NAME="rules"`、`DEFAULT_POLICY_FILE="default.rules"`，exec_policy.rs:54-56）。
- **持久化原语**：`blocking_append_allow_prefix_rule`（来自 `codex_execpolicy` crate，exec_policy.rs:20 import）在 `spawn_blocking` 中执行，避免阻塞 async runtime。
- **幂等性**：append 后重新检查当前策略，如果已经 Allow 则不重复 store（exec_policy.rs:491-506）。
- **修正案来源**：`ExecPolicyAmendment` 是 `proposed_execpolicy_amendment` 的载体，在 `create_exec_approval_requirement_for_parsed_commands`（exec_policy.rs:337-462）中通过三个 derive 函数计算：
  - `derive_requested_execpolicy_amendment_from_prefix_rule`（exec_policy.rs:958-997）：从用户传入的 prefix_rule 推导，经过 BANNED_PREFIX_SUGGESTIONS 过滤和 `prefix_rule_would_approve_all_commands` 验证。
  - `try_derive_execpolicy_amendment_for_prompt_rules`（exec_policy.rs:916-935）：从 heuristics Prompt 推导，但如果有 policy rule Prompt 则返回 None（修正案无法跳过显式策略规则）。
  - `try_derive_execpolicy_amendment_for_allow_rules`（exec_policy.rs:940-956）：从 heuristics Allow 推导，仅在沙箱失败后提示用户跳出沙箱时使用。

### 2c. 网络规则持久化：append_network_rule_and_update

```rust
pub(crate) async fn append_network_rule_and_update(
    &self, codex_home: &Path, host: &str, protocol: NetworkRuleProtocol,
    decision: Decision, justification: Option<String>,
) -> Result<(), ExecPolicyUpdateError> {
    // 同样的 update_lock + spawn_blocking + blocking_append_network_rule + 内存热更新
}
```

**关键事实**：网络规则和命令前缀规则共享同一套持久化机制（exec_policy.rs:514-558），都写入 `default.rules`。

### 2d. BANNED_PREFIX_SUGGESTIONS — 禁止自动固化的 88 个危险前缀

**锚点**：exec_policy.rs:57-146

**关键事实**：88 个被禁止自动固化为免审规则的前缀，包括所有 shell 包装器（`bash -c`、`sh -lc`、`powershell -Command`）、所有解释器（`python -c`、`node -e`、`perl -e`）、`sudo`、`rm`、`git` 等。这意味着**用户永远不能通过"允许这一次"来自动固化 `bash -c` 为免审**——每次 `bash -c` 都必须重新审批。这是一个关键的安全不变量。

### 2e. 策略加载：多层 config stack + requirements overlay

**锚点**：exec_policy.rs:662-716

**关键事实**：
- `load_exec_policy` 遍历 `config_stack.layers_low_to_high()`，从每个 layer 的 `config_folder/rules/*.rules` 加载（exec_policy.rs:668-683）。
- 高层 layer 可以覆盖低层 layer 的规则（低到高顺序 append，后加载的规则优先级更高）。
- `ignore_user_and_project_exec_policy_rules()` 可以跳过 User 和 Project layer 的规则（exec_policy.rs:670-677）——这是 managed/enterprise 环境的强制策略机制。
- 最后 `merge_overlay(requirements_policy)` 叠加 requirements 层的强制策略（exec_policy.rs:711-715）。

---

## EP-03：Guardian 自动审批审查器 — 独立 session + fail-closed + 熔断器

**子系统**：Guardian 自动审查
**锚点**：`codex-rs/core/src/guardian/mod.rs:62-77, 169-246`，`guardian/review.rs:321-343, 655-689, 977-990`

### 3a. 路由条件：仅 OnRequest/Granular + AutoReview

```rust
pub(crate) fn routes_approval_policy_to_guardian(
    approval_policy: AskForApproval, approvals_reviewer: ApprovalsReviewer
) -> bool {
    matches!(approval_policy, AskForApproval::OnRequest | AskForApproval::Granular(_))
        && approvals_reviewer == ApprovalsReviewer::AutoReview
}
```

**关键事实**（review.rs:209-217）：Never 和 UnlessTrusted 不走 Guardian——Never 直接执行，UnlessTrusted 必须人工审批。只有 OnRequest 和 Granular 且配置了 AutoReview 才进入 Guardian。

### 3b. Full Access 短路：has_full_access 直接 Approved

**锚点**：review.rs:330-343

```rust
if context.environments().has_full_access(
    context.approval_policy,
    &turn.config.permissions.effective_permission_profile(),
) {
    return if external_cancel...is_cancelled() { Abort } else { Approved };
}
```

**关键事实**：Full Access（Never + 无沙箱限制）完全跳过 Guardian 审查，直接返回 Approved。这是 Guardian 的 bypass 路径。

### 3c. 独立 review session：approval_policy=never + 无继承 exec-policy

**锚点**：review.rs:977-990（doc comment）

```
The guardian itself should not mutate state or trigger further approvals, so
it is pinned to a read-only sandbox with `approval_policy = never` and
nonessential agent features disabled. ... It may still reuse the parent's
managed-network allowlist for read-only checks, but it intentionally runs
without inherited exec-policy rules.
```

**关键事实**：
- Guardian 运行在**独立的 review session** 中，`approval_policy=never`（它自己不需要审批），且**不继承父 session 的 exec-policy 规则**（防止父 session 的免审规则影响 Guardian 的判断）。
- Guardian review session 是可复用的：cached trunk session idle 时后续审批追加到同一 conversation（preserve prompt-cache key）；trunk busy 时 fork ephemeral session（并行审批不互相阻塞）。
- `GUARDIAN_REVIEW_TIMEOUT = Duration::from_secs(90)`（mod.rs:62）。

### 3d. 上下文预算硬约束（5 层限流）

**锚点**：mod.rs:71-77

```rust
const GUARDIAN_MAX_MESSAGE_TRANSCRIPT_TOKENS: usize = 20_000;
const GUARDIAN_MAX_TOOL_TRANSCRIPT_TOKENS: usize = 10_000;
const GUARDIAN_MAX_MESSAGE_ENTRY_TOKENS: usize = 5_000;
const GUARDIAN_MAX_TOOL_ENTRY_TOKENS: usize = 1_000;
const GUARDIAN_RECENT_ENTRY_LIMIT: usize = 40;
```

**关键事实**：Guardian 的输入上下文有 5 层硬上限——总 message transcript 20K tokens、总 tool transcript 10K、单条 message 5K、单条 tool result 1K、最近 40 条 entry。这是上下文治理的具体实例。

### 3e. Fail-closed：超时/解析失败/会话失败 → Deny

**锚点**：review.rs:655-689

```rust
GuardianReviewError::PromptBuild { .. }
| GuardianReviewError::Session { .. }
| GuardianReviewError::Parse { .. } => {
    (GuardianAssessment {
        risk_level: GuardianRiskLevel::High,
        user_authorization: GuardianUserAuthorization::Unknown,
        outcome: GuardianAssessmentOutcome::Deny,
        rationale,
    }, false)  // false = 不计入熔断器
}
```

**关键事实**：
- Timeout → `ReviewDecision::TimedOut`（review.rs:616），不执行命令。
- PromptBuild/Session/Parse 错误 → 构造一个 `risk_level=High, outcome=Deny` 的 GuardianAssessment，等同于拒绝。
- **重试策略**：仅对 transient 错误重试（ServerOverloaded、HttpConnectionFailed、ResponseStreamConnectionFailed、InternalServerError、ResponseStreamDisconnected、Parse），最多 3 次（`GUARDIAN_REVIEW_MAX_ATTEMPTS=3`，review.rs:83），且在 90s deadline 内（review.rs:1098）。

### 3f. 拒绝熔断器：连续/近期拒绝 → 中断 turn

**锚点**：mod.rs:64-68, 196-246

```rust
const MAX_CONSECUTIVE_GUARDIAN_DENIALS_PER_TURN: u32 = 3;
const MAX_RECENT_AUTO_REVIEW_DENIALS_PER_TURN: u32 = 10;
const AUTO_REVIEW_DENIAL_WINDOW_SIZE: usize = 50;
const MAX_CONSECUTIVE_CYBER_GUARDIAN_DENIALS_PER_TURN: u32 = 1;
const MAX_RECENT_CYBER_AUTO_REVIEW_DENIALS_PER_TURN: u32 = 1;
```

**关键事实**：
- Standard 模型：连续 3 次拒绝或最近 50 次审查中有 10 次拒绝 → `InterruptTurn`（mod.rs:210-231）。
- Cyber 模型（`MODEL_SPECIALTY_CYBER`）：连续 1 次或近期 1 次 → 中断（review.rs:258-262）。
- 熔断器触发后发送 `GuardianWarning` 事件并 `abort_turn_if_active(TurnAbortReason::Interrupted)`（review.rs:281-307）。
- `record_non_denial` 重置 consecutive 计数（mod.rs:234-238）。

---

## EP-04：网络审批闭环 — 代理回调 + 策略固化

**子系统**：网络访问治理
**锚点**：`codex-rs/core/src/tools/network_approval.rs:1014-1030, 1032-1158`，`network_policy_decision.rs:74-102`，`exec_policy.rs:514-558`

### 4a. NetworkPolicyDecider：代理 allowlist-miss 回调

```rust
pub(crate) fn build_network_policy_decider(
    network_approval: Arc<NetworkApprovalService>,
    network_policy_decider_session: Arc<RwLock<std::sync::Weak<Session>>>,
) -> Arc<dyn NetworkPolicyDecider> {
    Arc::new(move |request: NetworkPolicyRequest| {
        async move {
            let Some(session) = network_policy_decider_session.read().await.upgrade() else {
                return NetworkDecision::ask("not_allowed");  // session 已死 → ask（保守）
            };
            network_approval.handle_inline_policy_request(session, request).await
        }
    })
}
```

**关键事实**（network_approval.rs:1014-1030）：
- 网络代理（codex-network-proxy）在遇到 allowlist 未命中的请求时，通过 `NetworkPolicyDecider` trait 回调到 core。
- 回调通过 `Weak<Session>` 持有 session 引用，避免循环引用；session 已释放时返回 `ask("not_allowed")`（保守降级）。
- `handle_inline_policy_request` 由 `NetworkApprovalService` 处理，决定 Allow/Deny/Ask。

### 4b. begin_network_approval：执行时网络代理构建

**锚点**：network_approval.rs:1032-1158

**关键事实**：
- `begin_network_approval` 在工具执行前构建 execution-scoped network proxy（network_approval.rs:1124-1136），将 `fallback_policy_decider`（即上面的 build_network_policy_decider）注入代理。
- `NetworkApprovalSpec`（network_approval.rs:52）携带 network、trigger、tool_name、command、environment_id、permission_profile、network_policy。
- `ActiveNetworkApproval` 持有 registration_id、cancellation_token、execution_proxy（network_approval.rs:1153-1157）。
- `register_call` 记录活跃网络审批调用（network_approval.rs:1138-1151），用于后续 finish 和审计。

### 4c. 网络规则固化：append_network_rule_and_update

**锚点**：exec_policy.rs:514-558，network_policy_decision.rs:74-102

**关键事实**：
- `execpolicy_network_rule_amendment`（network_policy_decision.rs:74-102）将 `NetworkPolicyAmendment`（Allow/Deny + host + protocol）转换为 `ExecPolicyNetworkRuleAmendment`，写入 `default.rules`。
- `append_network_rule_and_update`（exec_policy.rs:514-558）与命令前缀规则共享同一套 `update_lock + spawn_blocking + blocking_append_network_rule + 内存热更新` 机制。
- 被拒绝的网络请求通过 `build_blocked_request_observer`（network_approval.rs:1002-1012）记录为 `record_network_sandbox_violation` + `record_blocked_request`。

---

## EP-05：上下文治理硬约束（AGENTS.md Model visible context）

**子系统**：上下文/提示治理
**锚点**：`AGENTS.md:91-100`

```
### Model visible context

Codex maintains a context (history of messages) that is sent to the model in inference requests.

1. No history rewrite - the context must be built up incrementally.
2. Avoid frequent changes to context that cause cache misses.
3. No unbounded items - everything injected in the model context must have a bounded size and a hard cap.
4. No items larger than 10K tokens.
5. Highlight new individual items that can cross >1k tokens as P0. These need an additional manual review.
6. All injected fragments must be defined as structs in `core/context` and implement ContextualUserFragment trait
```

**关键事实**：
- 这是 6 条**硬约束**，不是建议。第 5 条明确 ">1K tokens 的新 context item 标记为 P0，需要额外人工审查"。
- 第 6 条要求所有注入 fragment 必须在 `core/context/` 定义为 struct 并实现 `ContextualUserFragment` trait——这是类型安全的上下文治理。
- `core/context/` 目录下有 50+ 个 context fragment 文件（如 `guardian_policy.rs`、`permissions_instructions.rs`、`rollout_budget.rs`、`token_budget_context.rs`），每个都是一个结构化的上下文注入单元。

---

## EP-06：RolloutBudget — 跨 agent 树的共享 token 预算治理

**子系统**：资源治理
**锚点**：`codex-rs/core/src/rollout_budget.rs:11-127`，`agent/control.rs:131, 148-167, 183-185`

### 6a. 数据结构

```rust
pub(crate) struct RolloutBudget {
    state: OnceLock<Mutex<RolloutBudgetState>>,
}
struct RolloutBudgetState {
    config: RolloutBudgetConfig,
    weighted_tokens_used: f64,
    deliveries: HashMap<ThreadId, ThreadBudgetDelivery>,  // 每线程最后提醒位置
}
```

**关键事实**（rollout_budget.rs:11-32）：
- `RolloutBudget` 通过 `Arc<RolloutBudget>` 在 `AgentControl` 中共享（agent/control.rs:131），因此**整个 agent 树（root + 所有 subagent）共享同一个 token 预算**。
- `OnceLock<Mutex<...>>` 延迟初始化：`configure` 在 `AgentControl::new` 中调用（agent/control.rs:163-165），未配置时 `lock()` 返回 None，所有方法 no-op。

### 6b. 加权计费

```rust
let units = if let Some(units) = usage.codex_rollout_budget_units.as_ref() {
    // 服务端返回的精确 units
    units.as_f64()...
} else {
    // 本地加权：output_tokens * sampling_token_weight + non_cached_input * prefill_token_weight
    usage.output_tokens.max(0) as f64 * state.config.sampling_token_weight
        + usage.non_cached_input() as f64 * state.config.prefill_token_weight
};
state.weighted_tokens_used += units;
Ok(state.weighted_tokens_used >= state.config.limit_tokens as f64)
```

**关键事实**（rollout_budget.rs:46-65）：
- 优先使用服务端返回的 `codex_rollout_budget_units`（精确计费），fallback 到本地加权（output × sampling_weight + non-cached-input × prefill_weight）。
- `record_usage` 返回 `bool`：预算是否已耗尽。耗尽后后续调用持续返回 true（latch 行为）。

### 6c. 提醒机制：阈值穿越 + 每线程去重

**锚点**：rollout_budget.rs:67-118

**关键事实**：
- `pending_reminder(thread_id, window_id)` 计算 `remaining_tokens`，然后数 `reminder_at_remaining_tokens` 中已穿越的阈值数量作为 `reminder_index`。
- 每线程记录 `ThreadBudgetDelivery { window_id, reminder_index }`，同一 window 内 index 不递增则不重复提醒（rollout_budget.rs:82-86）。
- `rearm_reminder(thread_id)` 清除该线程的 delivery 记录，强制下次请求重申剩余预算（rollout_budget.rs:112-118）。

---

## EP-07：PermissionProfileState — Session 级权限 SSOT

**子系统**：权限配置
**锚点**：`session/session.rs:96-99, 164-201, 360-502`

**关键事实**：
- `SessionConfiguration.permission_profile_state: PermissionProfileState` 是 session 级权限的单一事实源（session/session.rs:96-99），注释明确要求"constrained profile, active profile id, and profile-defined workspace roots in sync by using the methods below instead of mutating the fields independently"。
- `permission_profile()`（session/session.rs:164-166）从 state 快照。
- `effective_permission_profile()`（session/session.rs:177-190）优先使用选中 environment 的 config，fallback 到 materialized（含 workspace roots）。
- `apply()`（session/session.rs:360-502）在更新 permission_profile 时通过 `set_permission_profile_projection`（session/session.rs:504-538）保持三者同步，并保留 deny-read 限制（`preserve_deny_read_restrictions_from`）。

---

## EP-08：AgentControl — Weak 引用 + 执行限制器 + 共享状态

**子系统**：多 agent 控制面
**锚点**：`agent/control.rs:111-134, 146-173`

**关键事实**：
- `AgentControl.manager: Weak<ThreadManagerState>`（agent/control.rs:124）——注释明确说明这是为了避免引用循环 `ThreadManagerState → CodexThread → Session → SessionServices → ThreadManagerState`。
- `AgentControl` 实例在 root thread/session tree 中只创建一次，然后 clone 给所有 subagent（agent/control.rs:114-116），保持 registry  scoped to root thread。
- `agent_execution_limiter: Arc<AgentExecutionLimiter>`（agent/control.rs:129）在 `with_session_id` 中用 `effective_agent_max_threads` 初始化（agent/control.rs:169-173），限制并发 agent 数。
- `rollout_budget: Arc<RolloutBudget>`（agent/control.rs:131）和 `root_service_tier: Arc<ArcSwapOption<String>>`（agent/control.rs:133）都是跨 agent 树共享的。

---

## 覆盖矩阵（v1 遗漏检查）

| 子系统 | v1 覆盖 | v2 证据包 | 状态 |
|--------|---------|-----------|------|
| ExecApprovalRequirement 三态 | 部分（仅提及） | EP-01 | ✅ 补全 |
| exec_policy 持久化闭环 | ❌ 遗漏 | EP-02 | ✅ 新增 |
| Guardian 独立 session + fail-closed | ❌ 遗漏 | EP-03 | ✅ 新增 |
| 网络审批闭环 | ❌ 遗漏 | EP-04 | ✅ 新增 |
| 上下文治理硬约束 | ❌ 遗漏 | EP-05 | ✅ 新增 |
| rollout_budget 资源治理 | ❌ 遗漏 | EP-06 | ✅ 新增 |
| PermissionProfile SSOT | 部分 | EP-07 | ✅ 补全 |
| AgentControl Weak + limiter | 部分 | EP-08 | ✅ 补全 |
| AGENTS.md 治理规则 | ❌ 遗漏 | EP-05 | ✅ 新增 |

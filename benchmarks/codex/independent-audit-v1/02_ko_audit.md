# 02 · KO 审计（Source Truth Audit + Causal Integrity）

> 审计者独立核验产物。方法：先独立读代码（orchestrator.rs 全文 / session.rs / guardian/mod.rs / review.rs / sandboxing.rs / control.rs / exec_policy.rs / AGENTS.md），再逐条对照 wave3 中 7 个 KO 的 L1 事实声明与推导。
> 判定标准：仓库代码是最高证据。README/ADR/注释/已有 KO 都不能单独作为事实。
> 时间：2026-09-03

---

## 审计总览

| KO | Source Truth | Causal | Abstraction | Counterexample | Epistemic | 最终决策 |
|----|-------------|--------|-------------|----------------|-----------|---------|
| KO-01 | **CONTRADICTED（部分）** | PASS | VALID | found | Pattern→**降级为 Observation/Pattern(条件化)** | **PARTIALLY_CONFIRMED（需修正事实错误）** |
| KO-02 | **CONFIRMED** | PASS | VALID | none | Pattern | **CONFIRMED** |
| KO-03 | PARTIALLY_CONFIRMED | PASS | **OVERREACH** | found | Pattern→**降级为 Observation/Pattern(弱)** | **DOWNGRADED** |
| KO-04 | **CONFIRMED** | PASS | VALID | none | Pattern | **CONFIRMED** |
| KO-05 | PARTIALLY_CONFIRMED | PASS | VALID | none（条件需补充） | Principle→**条件化 Principle** | **PARTIALLY_CONFIRMED** |
| KO-06 | **CONFIRMED** | PASS | VALID | none | Pattern | **CONFIRMED（有覆盖缺口）** |
| KO-07 | **CONFIRMED** | PASS | VALID | none | Pattern | **CONFIRMED（需注明同源设计）** |

---

## KO-01 · Intelligence ≠ Authority（判断力与执行权分离）

```
knowledge_id: KO-01
claim: "模型产生 tool_call 的智能不直接拥有执行权，执行权由 Approval+Sandbox+Network Proxy 三重 Gate 控制；
       approval_policy 三态：Never(Full Access)/OnRequest(Guardian 独立 AI 审查自动批准)/AlwaysAsk(始终请求用户)"
original_level: L4
validated_level: L4（核心模型成立；但 L1 事实层存在明确错误）
```

### source_truth: PARTIALLY_SUPPORTED → 一处 CONTRADICTED（事实错误）

**证据（支持核心模型）：**
- `tools/sandboxing.rs:152-171` — `ExecApprovalRequirement` 三态（Skip/NeedsApproval/Forbidden）确实存在，对应 L1 "Approval Gate 三态" ✅
- `tools/orchestrator.rs:173-230` — Approval Gate 驱动三态分支 ✅
- `tools/orchestrator.rs:274` — `SandboxManager.select_initial` 沙箱选择 ✅
- `guardian/mod.rs:62` — GUARDIAN_REVIEW_TIMEOUT=90s ✅
- `guardian/mod.rs:12-13` — fail closed ✅

**证据（反驳 L1 事实声明）：**
- ❌ **"approval_policy 三态"是事实错误**。真实 `AskForApproval` 是**四态**：`Never / OnRequest / Granular(_) / UnlessTrusted`（`tools/sandboxing.rs:189-207` 注释明确列出四态语义；`exec_policy.rs:49-53` 也出现 `AskForApproval::Granular.sandbox_approval` 与 `.rules` 两个子开关）。
  - "AlwaysAsk" 这个名字**不存在**，真实变体叫 `UnlessTrusted`（语义也不等价——`UnlessTrusted` 在 `default_exec_approval_requirement` 中恒返回 needs_approval=true，即"始终问"，但命名语义与"trusted 上下文"相关，不是简单的"AlwaysAsk"）。
  - KO-01 完全漏掉了 **Granular 态**（粒度审批，带 sandbox_approval + rules 两个子开关，关闭时直接 Forbidden 而非问用户）。
- ❌ **"OnRequest = Guardian 独立 AI 审查自动批准" 是条件性简化**：
  - `tools/sandboxing.rs:194-207` — OnRequest 的 needs_approval 取决于 `file_system_sandbox_policy.kind == Restricted`；文件系统不受限时 OnRequest 走 Skip（不问、不 Guardian）。
  - `guardian/review.rs:209-217` — `routes_approval_policy_to_guardian` 仅当 `approval_policy ∈ {OnRequest, Granular}` **且** `approvals_reviewer == AutoReview` 才路由到 Guardian；`Never` 和 `UnlessTrusted` **不经过 Guardian**（前者直接批准，后者直接给用户）。"OnRequest→Guardian" 缺少 `approvals_reviewer==AutoReview` 这一必需条件。

### causal_integrity: PASS
"因为 Agent 判断不可靠，所以判断力与执行权分离" — 因果链成立，且被 Guardian fail closed 设计佐证。判断层（模型）与执行层（Gate 链）在代码中确实物理分离。

### abstraction: VALID
L4 "Intelligence ≠ Authority" 解释范围确实扩大（工具执行/Guardian/网络访问三个机制都指向同一关系）。非 L4_OVERREACH。

### counterexamples: found（配置驱动的 bypass 路径）
- `tools/orchestrator.rs:233-234, 266, 280-282` — `unsandboxed_allowed && BypassSandboxFirstAttempt` 时**首次尝试即无沙箱**执行（不经过 Sandbox Gate）。
- `tools/sandboxing.rs:250-260` — ExecPolicy `Allow` 可产生 `Skip { bypass_sandbox: true }`，显式绕过沙箱。
- `exec_policy.rs:48-53` — `AskForApproval::Never` 时策略要求审批 → 冲突拒绝（说明存在"策略要审批但策略级 Never 放行"的张力）。
- **结论**：默认/受管路径确实多 Gate；但存在**配置驱动的无沙箱路径**（bypass 由策略显式授权）。KO-01 的 L5.5 "降级必须触发额外审批" 不完全成立——BypassSandboxFirstAttempt 是**首次**就无沙箱，不是"降级"。

### coverage
- independent_support: 核心分离模型被代码强支持（Approval+Sandbox 两重 Gate 明确；Network Proxy 仅网络审批路径出现，非所有执行）
- independent_contradiction: approval_policy 枚举、OnRequest 路由条件、bypass 首次无沙箱路径

### epistemic_status
- original: Validated Pattern（codex 内多源）
- validated: **Observation / Pattern（条件化）** — 需在 claim 中写明"受管默认路径"边界；"AlwaysAsk 三态"错误需修正为四态

### scope
- valid_when: 受管默认执行路径（approval_policy∈{OnRequest,Granular}+AutoReview 或 Never 白名单、无显式 bypass 策略）
- invalid_when: 配置了 ExecPolicy Allow(bypass_sandbox=true) 或 BypassSandboxFirstAttempt 的工具调用（存在无沙箱直通路径）；UnlessTrusted 下的非 Guardian 人工路径

### final_decision: **PARTIALLY_CONFIRMED（修正 L1 事实错误后 CONFIRMED）**

---

## KO-02 · 声明 ≠ 证据（AI 审批的证据生产链）

```
knowledge_id: KO-02
claim: "AI 对 AI 的审批不能仅凭声明被信任，必须有可审计证据生产链（transcript 限流/结构化评估/90s 超时 fail closed/熔断器/延迟确认）"
original_level: L4
validated_level: L4
```

### source_truth: SUPPORTED（全部 L1 事实经代码确认）
- ✅ `guardian/mod.rs:71-77` — GUARDIAN_MAX_MESSAGE_TRANSCRIPT_TOKENS=20_000 / TOOL=10_000 / MESSAGE_ENTRY=5_000 / TOOL_ENTRY=1_000 / RECENT_ENTRY_LIMIT=40（与 KO-02 数值完全一致）
- ✅ `guardian/mod.rs:162-167` — GuardianAssessment 结构化（risk_level/user_authorization/outcome/rationale）
- ✅ `guardian/mod.rs:62, 12-13` — 90s 超时 + fail closed（timeout/execution failure/malformed output）
- ✅ `guardian/mod.rs:64-68, 196-232` — GuardianRejectionCircuitBreaker：标准策略连续 3 次或最近 50 次中 10 次 → InterruptTurn；CyberModel 连续 1 次即中断
- ✅ `tools/orchestrator.rs:66-123` — 网络审批 begin→ActiveNetworkApproval(proxy+cancellation_token)→执行→Deferred→finish（延迟确认）

### causal_integrity: PASS
"因为 Guardian 也是 AI 也可能错，所以不能把它的声明当证据" → 证据生产链设计。因果成立。

### abstraction: VALID
L4 "声明≠证据；信任=证据链完整性与真实性" 解释了 transcript 限流/结构化/超时/熔断/延迟确认 5 个机制，解释范围确实扩大。

### counterexamples: none found
未发现绕过证据生产链的路径（熔断器、超时、结构化校验均强制）。

### coverage（补充而非矛盾）
- 触发 CyberModel 的条件是 `model_info().model_specialty == Some("cyber")`（`guardian/review.rs:258-262`），KO-02 未说明触发条件（不构成错误，但 scope 需注明 CyberModel 非所有模型生效）。
- `guardian/review.rs:209-217` — Guardian 仅在 OnRequest/Granular+AutoReview 时被路由；"声明≠证据"的适用范围应限定在 auto-review 路径。

### epistemic_status
- original: Validated Pattern
- validated: Validated Pattern ✅（证据链完整，多源代码确认）

### scope
- valid_when: 自动审批/自动决策路径（Guardian auto-review 生效时）
- invalid_when: 纯人工审批（人工直接判断不经证据链）；Never 白名单直接批准路径

### final_decision: **CONFIRMED**

---

## KO-03 · 自举系统的递归矛盾与显式逃逸阀

```
knowledge_id: KO-03
claim: "Agent 运行在沙箱中又管理沙箱，形成自举递归；CODEX_SANDBOX* 环境变量是打破递归的显式逃逸阀（外部设置/内部只读/禁止修改）"
original_level: L3
validated_level: L2（条件化的环境隐式契约）→ 建议 DOWNGRADED 或改写 claim
```

### source_truth: PARTIALLY_SUPPORTED
- ✅ `AGENTS.md:8` — "Never add or modify any code related to CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR or CODEX_SANDBOX_ENV_VAR"
- ✅ `AGENTS.md:9-10` — 运行时确实设置这些环境变量（shell tool 时 CODEX_SANDBOX_NETWORK_DISABLED=1；spawn Seatbelt 时 CODEX_SANDBOX=seatbelt）；代码/测试用它们提前退出
- ⚠️ "Agent 自身运行在沙箱中，但功能包括管理沙箱 → 形成递归" — 部分成立但被夸大：
  - 代码中的实际用途是**测试环境检测/跳过**（"often used to early exit out of tests"），不是"递归点的逃逸"。
  - "递归"的表述暗示这是一种结构性自举矛盾；实际是**受限运行环境的隐式契约**（外部运行时把约束通过环境变量告知内部代码），递归性只在"沙箱内 spawn 沙箱"的测试场景短暂存在。

### causal_integrity: PASS
因果链（运行时设变量 → 内部只读 → 用于跳过受限测试）在 AGENTS.md 中明确。但"递归矛盾"是解释层的过度演绎。

### abstraction: OVERREACH（核心问题）
- L3 判定：同类对照（编译器 stage0/VM host-guest/Docker socket）描述的是**真正的结构递归**（管理者逻辑上被自己管理）。codex 的 CODEX_SANDBOX* 更接近"测试环境感知 / 特性检测"，**不具备同类递归结构**。解释范围并未真正扩大，而是用一个更强的模型（自举递归）去套一个较弱的机制（环境隐式契约）。
- 建议：改写为 L2/L3 弱版 —— "**被管环境的隐式契约**：运行时通过外部设置的只读信号告知代码当前环境约束，代码据此降级或跳过，且该信号被文档化禁止修改"。若保留"递归/逃逸阀"，证据不足，应进 Candidates（C-xxx）。

### counterexamples: found
- AGENTS.md 的 CODEX_SANDBOX* 约束只作用于 **shell/测试环境**；Codex 的沙箱策略核心（PermissionProfile/ExecPolicy/Approval Gate）**完全不依赖**这两个环境变量——沙箱管理并不通过该"逃逸阀"实现。即"逃逸阀打破递归"在核心沙箱逻辑中没有对应实现，仅存在于测试跳过层。

### epistemic_status
- original: Validated Pattern
- validated: **Observation（弱 Pattern）** — 单一证据（AGENTS.md 测试跳过）不足以支撑 L3 递归模式；递归模型未验证

### scope
- valid_when: 受限环境中的测试/运行降级检测（环境隐式契约成立）
- invalid_when: 作为"自举系统的通用逃逸阀"推广到所有沙箱管理场景

### final_decision: **DOWNGRADED（L3→L2 弱模式；"递归矛盾"模型进 Candidates）**

---

## KO-04 · 单一真相源与派生策略的一致性治理

```
knowledge_id: KO-04
claim: "PermissionProfile 是沙箱策略单一真相源，file_system/network/sandbox_policy 均从中派生；三字段必须经方法同步；支持环境级覆盖"
original_level: L3
validated_level: L3
```

### source_truth: SUPPORTED（全部代码确认）
- ✅ `session/session.rs:212-221` — `sandbox_policy()` 从 `materialized_permission_profile` 派生
- ✅ `session/session.rs:223-229` — `file_system_sandbox_policy()` 从 permission_profile 派生
- ✅ `session/session.rs:231-235` — `network_sandbox_policy()` 从 permission_profile 派生
- ✅ `session/session.rs:96-99` — "Keep the constrained profile, active profile id, and profile-defined workspace roots in sync by using the methods below instead of mutating the fields independently"
- ✅ `session/session.rs:177-190` — effective_permission_profile 优先环境配置，否则 session materialize（环境级覆盖）

### causal_integrity: PASS
"多维度策略独立配置会冲突 → 单一真相源 + 纯函数派生" 因果成立，且被代码注释与三处派生函数证实。

### abstraction: VALID
L3 "SSOT + 显式派生函数" 解释范围确实扩大（三策略维度 + 环境覆盖层级）。非 OVERREACH。

### counterexamples: none found
未发现绕过 permission_profile 直接配置某维度的路径（全部派生函数集中）。TODO(session.rs:102-103) 提到 legacy thread defaults 与 TurnEnvironment::sandbox_context 需 reconcile——存在已知的技术债，但不推翻 SSOT claim。

### coverage（补充）
- `session/session.rs:48-49` — "Keep this separate from `state` so storage I/O does not block runtime state access"（thread_settings_persistence 与 runtime state 分离）是**独立且相关**的架构决策（持久化与运行时状态分离），KO-04 未提取，可并入或单列。

### epistemic_status
- original: Validated Pattern
- validated: Validated Pattern ✅

### scope
- valid_when: 多维度策略/配置系统
- invalid_when: 单一维度无派生关系的配置

### final_decision: **CONFIRMED（可补充"持久化与运行时状态分离"证据）**

---

## KO-05 · Fail Closed 作为 AI 系统的安全默认

```
knowledge_id: KO-05
claim: "AI 系统所有安全关键 Gate 不确定时默认拒绝（Guardian 超时/失败/格式错误、沙箱升级严格条件、附件网络策略不可绕过、多 Agent 超限、熔断器）"
original_level: L4
validated_level: L4（条件化）
```

### source_truth: PARTIALLY_SUPPORTED（各实例多数成立，一处条件列举不完整）
- ✅ `guardian/mod.rs:12-13` — fail closed（timeout/execution failure/malformed）
- ✅ `tools/orchestrator.rs:149-159` — attachment-owned network policy 不可被沙箱升级绕过
- ✅ `agent/control.rs:169-171` — `agent_execution_limiter.initialize(max_threads)`；`control_tests.rs:3009-3015` 确认 `AgentLimitReached`（⚠️ KO-05 引用位置 execution.rs:44-60 与实测不符，实际 limiter 在 agent/control.rs，位置引用需修正）
- ✅ `guardian/mod.rs:196-232` — 熔断器 InterruptTurn
- ⚠️ "沙箱拒绝后升级重试条件：escalate_on_failure + unsandboxed_allowed + approval_policy 允许 + 非 strict_auto_review" — **列举不完整**。真实条件（orchestrator.rs:317-512 全文核验）还包含：
  - `network_approval_context.is_none()`（orchestrator.rs:342-351, 409-411）——网络审批上下文已建立时，沙箱外重试行为不同/需 fresh review
  - `tool.should_bypass_approval(approval_policy, already_approved)`（orchestrator.rs:409-411）——bypass_retry_approval 还依赖该函数
  - `unsandboxed_allowed = !owner_network_policy && unsandboxed_execution_allowed(...)`（orchestrator.rs:233-234）——owner_network_policy 也是条件
  - "任一不满足保持拒绝" 的结论方向正确，但条件清单需补全，否则读者会误以为条件只有 4 个。

### causal_integrity: PASS
"误放代价 >> 误拒代价 → fail closed" 因果成立，且被多实例证实。

### abstraction: VALID
L4 "fail closed 是对智能不确定性的结构性回应" 解释范围确实扩大（5 个实例）。非 OVERREACH。

### counterexamples: none found（但需边界化）
- fail closed 的例外是**显式白名单**：`AskForApproval::Never` + `ExecPolicy Allow(bypass_sandbox=true)` 是"用户显式授权放行"，不构成对 fail closed 的反例，但说明 fail closed 是**默认**而非绝对——KO-05 L5.4 已承认白名单机制，需在 claim 中强调"默认路径"边界。

### epistemic_status
- original: Principle（codex 内验证）
- validated: **条件化 Principle** — "AI 系统安全关键 Gate 应 fail closed"作为方法论成立（codex 多实例 + 泛化逻辑），但 claim 中需写明：仅在受管默认路径生效、显式授权（Never/Allow bypass）是已知例外。

### scope
- valid_when: 安全关键 Gate 的默认行为（不确定时）
- invalid_when: 用户显式白名单（Never/ExecPolicy Allow）；非安全关键路径

### final_decision: **PARTIALLY_CONFIRMED（补全升级条件清单 + 修正位置引用后 CONFIRMED）**

---

## KO-06 · 大型代码库的核心模块反膨胀治理

```
knowledge_id: KO-06
claim: "核心模块膨胀是路径依赖+正反馈的结构性倾向，必须通过显式治理规则+code review 对抗"
original_level: L3
validated_level: L3
```

### source_truth: SUPPORTED（全部代码/文档确认）
- ✅ `AGENTS.md:72-74` — codex-core bloated 原因自述
- ✅ `AGENTS.md:76-83` — "resist adding code to codex-core" + 新建 crate 建议 + code review push back
- ✅ `session/session.rs:42-78` — Session struct 约 30 字段（"20+" 成立）

### causal_integrity: PASS
"添加到 core 比重构更容易 → 路径依赖正反馈 → 需显式治理" 因果成立，且 AGENTS.md 明确自述。

### abstraction: VALID（但覆盖不完整）
L3 "模块边界治理" 成立。**关键覆盖缺口**：AGENTS.md 还存在**两个同族但独立的治理机制**未被 KO-06 提取：
- `AGENTS.md:49-53` — **模块级行数治理**：Rust modules <500 LoC 目标；>800 LoC 时新建模块；列出高触碰文件黑名单（tui/src/app.rs 等）
- `AGENTS.md:125-131` — **变更大小治理**：非机械变更总行数 ≤800；复杂逻辑 ≤500；超限需分阶段
- `AGENTS.md:87-89` — **crate API surface 最小化**（避免 test-only helpers 泛滥）
这三个机制与 crate 反膨胀构成一个**多层级膨胀治理体系**（crate 级 / 模块级 / 变更级 / API 级）。KO-06 只覆盖 crate 级，是"提取了局部而非整体"——需要补充或单列知识对象。

### counterexamples: none found
未发现治理失效的路径（但 472 文件本身说明治理是"对抗性"而非"已解决"——治理规则的长期有效待验证）。

### epistemic_status
- original: Validated Pattern
- validated: Validated Pattern（但表述需扩展为多层级治理）

### scope
- valid_when: 大型代码库（核心模块 >100 文件）
- invalid_when: 小型项目

### final_decision: **CONFIRMED（建议扩展为"多层级膨胀治理体系"以覆盖模块级/变更级/API 级）**

---

## KO-07 · 产生→验证→授权→执行→记录的五段式架构指纹

```
knowledge_id: KO-07
claim: "Codex 中 4 处独立出现产生→验证→授权→执行→记录五段式结构，是可审计代理系统的架构指纹"
original_level: L4
validated_level: L4（需注明四实例同源设计）
```

### source_truth: SUPPORTED（4 实例均经代码确认）
- ✅ F-01 工具执行（orchestrator.rs 全文：Approval→Sandbox→run→Event）
- ✅ F-04 Guardian 审批（guardian/mod.rs：approval→transcript 验证→Guardian 授权→放行→熔断器记录）
- ✅ 网络访问（orchestrator.rs:66-123：begin→ActiveNetworkApproval→执行→Deferred→finish）
- ✅ Session 初始化（session.rs：配置产生→并行 setup→权限授权→Session→SessionConfigured Event）

### causal_integrity: PASS
"每段独立关注点，分离后可独立审计/升级/失败" 因果成立。

### abstraction: VALID（但需 epistemic nuance）
L4 "架构指纹" 解释范围确实扩大（4 机制）。但需注意：**4 处实例高度同源**——它们很可能都是同一架构理念（Gate 链/关注点分离）在作者刻意设计下的多点实例化，而非 4 个独立演化出的自然涌现结构。作为 observed pattern 成立，但"指纹"一词暗示自然涌现/普遍性，需限定为"该架构团队的设计签名"。

### counterexamples: none found
五段式在 4 处均成立。未发现缺段路径（绕过记录段的直通路径不存在——所有执行都有 Event/审计记录）。

### coverage（补充）
- 五段式的**记录段**完整性是 L5.5 的核心，但 KO-07 未充分展开"记录段不可篡改"的实现证据（Event 系统 / audit trail）。可补充 session event 系统的证据强度。

### epistemic_status
- original: Validated Pattern（codex 内 4 处）
- validated: Validated Pattern（注明同源设计签名后成立）

### scope
- valid_when: 需要可审计性的代理/自动化系统
- invalid_when: 一次性脚本

### final_decision: **CONFIRMED（补充"同源设计"标注）**

---

## 审计者补充的核心事实核验摘要

除 KO 对照外，本次独立核验确认以下**仓库事实**（供 coverage/abstraction/flow 审计引用）：
1. `AskForApproval` 四态：Never / OnRequest / Granular(_) / UnlessTrusted（sandboxing.rs:189-207）
2. Guardian 路由条件：OnRequest|Granular + approvals_reviewer==AutoReview（review.rs:209-217）
3. 沙箱外重试条件含 network_approval_context.is_none() + should_bypass_approval（orchestrator.rs:342-351, 409-411）
4. exec_policy 完整子系统：ExecPolicyManager / ModelPolicy / ExecutableIdentity / 危险命令黑名单 / blocking_append_allow_prefix_rule（exec_policy.rs）
5. 上下文 6 条硬限制：无重写/缓存友好/有界/10K cap/1K P0 审查/ContextualUserFragment trait（AGENTS.md:93-100）
6. 持久化与运行时状态分离：thread_settings_persistence 独立于 state（session.rs:48-49）
7. AgentControl 用 Weak<ThreadManagerState> 避免引用循环与隐式持久化（control.rs:122-123）
8. rollout_budget 多 Agent 预算机制（control.rs:131, 163-165）
9. 多层级膨胀治理：crate 级（AGENTS.md:72-83）/ 模块级（:49-53）/ 变更级（:125-131）/ API surface（:87-89）

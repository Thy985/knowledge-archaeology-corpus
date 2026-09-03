# 03 · Coverage Audit（覆盖缺口）

> 方法：将审计者的 Independent Findings（盲重建的独立观察 + 代码核验）与已有 7 个 KO 对比。
> 判定：YES（已有 KO 覆盖）/ PARTIAL（部分覆盖）/ MISSING（完全遗漏）。
> 重点找 Critical Knowledge Missing（被遗漏的重要机制），不把小实现细节同等对待。

---

## 审计者独立发现清单（来源：代码/AGENTS.md 第一手核验）

> 注：本清单为审计者在**独立阅读代码**过程中形成的观察（部分早于读取 wave3 全文），与盲审 SubAgent 的完全隔离重建互为独立来源。

| # | Independent Finding | 证据 | 已有 KO 覆盖 |
|----|--------------------|------|-------------|
| IF-01 | `AskForApproval` 是**四态**（Never/OnRequest/**Granular**/UnlessTrusted），Granular 带 sandbox_approval+rules 子开关 | sandboxing.rs:189-207, exec_policy.rs:49-53 | KO-01 有但**表述错误**（三态）→ PARTIAL/错误 |
| IF-02 | Guardian 路由条件：OnRequest\|Granular + AutoReview（Never/UnlessTrusted 不经 Guardian） | review.rs:209-217 | KO-01 PARTIAL（无条件化）；KO-02 未限范围 |
| IF-03 | 沙箱外重试条件含 network_approval_context.is_none() + should_bypass_approval + owner_network_policy + wants_no_sandbox_approval | orchestrator.rs:233-234, 342-351, 362-386, 409-411 | KO-05 PARTIAL（条件不完整） |
| IF-04 | **exec_policy 完整子系统**：ExecPolicyManager + ModelPolicy + ExecutableIdentity + 危险命令黑名单 + blocking_append_allow_prefix_rule（审批→策略固化闭环） | exec_policy.rs 全文 | **MISSING**（重大） |
| IF-05 | **上下文 6 条硬限制**：无重写/缓存友好/有界/10K cap/1K P0 审查/ContextualUserFragment trait | AGENTS.md:93-100 | **MISSING**（重大） |
| IF-06 | 持久化与运行时状态分离：thread_settings_persistence 独立于 state | session.rs:48-49 | **MISSING** |
| IF-07 | AgentControl 用 Weak<ThreadManagerState> 避免引用循环与隐式持久化 | control.rs:122-123 | **MISSING** |
| IF-08 | rollout_budget 多 Agent 预算机制 | control.rs:131, 163-165 | **MISSING** |
| IF-09 | 多层级膨胀治理：crate 级(72-83)/模块级(49-53)/变更级(125-131)/API surface(87-89) | AGENTS.md | KO-06 PARTIAL（只覆盖 crate 级） |
| IF-10 | `proposed_execpolicy_amendment`：审批通过后可提议"未来跳过类似命令审批"（审批→策略学习的反馈闭环） | sandboxing.rs:158-167, exec_policy.rs:20 | **MISSING**（重大） |
| IF-11 | CyberModel 触发条件：model_specialty==cyber（非所有模型） | review.rs:258-262 | KO-02 未说明（不构成错误） |
| IF-12 | 附件持有网络策略时沙箱升级恒不可能 | orchestrator.rs:149-159 | KO-05 ✅ 已覆盖 |

---

## 覆盖判定

### Critical Knowledge Missing（最重要的遗漏）

1. **IF-04 · exec_policy 执行策略系统（MISSING，严重）**
   - 这是 Codex 权限系统的**另一个支柱**（与 PermissionProfile 并列）：`ExecPolicyManager` 用 rules 文件（default.rules）管理命令级审批策略，`ExecutableIdentity` 识别可执行文件身份，`ModelPolicy` 提供模型级策略，`dangerous_command_match` + `BANNED_PREFIX_SUGGESTIONS`（bash -c / node -e / python -c 等黑名单）用于危险命令检测。
   - `blocking_append_allow_prefix_rule` + `proposed_execpolicy_amendment` 构成**审批→策略固化**的反馈闭环：用户/Guardian 批准一条命令后，可提议把该命令加入 allow 规则，未来同类命令免审。这是"从逐次审批演进到策略化"的机制，是 Agent 权限系统最可迁移的设计之一。
   - 7 个 KO 完全未提及 exec_policy（KO-01 只提"三重 Gate"，未提命令级策略层；KO-04 只提 PermissionProfile）。**这是本次审计发现的最大覆盖缺口。**

2. **IF-05 · 模型可见上下文治理（MISSING，严重）**
   - AGENTS.md:93-100 定义了 6 条硬限制：①无历史重写（增量构建）②避免频繁变更导致缓存 miss ③所有注入项有界 + 硬 cap ④单条 ≤10K tokens ⑤>1K 的新项标记 P0 需人工审查 ⑥所有注入片段必须是 ContextualUserFragment trait 的实现。
   - 这是"Agent 系统如何治理喂给模型的上下文"的系统性规则，直接关系到 Agent 的长期会话质量、成本与行为稳定性。对任何做长上下文 Agent 的项目都是高价值知识。
   - 7 个 KO 完全未覆盖（KO-02 的 transcript 限流是 Guardian 审批的，不是主会话上下文的）。**重大遗漏。**

3. **IF-10 · 审批→策略固化闭环（MISSING，严重）**
   - 与 IF-04 关联。`ExecApprovalRequirement` 的 Skip/NeedsApproval 都携带 `proposed_execpolicy_amendment`——即"这个命令被批准后，可以把它固化成策略，未来同类免审"。这是把一次性审批变成持久策略的机制。
   - 涉及一个重要的 Agent 工程问题：**如何让系统从"逐次确认"平稳过渡到"策略化放行"而不失控**。未覆盖。

### 中等/次要遗漏

4. **IF-06 · 持久化与运行时状态分离** — Session 把 `thread_settings_persistence`（Semaphore）与 `state` 分开（session.rs:48-49），避免存储 I/O 阻塞运行时状态访问。这是"IO 与状态分离"的实例，可归入 KO-04 或单列为 L2/L3。
5. **IF-07 · Weak 引用避免引用循环** — AgentControl 用 `Weak<ThreadManagerState>` 避免 `ThreadManagerState→CodexThread→Session→SessionServices→ThreadManagerState` 引用循环与隐式持久化（control.rs:122-123）。经典 Rust 架构决策，可迁移。
6. **IF-08 · rollout_budget** — 多 Agent 的 rollout 预算机制（每轮可用资源预算），是"多 Agent 资源治理"的实例。
7. **IF-09 · 多层级膨胀治理** — KO-06 只提取了 crate 级，遗漏模块级（500/800 LoC）+ 变更级（800/500 行）+ API surface 三个同族机制。应扩展。

### 已被现有 KO 充分覆盖的独立发现（确认无遗漏）

- IF-12（附件网络策略不可绕过）→ KO-05 ✅
- Guardian fail closed / transcript 限流 / 熔断器 / 延迟确认 → KO-02 ✅
- PermissionProfile SSOT → KO-04 ✅
- 五段式重复结构 → KO-07 ✅

---

## 覆盖缺口汇总

| 严重度 | 独立发现 | 判定 |
|--------|---------|------|
| **严重** | IF-04 exec_policy 执行策略系统 | **MISSING** |
| **严重** | IF-05 模型可见上下文治理 | **MISSING** |
| **严重** | IF-10 审批→策略固化闭环 | **MISSING** |
| 中等 | IF-06 持久化/运行时分离 | MISSING |
| 中等 | IF-07 Weak 引用 | MISSING |
| 中等 | IF-08 rollout_budget | MISSING |
| 中等 | IF-09 多层级膨胀治理 | PARTIAL（KO-06 只有 crate 级） |
| 修正 | IF-01 四态 | KO-01 错误 |
| 边界 | IF-02/03 Guardian 路由与升级条件 | PARTIAL |
| 确认 | IF-11/12 | ✅ 已覆盖 |

**结论**：7 个 KO 对"安全 Gate / 证据链 / SSOT"方向覆盖扎实，但**遗漏了 exec_policy 策略系统、上下文治理、策略固化闭环这三个 Codex 真正有特色的工程机制**——这些恰恰是 Codex 区别于普通"加 Gate 的 Agent"的关键。这是 Critical Knowledge Missing，应在新一轮提取中补充为 KO-08/09/10。

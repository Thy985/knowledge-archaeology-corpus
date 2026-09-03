# 06 · Counterexample Hunt（反例猎杀）

> 目的不是证明已有知识正确，而是尝试杀死它。
> 对每个高价值 claim 的绝对化表述（always/must/all/never/the system requires）主动搜索 bypass / exception / override / admin path / fallback / direct call / legacy path / special case / test-only path。

---

## CE-01 · 攻击 KO-01 "执行权由三重 Gate 控制" → 找到无沙箱直通路径

- **攻击目标**：KO-01 L1.23 "模型不直接拥有执行权，执行权由 Approval + Sandbox + Network Proxy 三重 Gate 控制"；L5.5 "降级必须触发额外审批"。
- **反例证据**：
  1. `tools/orchestrator.rs:233-234, 266, 280-282` — `unsandboxed_allowed && BypassSandboxFirstAttempt` 时**首次尝试即无沙箱执行**（不经 Sandbox Gate）。
  2. `tools/sandboxing.rs:250-260` — ExecPolicy `Allow` 可产生 `Skip { bypass_sandbox: true }`，**策略级显式授权无沙箱**。
  3. `agent/control.rs` / `guardian/review.rs` — Guardian 审批和工具执行是两条**独立路径**（工具执行不一定经过 Guardian，取决于 approval_policy + reviewer）。
- **结论**：默认/受管路径确实多 Gate，但存在**配置驱动的 bypass**（策略显式 Allow 或 BypassSandboxFirstAttempt）。这不是"绕过安全"——是**策略显式授权的白名单路径**。因此 KO-01 的 claim 必须**边界化**："受管默认路径受多重 Gate 控制；显式授权（ExecPolicy Allow / Never 白名单 / BypassSandboxFirstAttempt）可构造无沙箱直通路径"。
- **处置**：**修改适用范围**，不删除 KO-01（核心模型仍成立）。

## CE-02 · 攻击 KO-01 "approval_policy 三态" → 四态反例

- **攻击目标**：KO-01 L1.21 "approval_policy 三态：Never/OnRequest/AlwaysAsk"。
- **反例证据**：`tools/sandboxing.rs:189-207` — `AskForApproval` 四态：Never / OnRequest / **Granular(_)** / **UnlessTrusted**。其中：
  - Granular 带 `sandbox_approval` 和 `rules` 两个子开关（exec_policy.rs:50-53），关闭时直接 Forbidden（不是问用户，也不是 Guardian）。
  - "AlwaysAsk" 不存在，真实变体名 "UnlessTrusted"。
- **结论**：**CONTRADICTED**（事实错误）。必须修正为四态，并补 Granular 子开关语义。
- **处置**：**事实修正**，不改核心模型。

## CE-03 · 攻击 KO-01 "OnRequest = Guardian 独立 AI 审查自动批准" → 条件性

- **攻击目标**：KO-01 对 OnRequest 的描述。
- **反例证据**：
  1. `guardian/review.rs:209-217` — `routes_approval_policy_to_guardian` 仅当 OnRequest/Granular **且** `approvals_reviewer == AutoReview` 才走 Guardian。OnRequest + 非 AutoReview（如人工 reviewer）→ 直接给用户，不走 Guardian。
  2. `tools/sandboxing.rs:194-207` — OnRequest 的 needs_approval 取决于 `file_system_sandbox_policy.kind == Restricted`；文件系统不受限时 OnRequest 走 Skip（不审批）。
- **结论**：OnRequest → Guardian 是**条件性**的（需 AutoReview + 文件系统 Restricted）。KO-01 描述为无条件 → 需修正。
- **处置**：**条件化修正**。

## CE-04 · 攻击 KO-02 "延迟确认模式" → 可取消性边界

- **攻击目标**：KO-02 L5.5 "延迟确认模式：先放行执行但保持可取消，执行后最终确认"。
- **反例/边界**：`tools/orchestrator.rs:66-123` — 延迟确认仅用于**网络审批**（NetworkApproval 有 proxy + cancellation_token）。非网络操作（普通 shell/文件工具）不适用延迟确认——它们是同步审批或 fail closed。因此"延迟确认"是网络代理特化的模式，**不能推广为所有执行路径的通用模式**。
- **处置**：**限定适用范围**（仅可取消/可代理的网络型操作）；非反例但需边界化。

## CE-05 · 攻击 KO-05 "沙箱升级条件" → 条件清单不完整

- **攻击目标**：KO-05 L1.2 "升级重试条件：escalate_on_failure + unsandboxed_allowed + approval_policy 允许 + 非 strict_auto_review，任一不满足保持拒绝"。
- **反例/补充证据**（orchestrator.rs:317-512 全文核验）：
  1. `network_approval_context.is_none()`（orchestrator.rs:342-351）— 网络审批上下文已建立时，无沙箱重试行为不同（可能直接拒绝）。
  2. `tool.should_bypass_approval(approval_policy, already_approved)`（orchestrator.rs:409-411）— bypass_retry_approval 依赖该函数。
  3. `unsandboxed_allowed = !owner_network_policy && unsandboxed_execution_allowed(...)`（orchestrator.rs:233-234）— owner_network_policy 也参与。
  4. `!tool.wants_no_sandbox_approval(approval_policy)`（orchestrator.rs:362-386）— Under Never/OnRequest 时 `wants_no_sandbox_approval` 为真则不做无沙箱重试。
- **结论**：KO-05 列举的 4 条件是**必要但不完整**。方向正确（任一不满足保持拒绝）但读者会误以为条件只有 4 个。
- **处置**：**补全条件清单**。

## CE-06 · 攻击 KO-03 "自举递归逃逸阀" → 核心沙箱不依赖逃逸阀

- **攻击目标**：KO-03 L4 "被管理者同时是管理者的宿主，必须存在显式逃逸阀打破递归"。
- **反例证据**：
  1. Codex 的沙箱策略核心（PermissionProfile / ExecPolicy / Approval Gate / SandboxManager）**完全不依赖** CODEX_SANDBOX* 环境变量——沙箱管理不通过"逃逸阀"实现。
  2. CODEX_SANDBOX* 仅作用于测试/早期退出层（AGENTS.md:9-10 "often used to early exit out of tests"）。
  3. 找不到"递归矛盾"在核心沙箱逻辑中的实现点（没有任何代码因 CODEX_SANDBOX* 而改变沙箱管理行为）。
- **结论**：作为"自举递归逃逸阀"的解释**未被实现证据支持**。真实机制是"受限运行环境的隐式契约"。
- **处置**：**修改 claim**（环境隐式契约），递归模型降为 HYPOTHESIS。

## CE-07 · 攻击 KO-07 "架构指纹（自然涌现）" → 同源设计

- **攻击目标**：KO-07 L4 "可审计代理系统的架构指纹" 中"指纹"暗示自然涌现/普遍性。
- **反例/边界**：4 处五段式实例均出自**同一架构团队**的刻意设计（Gate 链理念）。不能据此推断"所有代理系统自然涌现五段式"——跨项目重现性**未验证**。若另一项目（如简单脚本工具）未采用该设计，不构成反例，但说明"指纹"是**设计签名**而非**自然规律**。
- **处置**：**epistemic 边界化**（标注同源设计；跨项目验证 pending）。

## CE-08 · 攻击"fail closed 覆盖所有安全 Gate" → 显式白名单例外

- **攻击目标**：KO-05 若被理解为"所有安全 Gate 无条件 fail closed"。
- **反例/边界**：`AskForApproval::Never`（Full Access 白名单）+ `ExecPolicy Allow(bypass_sandbox=true)` 是**显式用户/策略授权放行**——在这些路径上安全 Gate 被策略显式绕过。这不是 fail closed 的反例（fail closed 是默认行为），但**必须作为已知例外写入 claim**，否则读者会误以为 fail closed 绝对成立。
- **处置**：**例外清单化**。

---

## 反例猎杀汇总

| 反例 | 目标 KO | 结果 | 处置 |
|------|--------|------|------|
| CE-01 | KO-01 | bypass 路径存在（策略授权） | 边界化 |
| CE-02 | KO-01 | 四态 CONTRADICTED | 事实修正 |
| CE-03 | KO-01 | Guardian 路由条件性 | 条件化 |
| CE-04 | KO-02 | 延迟确认仅限网络型 | 限定范围 |
| CE-05 | KO-05 | 升级条件不完整 | 补全 |
| CE-06 | KO-03 | 递归模型无实现支持 | 改写为环境隐式契约 |
| CE-07 | KO-07 | 同源设计非自然涌现 | 边界化 |
| CE-08 | KO-05 | 白名单例外 | 例外清单化 |

**结论**：8 个反例/边界中，2 个为事实错误（CE-02, CE-03 属 KO-01 的 L1），1 个推翻核心模型（CE-06 使 KO-03 的递归模型不成立），其余为范围/例外边界化。**没有任何一个 KO 的核心方向被完全推翻**，但 KO-01 的事实层和 KO-03 的抽象层必须修正。

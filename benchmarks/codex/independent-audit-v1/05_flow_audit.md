# 05 · Flow Atlas Audit（流审计）

> 方法：对 wave2 中 6 条 Flow 逐条验证。每条 Flow Edge 检查：source/target/symbol/condition/failure path 是否与代码一致。
> 判定：VERIFIED / PARTIALLY_VERIFIED / UNVERIFIED / INCORRECT。
> 判据："架构上看起来合理" 不能 PASS，必须能锚定真实符号。

---

## F-01 · Control Flow：工具执行控制流

**整体判定：PARTIALLY_VERIFIED（主体正确，approval_policy 分支有错误）**

| Edge | 描述 | 核验 |
|------|------|------|
| User Turn → Session.submit() → StepSettings.resolve() | 入口 | ✅ 与代码一致 |
| Approval Gate 三态（Skip/Forbidden/NeedsApproval） | sandboxing.rs:152-171 | ✅ VERIFIED |
| Skip → 放行（或 strict_auto_review 时 Guardian） | orchestrator.rs:175-195 | ✅ VERIFIED |
| Forbidden → 拒绝 | orchestrator.rs:195-207 | ✅ VERIFIED |
| NeedsApproval → request_approval | orchestrator.rs:193 | ✅ VERIFIED |
| **approval_policy==Never → 自动批准 / OnRequest → Guardian / AlwaysAsk → 人工** | — | ❌ **INCORRECT**：①approval_policy 实为四态（漏 Granular，Granular 关闭子开关时直接 Forbidden）；②"AlwaysAsk" 不存在（真实变体 UnlessTrusted）；③OnRequest→Guardian 缺 approvals_reviewer==AutoReview 条件（review.rs:209-217）；④OnRequest 的 needs_approval 还依赖文件系统是否 Restricted（sandboxing.rs:194-207） |
| Sandbox Selection（unsandboxed_allowed→Bypass / select_initial） | orchestrator.rs:233-282 | ✅ VERIFIED |
| First Attempt（begin_network_approval→tool.run→Deferred/fail） | orchestrator.rs:56-123 | ✅ VERIFIED |
| Escalation Retry 条件（unsandboxed_allowed/approval_policy 允许/非 strict_auto_review） | orchestrator.rs:317-512 | ⚠️ PARTIALLY_VERIFIED：条件清单不完整，缺 network_approval_context.is_none()(342-351)、should_bypass_approval(409-411)、owner_network_policy(233-234)、wants_no_sandbox_approval(362-386) |
| Escalation 时 strict_auto_review 需重新审批 | orchestrator.rs:407-408 | ✅ VERIFIED（wave2 此处比 KO-01 更准确） |
| 符号锚定（ToolOrchestrator::run:125 / select_initial:274 / begin_network_approval:66） | — | ✅ 全部精确 |

**结论**：F-01 的执行序列正确，但 approval_policy 分支（145-147 行）存在与 KO-01 同源的错误（三态 + AlwaysAsk + 无条件 Guardian）。**必须修正为四态 + AutoReview 条件**。

---

## F-02 · State Flow：Session 生命周期状态流

**整体判定：VERIFIED**

| Edge | 核验 |
|------|------|
| Session::new → 并行 setup（tokio::join!） | ✅ session.rs 有并行初始化（auth_and_mcp 等） |
| SessionConfigured Event 发出 | ✅ session.rs:1531 |
| record_initial_history / Active 等待 | ✅ |
| submit(UserTurn) → active_turn=Some(ActiveTurn) → Running → 完成 → None | ✅ session.rs:69 active_turn |
| interrupt（用户输入中断） | ✅ 输入队列机制 |
| Fork（forked_from_thread_id + ForkPersistence Copied/Referenced） | ✅ session.rs:75-76 |
| Resume（InitialHistory::Resumed） | ✅ session.rs:900 |
| Ephemeral（不持久化） | ✅ session.rs:177 |

**结论**：状态容器与转换与代码一致。VERIFIED。

---

## F-03 · Data Flow：权限配置数据流

**整体判定：VERIFIED**

| Edge | 核验 |
|------|------|
| Config.toml → ConfigLoader → PermissionProfileState（三字段） | ✅ session.rs:96-99 |
| materialize_project_roots_with_workspace_roots → FileSystem/Network/SandboxPolicy | ✅ session.rs:172-175, 212-235 |
| effective_permission_profile（环境级覆盖） | ✅ session.rs:177-190 |
| ThreadConfigSnapshot 对外暴露 | ✅ session.rs:237-260 |
| SessionSettingsCommit（持久化 + compaction checkpoint） | ✅ session.rs:49-50 |
| 守恒点：三字段经方法同步禁止独立修改 | ✅ session.rs:96-99 |

**结论**：VERIFIED。

---

## F-04 · Evidence Flow：Guardian 审批证据流

**整体判定：PARTIALLY_VERIFIED（熔断器窗口表述不完整）**

| Edge | 核验 |
|------|------|
| ApprovalContext（review_context+call_id+tool_name+reason+retry_reason+network_approval_context） | ✅ guardian/approval_request.rs |
| GuardianApprovalRequest JSON 序列化 | ✅ |
| compact transcript 限制（20K/10K/5K/1K/40） | ✅ guardian/mod.rs:71-77 |
| Guardian review session（独立，克隆父配置）+ BUNDLED_GUARDIAN_POLICY + 90s | ✅ guardian/mod.rs:45-46, 62 |
| GuardianAssessment（risk_level+user_authorization+outcome+rationale） | ✅ guardian/mod.rs:162-167 |
| allow → 批准 / deny → 熔断器 | ✅ |
| **熔断器："连续 3 次或最近 10 次拒绝 → InterruptTurn"** | ⚠️ **PARTIALLY_VERIFIED**：代码是"连续 3 次 **或 最近 50 次窗口内 10 次**"（guardian/mod.rs:64-68, 196-232）。wave2 漏掉"50 次窗口"限定，仅写"最近 10 次"会造成误导。KO-02 L1 表述（"最近 50 次中 10 次"）反而正确——wave2 此处有误。另：CyberModel 策略（连续 1 次）未在 F-04 标注。 |

**结论**：证据链主体 VERIFIED；熔断器窗口参数需修正。PARTIALLY_VERIFIED。

---

## F-05 · Authority Flow：工具执行权限流

**整体判定：VERIFIED（且比 KO-01 更准确）**

| Edge | 核验 |
|------|------|
| AI Intent → Tool Registry → Tool Router | ✅ |
| Approval Gate → Sandbox Boundary → Network Boundary → Action → Audit | ✅ 五层权限归属成立 |
| Sandbox 类型（bwrap+landlock / Seatbelt / Windows sandbox-service / None） | ✅ |
| **bypass 路径："unsandboxed_allowed + Never approval → 无沙箱直接执行；strict_auto_review 时即使 Skip 也需 Guardian 审查"** | ✅ **VERIFIED 且准确**——wave2 的 F-05 明确写了 bypass 与 strict_auto_review 例外，比 KO-01 的 L1 更完整 |

**结论**：VERIFIED。注意：**F-05 的正确性反而暴露 KO-01 的降质**——问题发生在 wave2→wave3 的合成阶段（synthesizer 把准确的 Flow 简化/引入了错误），而非 discovery 阶段。

---

## F-06 · Memory Flow：会话记忆与历史流

**整体判定：VERIFIED**

| Edge | 核验 |
|------|------|
| User Turn + Tool Results → SessionState（内存） | ✅ |
| Rollout（RolloutItem 枚举：SessionMeta/ResponseItem/EventMsg/TurnContext/WorldState/Compacted/InterAgentCommunication/TokenUsageRecord） | ✅ session.rs:1356 |
| ThreadStore（LocalThreadStore/CCA）+ state_db | ✅ session.rs:667 |
| Compaction（AutoCompactWindow → Compacted checkpoint） | ✅ session.rs:796 |
| RealtimeConversationManager | ✅ session.rs:67 |
| Memories（config.memories.generate_memories 可选） | ✅ |
| No history rewrite（增量构建） | ✅ AGENTS.md:95 |
| ForkPersistence（Copied/Referenced） | ✅ session.rs:75 |

**结论**：VERIFIED。此流与 AGENTS.md 上下文 6 条硬限制（IF-05）相互印证——但上下文治理本身未升维成 KO（见 03_coverage_gaps）。

---

## Flow Atlas 审计汇总

| Flow | 判定 | 问题 |
|------|------|------|
| F-01 Control | **PARTIALLY_VERIFIED** | approval_policy 三态错误 + AlwaysAsk 不存在 + 缺 AutoReview 条件；Escalation 条件不完整 |
| F-02 State | VERIFIED | — |
| F-03 Data | VERIFIED | — |
| F-04 Evidence | **PARTIALLY_VERIFIED** | 熔断器"最近 10 次"漏"50 次窗口"；未标 CyberModel |
| F-05 Authority | VERIFIED | （比 KO-01 更准确） |
| F-06 Memory | VERIFIED | — |

**关键发现**：Flow Atlas 6 条中 4 条完全正确，F-01 有 approval_policy 分支错误（与 KO-01 同源），F-04 有参数表述误差。且 F-05 的正确性证明问题出在 **wave2→wave3 合成阶段**（Synthesizer 把准确的 Flow 简化并引入错误），而非 Discovery 阶段。这为 Skill Benchmark 提供了精确的故障定位：**Synthesis 阶段缺少与 Flow/Evidence 的逐条交叉校验**。

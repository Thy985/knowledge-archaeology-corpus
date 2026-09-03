# 01 · Independent Findings（独立盲重建）

> 本文件回答：**如果完全不知道已有 Knowledge Archaeology 结果，这个项目最值得沉淀的知识是什么？**
> 来源一（主）：审计者在独立阅读代码过程中的第一手观察（orchestrator.rs 全文 / session.rs / guardian/mod.rs / review.rs / sandboxing.rs / control.rs / exec_policy.rs / AGENTS.md）。
> 来源二（补充）：独立盲审 SubAgent（o_0000PxPWaTb，完全隔离于已有考古结论）的独立重建——其产出到达后并入本文件。
> 本文件的内容与已有 7 个 KO **不存在任何预设关系**，是独立的观察清单。

---

## 一、关键工程事实（独立观察，带文件:行号）

| # | 事实 | 证据 |
|---|------|------|
| IF-A1 | 工具执行走 Approval→Sandbox→Attempt→Escalation 序列，Escalation 是多条件 Gate | orchestrator.rs:125-527 |
| IF-A2 | approval_policy（AskForApproval）是**四态**：Never/OnRequest/Granular/UnlessTrusted，非三态 | sandboxing.rs:189-207 |
| IF-A3 | Granular 态带 sandbox_approval + rules 两个子开关，关闭时直接 Forbidden | sandboxing.rs:194-219, exec_policy.rs:49-53 |
| IF-A4 | Guardian 审批路由条件：OnRequest\|Granular + AutoReview；Never/UnlessTrusted 不经 Guardian | review.rs:209-217 |
| IF-A5 | Guardian 是独立 review session，transcript 有 5 层 token 限制，90s 超时 fail closed，有熔断器 | guardian/mod.rs:62-77, 170-232 |
| IF-A6 | 网络审批采用 Active→Deferred 延迟确认（可取消 + proxy） | orchestrator.rs:66-123 |
| IF-A7 | **exec_policy 完整子系统**：命令级审批策略（rules 文件）、ExecutableIdentity、ModelPolicy、危险命令黑名单 | exec_policy.rs 全文 |
| IF-A8 | **审批→策略固化闭环**：批准命令后可提议把 allow 规则追加进策略（blocking_append_allow_prefix_rule） | exec_policy.rs:20, sandboxing.rs:158-167 |
| IF-A9 | **模型可见上下文 6 条硬限制**：无重写/缓存友好/有界/10K cap/1K P0/ContextualUserFragment | AGENTS.md:93-100 |
| IF-A10 | PermissionProfile 是沙箱策略 SSOT，三策略派生 + 三字段同步 + 环境覆盖 | session.rs:96-99, 164-235 |
| IF-A11 | 持久化与运行时状态分离（thread_settings_persistence 独立于 state） | session.rs:48-49 |
| IF-A12 | AgentControl 用 Weak<ThreadManagerState> 避免引用循环与隐式持久化 | control.rs:122-123 |
| IF-A13 | 多 Agent 有并发限制（AgentExecutionLimiter）+ rollout 预算 | control.rs:129-131, 169-171 |
| IF-A14 | CODEX_SANDBOX* 环境变量由运行时设置、内部只读、用于测试跳过 | AGENTS.md:8-10 |
| IF-A15 | 多层级膨胀治理：crate 级 / 模块级(500-800 LoC) / 变更级(800-500 行) / API surface | AGENTS.md:49-53, 72-83, 87-89, 125-131 |
| IF-A16 | 沙箱外重试条件含 network_approval_context / should_bypass_approval / owner_network_policy / wants_no_sandbox_approval | orchestrator.rs:233-234, 342-351, 362-386, 409-411 |

---

## 二、我认为这个项目在解决什么问题（独立判断）

Codex 在解决一个双层问题：

**表层（用户问题）**：让 AI 能在终端里安全地读写代码、跑命令、访问网络，像一个懂工程的协作者一样工作。

**深层（工程问题）**：**当"产生意图的智能"与"执行动作的通道"在同一系统里时，如何保证系统在智能出错时仍然可控、可审计、可恢复？** 这不是一个功能问题，而是一个"代理系统的安全治理架构"问题。Codex 的绝大多数复杂机制（Approval Gate、Guardian、沙箱、exec_policy、网络代理、延迟确认、熔断器）都在回答这一个问题。

这个判断与产品层无关——即使换一个 AI 工具，只要它有自主执行能力，就必须回答这个问题。

---

## 三、独立识别的工程问题与根因（盲重建）

| 问题 | 独立观察到的机制 | 根因判断 |
|------|----------------|---------|
| AI 审批（Guardian）本身不可靠 | 证据生产链（transcript 限流/结构化/超时/熔断） | AI 判断是概率性的，不能作为安全事实 |
| 逐次人工审批无法规模化 | exec_policy 策略固化（审批→allow 规则落盘） | 人工审批是稀缺资源，需要沉淀为可复用策略 |
| 多维度安全策略易冲突 | PermissionProfile SSOT + 纯函数派生 | 多真相对齐成本随维度指数增长 |
| Agent 长期会话上下文失控 | 上下文 6 条硬限制 + compaction | 模型上下文是有限资源，无界注入必然劣化 |
| 核心 crate 无限膨胀 | 多层级膨胀治理（crate/模块/变更/API） | 路径依赖 + 正反馈使核心模块只进不出 |
| 沙箱内运行又管理沙箱 | 环境变量隐式契约（运行时设置、内部只读） | 受限运行环境需显式告知代码约束 |

---

## 四、独立观察到的重复结构（盲重建，不宣布原则）

1. **Gate 链结构**：Approval Gate、Sandbox Gate、Network Proxy Gate 反复出现——同一"验证→授权→执行"节奏在多个安全关键点重复。
2. **fail closed 默认**：Guardian 超时/失败/格式错误、附件网络策略、升级条件不满足——全部"不确定时拒绝"。
3. **SSOT + 派生**：PermissionProfile→三策略、SessionConfiguration→ThreadConfigSnapshot、AGENTS.md→工程规则——同一"单一真相源"模式。
4. **熔断器/缓存/延迟确认**：连续拒绝中断、审批缓存、Active→Deferred——同一"状态机控制重复动作"模式。
5. **防御性命名**：`original_config_do_not_use`、TODO 标记、忽略的 `_output` 参数——技术债的显式标记。

---

## 五、六类流（独立锚定真实符号）

### Control Flow（谁决定下一步）
```
User Turn → Session.submit() → StepSettings.resolve() → ToolOrchestrator.run()
  → Approval Gate（Skip/Forbidden/NeedsApproval）→ Sandbox Selection → run_attempt
  → Escalation Retry（多条件）→ Event 输出
```

### State Flow（状态如何变化）
```
Session::new → 并行 setup → SessionConfigured → Active
  → submit → Running（active_turn=Some）→ 完成 → Active
  → Fork（ForkPersistence） / Resume（InitialHistory::Resumed） / Ephemeral
```

### Data Flow（数据从哪来到哪去）
```
Config.toml → PermissionProfileState → materialize → FileSystem/Network/SandboxPolicy
  → effective（环境覆盖）→ ThreadConfigSnapshot → SessionSettingsCommit
```

### Evidence Flow（如何证明成立）
```
Tool 调用 → ApprovalContext → GuardianApprovalRequest → compact transcript（限流）
  → Guardian review（90s fail closed）→ GuardianAssessment → allow/deny + 熔断器
```

### Authority Flow（谁拥有执行权）
```
AI Intent → Tool Registry → Router → Approval Gate → Sandbox Boundary
  → Network Boundary → Action → Audit
  （bypass 路径：ExecPolicy Allow / BypassSandboxFirstAttempt 可构造无沙箱直通）
```

### Memory Flow（记忆如何流转）
```
User Turn + Tool Results → SessionState → Rollout → ThreadStore
  → Compaction（AutoCompactWindow）→ RealtimeConversationManager → Memories（可选）
  （约束：No history rewrite / 有界注入 / ContextualUserFragment）
```

---

## 六、候选原则（独立判断，标注证据强度）

> 以下为审计者独立观察候选，不参考已有 KO 的结论。证据强度：S0-S8。

| 候选 | 表述 | 证据强度 | 备注 |
|------|------|---------|------|
| P-C1 | 自主执行系统的执行权必须独立于产生意图的智能，经多重 Gate 控制 | S5（orchestrator.rs 实现 + 测试） | 与已有 KO-01 方向一致 |
| P-C2 | AI 对 AI 的审批必须基于证据生产链而非声明 | S4（guardian 实现） | 与已有 KO-02 方向一致 |
| P-C3 | 逐次审批应通过策略固化演变为可复用策略（避免重复人工确认） | S4（exec_policy 实现） | **已有 KO 未覆盖** |
| P-C4 | Agent 长期会话的模型上下文必须有硬性治理（有界/缓存友好/结构类型化） | S4（AGENTS.md 规则） | **已有 KO 未覆盖** |
| P-C5 | 多维度安全策略必须 SSOT + 纯函数派生 | S4（session.rs） | 与已有 KO-04 方向一致 |
| P-C6 | 不确定时安全关键 Gate 默认拒绝（fail closed） | S5 | 与已有 KO-05 方向一致 |

---

## 七、审计者最强的独立结论

> **Codex 的真正贡献不是"一个 AI 写代码工具"，而是"自主执行系统的安全治理架构"：它系统性地回答了'当智能可能出错时，如何让执行仍然可控、可审计、可演进'。** 其中最有迁移价值的两条是：①多重 Gate + fail closed + 证据生产链（治理智能的不确定性）；②审批→策略固化 + 上下文硬治理（让系统随使用演进而不失控）。前者被已有 KO 较好覆盖，后者（exec_policy / 上下文治理）**被已有考古遗漏**。

---

> **状态说明**：本文件为审计者独立代码核验的观察结果。独立盲审 SubAgent（完全隔离于已有考古结论）的独立重建正在执行，其产出到达后将并入本文件作为第二独立来源并交叉验证本节结论。

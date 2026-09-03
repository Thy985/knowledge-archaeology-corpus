# Wave 2 · Evidence Graph + Mining Graphs

> Evidence Fusion（evidence-assembler）+ Problem/Decision/Flow/Pattern Miner 联合产出。
> 输入：Wave 1 的 28 条 Evidence Pack | 时间：2026-09-03

---

## 一、Evidence Graph（证据融合）

### Supported Facts（≥2 独立源支撑）

| ID | Fact | 支撑证据 | 强度 |
|----|------|---------|------|
| SF-01 | 工具执行遵循固定序列：审批→沙箱选择→尝试→失败后升级重试，重试无需重新审批（approval caching） | EV-C003, EV-A001, EV-A005 | S4 |
| SF-02 | Guardian 是独立 AI 审查 session，克隆父配置继承网络策略，90 秒超时，fail closed | EV-A002, EV-F004, EV-A003 | S4 |
| SF-03 | PermissionProfile 是沙箱策略的单一真相源，文件系统/网络/沙箱策略均从中派生 | EV-C005, EV-A004 | S4 |
| SF-04 | codex-core crate 有意识反膨胀，472 个源文件已是最大 crate，AGENTS.md 明确抵制新增 | EV-D002, EV-C001 | S4 |
| SF-05 | 模型可见上下文有 6 条硬限制：增量构建/缓存友好/有界/10K token 上限/>1K 标记 P0/必须实现 ContextualUserFragment | EV-D003, EV-C001 | S4 |
| SF-06 | 多 Agent V2 对子 Agent 实施并发线程数限制（AtomicUsize 计数 + Drop 自动释放） | EV-C006, EV-C002 | S3 |
| SF-07 | 沙箱自举问题通过环境变量逃逸阀解决（CODEX_SANDBOX* 由运行时设置，代码禁止修改） | EV-F001, EV-D001 | S4 |
| SF-08 | 网络访问有独立 Gate：托管代理 + allowlist-miss 回调 + 延迟确认（DeferredNetworkApproval） | EV-C007, EV-A005, EV-A006 | S4 |
| SF-09 | 测试体系规范：独立测试文件+#[path] / insta snapshot / 集成测试优先 / pretty_assertions 深比较 | EV-T001~T005 | S4 |
| SF-10 | 技术债管理：TODO 标记已知问题 / 字段名加 _do_not_use 防御扩散 / 占位实现预留扩展点 | EV-F002, EV-F003 | S3 |

### Unverified Observations（单源，不进入下游推理）

- UO-01：Session 初始化用 tokio::join! 并行三个 setup 任务（仅 EV-C002）
- UO-02：项目文档倾向外链到开发者文档站（仅 EV-D006）
- UO-03：Rust trait 优先原生 RPITIT 而非 async_trait（仅 EV-D005）
- UO-04：变更大小限制 800 行/复杂逻辑 500 行（仅 EV-D004）

### 冲突证据（保留，供 Reconciler）

- 无直接冲突。doc-analyst 的 design_intent 与 code-analyst 的 implementation 在所有点上一致（AGENTS.md 规定的规则在代码中均有对应实现）。

---

## 二、Problem Graph（problem-miner）

### P-01 · Agent 执行权限的可控性问题
- **Problem**：AI Agent 拥有执行 shell 命令的能力，如何在自主性与安全性之间取得平衡？
- **Constraint**：Agent 判断不可靠（Guardian fail closed 设计暗示），一个错误判断 = 一次未授权变更
- **Pain**：完全禁止 → 失去自主性；完全放行 → 安全风险
- **RootCause**：Intelligence（判断力）与 Authority（执行权）在 Agent 系统中天然耦合，但可靠性不对称
- **支撑证据**：SF-01, SF-02, SF-03, EV-A001, EV-A002

### P-02 · 沙箱自举的递归问题
- **Problem**：Agent 自身运行在沙箱中，但 Agent 的功能包括管理沙箱（spawn 子进程、配置沙箱策略），形成递归
- **Constraint**：沙箱内无法运行需要自己 spawn Seatbelt 的集成测试
- **Pain**：测试覆盖盲区 + 环境变量作为隐式契约
- **RootCause**：自举系统的内在矛盾——被管理的系统同时是管理者的宿主
- **支撑证据**：SF-07, EV-F001, EV-D001

### P-03 · 大型 Rust 工作空间的 crate 膨胀问题
- **Problem**：core crate 随功能增长不断膨胀（472 文件），成为所有功能的"方便倾倒场"
- **Constraint**：拆分 crate 需要重构依赖，成本高
- **Pain**：编译时间增长 / 职责边界模糊 / 变更影响面大
- **RootCause**："添加到 core 比重构出新 crate 更容易"的路径依赖
- **支撑证据**：SF-04, EV-D002, EV-C001

### P-04 · 模型上下文窗口的资源竞争问题
- **Problem**：Agent 会话历史、工具结果、系统提示、用户指令竞争有限的上下文窗口
- **Constraint**：单条注入 ≤10K tokens，>1K 需 P0 审查，一切注入必须有界
- **Pain**：上下文溢出导致遗忘 / 频繁变更导致 cache miss 增加成本
- **RootCause**：LLM 上下文窗口是稀缺资源，而 Agent 系统的状态天然增长
- **支撑证据**：SF-05, EV-D003

### P-05 · 多 Agent 并发的资源耗尽问题
- **Problem**：V2 多 Agent 允许子 Agent 并行，但无限制的子 Agent spawn 会耗尽系统资源
- **Constraint**：max_threads 配置限制，超限返回 AgentLimitReached
- **Pain**：用户等待 / 系统不稳定
- **RootCause**：Agent 递归 spawn 的指数增长倾向
- **支撑证据**：SF-06, EV-C006

### P-06 · 网络访问的细粒度管控问题
- **Problem**：Agent 需要网络访问（API 调用、依赖下载），但网络是最大的安全边界
- **Constraint**：托管代理 + allowlist + 延迟确认 + 附件级策略不可绕过
- **Pain**：合法网络请求被拦截 / 配置复杂
- **RootCause**：网络是 Agent 与外部世界交互的唯一通道，同时是最大攻击面
- **支撑证据**：SF-08, EV-C007, EV-A005, EV-A006

---

## 三、Decision Graph（decision-miner）

### D-01 · 审批-沙箱-执行的三级 Gate 架构
- **Decision**：选择"审批策略 → 沙箱策略 → 执行"的三级 Gate，而非单一权限检查
- **Alternative**：单一权限检查（简单但粒度粗）/ 完全人工审批（安全但慢）
- **Trade-off**：安全性 vs 自主性；三级 Gate 允许 Never（全自动）/ OnRequest（Guardian 自动+人工兜底）/ AlwaysAsk（全人工）
- **Rejected**：单一权限检查（无法区分工具风险等级）
- **支撑证据**：SF-01, EV-A001, EV-C003

### D-02 · Guardian 用独立 AI session 做自动审批，fail closed
- **Decision**：用独立的 Guardian review session（AI 审查 AI）替代简单规则匹配做 on-request 审批
- **Alternative**：规则匹配（可解释但僵硬）/ 完全人工（安全但慢）
- **Trade-off**：AI 审查的灵活性 vs 可解释性；fail closed 确保安全优先
- **Rejected**：规则匹配（无法处理复杂上下文）
- **支撑证据**：SF-02, EV-A002, EV-F004

### D-03 · PermissionProfile 作为沙箱策略单一真相源
- **Decision**：所有沙箱策略（文件系统/网络/执行）从 PermissionProfile 统一派生，而非各模块独立配置
- **Alternative**：各模块独立配置（灵活但不一致）
- **Trade-off**：一致性 vs 灵活性；单一真相源确保策略可审计、可推理
- **Rejected**：分散配置（策略冲突难以发现）
- **支撑证据**：SF-03, EV-C005, EV-A004

### D-04 · core crate 反膨胀的治理决策
- **Decision**：在 AGENTS.md 中明确规定"resist adding code to codex-core"，code review 时 push back
- **Alternative**：放任增长（短期快）/ 大规模重构（成本高）
- **Trade-off**：短期开发速度 vs 长期可维护性
- **Rejected**：大规模重构（在活跃开发中成本太高）
- **支撑证据**：SF-04, EV-D002

### D-05 · 模型上下文的硬限制治理
- **Decision**：用 6 条硬规则治理模型上下文注入，而非依赖开发者自觉
- **Alternative**：开发者自觉（不可靠）/ 运行时动态裁剪（复杂）
- **Trade-off**：开发约束 vs 系统稳定性；硬限制确保可预测性
- **支撑证据**：SF-05, EV-D003

### D-06 · 网络访问的延迟确认（Deferred Approval）模式
- **Decision**：网络审批采用"先放行后确认"（begin → Active → Deferred → finish），而非"先确认后放行"
- **Alternative**：先确认后放行（安全但延迟高）
- **Trade-off**：性能 vs 安全；延迟确认在工具执行期间保持网络代理，执行后最终确认
- **支撑证据**：SF-08, EV-A005

### D-07 · 测试体系的工程化决策
- **Decision**：独立测试文件+#[path] / insta snapshot / 集成测试优先 / pretty_assertions
- **Alternative**：内联测试（简单但混乱）/ 仅单元测试（快但覆盖浅）
- **Trade-off**：测试可维护性 vs 编写成本；集成测试优先确保 Agent 行为真实
- **支撑证据**：SF-09, EV-T001~T005

---

## 四、Flow Atlas（flow-miner · 六类流）

### F-01 · Control Flow：工具执行控制流
```
User Turn → Session.submit()
  → StepSettings.resolve()（模型/审批策略/沙箱级别）
  → ToolOrchestrator.run()
    → 1. Approval Gate
       ├─ ExecApprovalRequirement::Skip → 直接放行（或 strict_auto_review 时 Guardian 审查）
       ├─ ExecApprovalRequirement::Forbidden → 拒绝
       └─ ExecApprovalRequirement::NeedsApproval → request_approval()
         ├─ approval_policy == Never → 自动批准
         ├─ approval_policy == OnRequest → Guardian review（90s timeout, fail closed）
         └─ approval_policy == AlwaysAsk → 人工审批
    → 2. Sandbox Selection
       ├─ unsandboxed_allowed? → SandboxOverride::BypassSandboxFirstAttempt
       └─ SandboxManager.select_initial() → SandboxType（Linux bwrap / macOS Seatbelt / Windows / None）
    → 3. First Attempt（run_attempt）
       ├─ begin_network_approval() → ActiveNetworkApproval（含 proxy + cancellation_token）
       ├─ tool.run() → 执行
       └─ 成功 → DeferredNetworkApproval（延迟确认）/ 失败 → 立即 finalize
    → 4. Escalation Retry（仅当 SandboxErr::Denied + escalate_on_failure）
       ├─ 条件检查：unsandboxed_allowed / approval_policy 允许 / 非 strict_auto_review
       ├─ 可能需要重新审批（strict_auto_review 时）
       └─ SandboxManager.select_initial() with reduced sandbox → Second Attempt
  → Event 输出
```
- **关键 Gate**：Approval Gate（三态）/ Sandbox Selection（SandboxManager）/ Escalation Condition（多条件）
- **符号锚定**：ToolOrchestrator::run (orchestrator.rs:125), request_approval (orchestrator.rs:193), SandboxManager.select_initial (orchestrator.rs:274), begin_network_approval (orchestrator.rs:66)

### F-02 · State Flow：Session 生命周期状态流
```
Session::new()（初始化）
  → 并行 setup：thread_persistence / state_db / auth_and_mcp（tokio::join!）
  → SessionConfigured Event 发出
  → record_initial_history()
  → Active（等待 User Turn）
    → submit(UserTurn) → active_turn = Some(ActiveTurn)
      → Step 执行循环（模型推理 → 工具调用 → 结果）
      → Turn 完成 → active_turn = None
    → 可被 interrupt（用户输入中断）
  → Fork（子 Agent）→ forked_from_thread_id + ForkPersistence（Copied/Referenced）
  → Resume（从 thread_store 恢复）→ InitialHistory::Resumed
  → Ephemeral（不持久化）→ 无 thread_persistence
```
- **状态容器**：Session.state (Mutex<SessionState>) / active_turn (Mutex<Option<ActiveTurn>>)
- **关键转换**：new → Active / submit → Running / complete → Active / fork → Forked / resume → Resumed
- **符号锚定**：Session::new (session.rs:639), SessionConfiguredEvent (session.rs:1531), ForkPersistence (session.rs:75), InitialHistory::Resumed (session.rs:900)

### F-03 · Data Flow：权限配置数据流
```
Config.toml → ConfigLoader
  → PermissionProfileState（constrained profile + active profile id + workspace roots）
    → materialize_project_roots_with_workspace_roots()
      → FileSystemSandboxPolicy（entries: path + access read/write）
      → NetworkSandboxPolicy（allowlist + proxy）
      → SandboxPolicy（legacy compatibility）
    → effective_permission_profile()（环境级覆盖）
  → ThreadConfigSnapshot（对外暴露）
  → SessionSettingsCommit（持久化 + compaction checkpoint）
```
- **数据形态**：TOML → PermissionProfile → FileSystemSandboxPolicy + NetworkSandboxPolicy → SandboxAttempt
- **守恒点**：PermissionProfileState 三字段必须通过方法同步，禁止独立修改
- **符号锚定**：PermissionProfileState (session.rs:99), materialize_project_roots (session.rs:172), file_system_sandbox_policy (session.rs:223), ThreadConfigSnapshot (session.rs:237)

### F-04 · Evidence Flow：Guardian 审批证据流
```
Tool 调用请求 → ApprovalContext（review_context + call_id + tool_name + approval_reason + retry_reason + network_approval_context）
  → GuardianApprovalRequest（JSON 序列化）
    → 重建 compact transcript（user intent + recent assistant + tool context）
      → 限制：message transcript ≤20K tokens / tool transcript ≤10K / 单条 message ≤5K / 单条 tool ≤1K / recent ≤40 条
    → Guardian review session（独立 session，克隆父配置）
      → BUNDLED_GUARDIAN_POLICY（内置策略）
      → 90 秒超时
    → GuardianAssessment（risk_level + user_authorization + outcome + rationale）
      → allow → 批准执行
      → deny → 拒绝 + 记录熔断器
        → GuardianRejectionCircuitBreaker（连续 3 次或最近 10 次拒绝 → InterruptTurn）
```
- **证据生产链**：ApprovalContext → compact transcript → GuardianAssessment → outcome
- **关键 Gate**：transcript token 限制 / 90s timeout / fail closed / 熔断器
- **符号锚定**：GuardianReviewContext (guardian/mod.rs:87), GuardianApprovalRequest (guardian/mod.rs:40), GuardianAssessment (guardian/mod.rs:162), GuardianRejectionCircuitBreaker (guardian/mod.rs:170), GUARDIAN_REVIEW_TIMEOUT=90s (guardian/mod.rs:62)

### F-05 · Authority Flow：工具执行权限流
```
AI Intent（模型输出 tool_call）
  → Tool Registry（注册可用工具）
  → Tool Router（路由到具体 handler）
  → Approval Gate（见 F-01）
    → Skip/Forbidden/NeedsApproval 三态
  → Sandbox Boundary（SandboxManager 选择沙箱类型）
    → Linux: bwrap + landlock / macOS: Seatbelt / Windows: sandbox-service / None: 无沙箱
  → Network Boundary（Managed Network Proxy + allowlist）
    → begin_network_approval → ActiveNetworkApproval（proxy + cancellation_token）
  → Action（tool.run() 实际执行）
  → Audit（Event 输出 + telemetry + deferred network approval finalize）
```
- **权限归属**：模型（Intent）→ Orchestrator（Approval）→ SandboxManager（Sandbox）→ NetworkProxy（Network）→ Tool（Action）→ Event（Audit）
- **bypass 路径**：unsandboxed_allowed + Never approval → 无沙箱直接执行；strict_auto_review 时即使 Skip 也需 Guardian 审查
- **符号锚定**：ToolOrchestrator (orchestrator.rs:40), SandboxManager (orchestrator.rs:41), ActiveNetworkApproval (orchestrator.rs:15), begin_network_approval (orchestrator.rs:66)

### F-06 · Memory Flow：会话记忆与历史流
```
User Turn + Tool Results → SessionState（内存状态）
  → Rollout（持久化事件序列）
    → RolloutItem：SessionMeta / ResponseItem / EventMsg / TurnContext / WorldState / Compacted / InterAgentCommunication / TokenUsageRecord
  → ThreadStore（持久化存储）
    → LocalThreadStore（本地）/ CCA（云）
    → state_db（SQLite? 状态数据库）
  → Compaction（上下文压缩）
    → AutoCompactWindow（自动压缩窗口）
    → Compacted checkpoint（含 mcp_resource_origins 恢复点）
  → RealtimeConversationManager（实时对话）
  → Memories（可选记忆生成，config.memories.generate_memories）
```
- **记忆资产**：Rollout（事件源）/ ThreadStore（持久化）/ state_db（状态）/ Compacted checkpoints（压缩点）
- **关键约束**：No history rewrite（增量构建）/ ForkPersistence（Copied 全量复制 / Referenced 引用+保留）
- **符号锚定**：RolloutItem (session.rs:1356), ThreadStore (session.rs:667), AutoCompactWindow (session.rs:796), ForkPersistence (session.rs:75), RealtimeConversationManager (session.rs:67)

---

## 五、Pattern Graph（pattern-miner · 重复结构）

### PAT-01 · 产生→验证→授权→执行→记录 的五段式架构指纹
- **观察到的重复结构**：
  1. 工具执行：模型产生 tool_call → Approval 验证 → Sandbox 授权 → tool.run 执行 → Event 记录（F-01, F-05）
  2. Guardian 审批：ApprovalContext 产生 → transcript 验证 → Guardian 授权 → 执行放行 → 熔断器记录（F-04）
  3. 网络访问：请求产生 → begin_network_approval 验证 → proxy 授权 → 执行 → DeferredNetworkApproval 记录（EV-A005）
  4. Session 初始化：配置产生 → 并行 setup 验证 → 权限配置授权 → Session 创建 → SessionConfigured Event 记录（F-02）
- **出现次数**：4 处独立出现
- **证据**：SF-01, SF-02, SF-08, F-01, F-04, F-05
- **注意**：这是 Observed repeated structure，不自行宣布原则

### PAT-02 · 熔断器（Circuit Breaker）模式在安全关键路径的重复
- **观察到的重复结构**：
  1. Guardian 拒绝熔断器：连续 3 次/最近 10 次拒绝 → InterruptTurn（EV-A003）
  2. 工具审批缓存：首次审批后重试无需重新审批（approval caching，EV-C003）
  3. 网络延迟确认：Active → Deferred → finish 的状态机（EV-A005）
- **出现次数**：3 处
- **证据**：SF-02, SF-08

### PAT-03 · 单一真相源（Single Source of Truth）模式
- **观察到的重复结构**：
  1. PermissionProfile → 所有沙箱策略派生（SF-03）
  2. SessionConfiguration → ThreadConfigSnapshot + ThreadSettingsSnapshot 派生（session.rs:237-328）
  3. AGENTS.md → 所有工程规则的权威来源（EV-D001~D005）
- **出现次数**：3 处
- **证据**：SF-03, SF-04

### PAT-04 · Fail Closed 安全默认模式
- **观察到的重复结构**：
  1. Guardian 超时/失败/格式错误 → 默认拒绝（EV-F004）
  2. 沙箱拒绝 + 不满足升级条件 → 保持拒绝（EV-C004）
  3. 附件网络策略 + 升级权限 → 直接拒绝（EV-A006）
  4. 多 Agent 超限 → AgentLimitReached 拒绝（EV-C006）
- **出现次数**：4 处
- **证据**：SF-02, SF-06, SF-08

### PAT-05 · 防御性命名标记技术债
- **观察到的重复结构**：
  1. `original_config_do_not_use` 字段名（session.rs:116）
  2. TODO(anp) / TODO(pakrym) / TODO(jif) 标记（session.rs:102,115,1433）
  3. `_output` 参数被忽略但保留用于未来扩展（orchestrator.rs:542）
- **出现次数**：3 处
- **证据**：SF-10

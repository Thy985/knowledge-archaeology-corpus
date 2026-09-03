# Wave 1 · Evidence Packs（Discovery 阶段）

> 由 code-analyst / doc-analyst / test-analyst / failure-analyst / authority-analyst 五角色并行提取。
> 项目：openai/codex | 时间：2026-09-03

---

## 一、Code Evidence Pack（code-analyst）

### EV-C001 · Session 是 Agent 运行时的核心状态容器
- **stage**: code
- **discovered_by**: code-analyst
- **source**: codex-rs/core/src/session/session.rs:42-78
- **claim**: Session 结构体持有 thread_id / state(Mutex<SessionState>) / active_turn / services(SessionServices) / guardian_review_session / input_queue / conversation(RealtimeConversationManager)，是 Agent 运行时的核心状态容器，一次最多运行一个 task
- **data_form**: 结构体字段 → 运行时状态聚合
- **symbols**: [Session, SessionState, SessionServices, ActiveTurn, GuardianReviewSessionManager, RealtimeConversationManager]
- **strength**: S3
- **detail**: "A session has at most 1 running task at a time, and can be interrupted by user input."

### EV-C002 · Session 初始化采用并行异步 setup 降低启动延迟
- **stage**: code
- **discovered_by**: code-analyst
- **source**: codex-rs/core/src/session/session.rs:840-1002
- **claim**: Session::new() 将 thread_persistence / state_db / auth_and_mcp 三个独立 setup 任务用 tokio::join! 并行执行，注释明确说明"Kick off independent async setup tasks in parallel to reduce startup latency"
- **data_form**: 顺序初始化 → 并行 join
- **symbols**: [tokio::join!, thread_persistence_fut, state_db_fut, auth_and_mcp_fut]
- **strength**: S3

### EV-C003 · ToolOrchestrator 实现审批→沙箱→尝试→升级重试的固定序列
- **stage**: code
- **discovered_by**: code-analyst
- **source**: codex-rs/core/src/tools/orchestrator.rs:1-8, 125-527
- **claim**: ToolOrchestrator::run() 对任何 ToolRuntime 驱动固定序列：1) Approval（Skip/Forbidden/NeedsApproval 三态）→ 2) 选择沙箱（SandboxManager.select_initial）→ 3) 首次尝试 → 4) 若沙箱拒绝且 escalate_on_failure，则升级沙箱策略重试（可能无沙箱），重试无需重新审批（approval caching）
- **data_form**: 工具调用 → 审批 Gate → 沙箱 Gate → 执行 → 失败升级
- **symbols**: [ToolOrchestrator, ExecApprovalRequirement, SandboxManager, SandboxAttempt, SandboxOverride]
- **strength**: S3
- **detail**: "Central place for approvals + sandbox selection + retry semantics. Drives a simple sequence for any ToolRuntime: approval → select sandbox → attempt → retry with an escalated sandbox strategy on denial (no re-approval thanks to caching)."

### EV-C004 · 沙箱拒绝后升级重试有严格的前置条件
- **stage**: code
- **discovered_by**: code-analyst
- **source**: codex-rs/core/src/tools/orchestrator.rs:317-438
- **claim**: 沙箱拒绝后升级重试必须同时满足：tool.escalate_on_failure() == true / unsandboxed_allowed == true（或 network_approval_context 存在）/ approval_policy 允许无沙箱审批（Never 或 OnRequest 且满足条件）/ 非 strict_auto_review 模式。strict_auto_review 下重试无沙箱必须重新 Guardian 审批
- **data_form**: 沙箱拒绝 → 多条件 Gate → 升级重试
- **symbols**: [escalate_on_failure, unsandboxed_execution_allowed, wants_no_sandbox_approval, strict_auto_review]
- **strength**: S3

### EV-C005 · 权限配置文件（PermissionProfile）是沙箱策略的单一真相源
- **stage**: code
- **discovered_by**: code-analyst
- **source**: codex-rs/core/src/session/session.rs:96-99, 168-235
- **claim**: SessionConfiguration 持有 permission_profile_state（Constrained profile + active profile id + profile-defined workspace roots），所有沙箱策略（file_system_sandbox_policy / network_sandbox_policy / sandbox_policy）都从 permission_profile 派生，注释要求"Keep the constrained profile, active profile id, and profile-defined workspace roots in sync by using the methods below instead of mutating the fields independently"
- **data_form**: PermissionProfile → FileSystemSandboxPolicy + NetworkSandboxPolicy + SandboxPolicy
- **symbols**: [PermissionProfileState, PermissionProfile, FileSystemSandboxPolicy, NetworkSandboxPolicy, SandboxPolicy]
- **strength**: S3

### EV-C006 · 多 Agent V2 对子 Agent 实施并发线程数限制
- **stage**: code
- **discovered_by**: code-analyst
- **source**: codex-rs/core/src/agent/control/execution.rs:13-97
- **claim**: AgentExecutionLimiter 用 AtomicUsize 计数活跃子 Agent，max_threads 由 config.effective_agent_max_threads(MultiAgentVersion::V2) 配置（session.rs:813-816），仅对 V2 多 Agent 的 SubAgent 生效（is_execution_limited），超限时返回 AgentLimitReached 错误；AgentExecutionGuard 用 Drop 自动释放计数
- **data_form**: 子 Agent spawn → 信号量计数 → 超限拒绝
- **symbols**: [AgentExecutionLimiter, AgentExecutionGuard, MultiAgentVersion::V2, AgentLimitReached]
- **strength**: S3

### EV-C007 · 托管网络代理（Managed Network Proxy）作为网络访问的独立 Gate
- **stage**: code
- **discovered_by**: code-analyst
- **source**: codex-rs/core/src/session/session.rs:1262-1320, orchestrator.rs:244-263
- **claim**: 当 config.permissions.network 配置了 NetworkProxySpec 时，Session 启动托管网络代理（start_managed_network_proxy），代理可回调 core 做 allowlist-miss 决策（network_policy_decider）；ToolOrchestrator 中 managed_network_active 决定是否在沙箱尝试中 enforce_managed_network，网络拒绝时产生 network_approval_context 用于升级重试审批
- **data_form**: 网络请求 → 托管代理 → allowlist Gate → 拒绝/放行
- **symbols**: [NetworkProxySpec, managed_network_proxy, network_policy_decider, NetworkApprovalService, begin_network_approval]
- **strength**: S3

---

## 二、Doc Evidence Pack（doc-analyst）

### EV-D001 · AGENTS.md 明确禁止修改沙箱环境变量相关代码
- **stage**: doc
- **discovered_by**: doc-analyst
- **source**: AGENTS.md:8-10
- **claim**: AGENTS.md 规定"Never add or modify any code related to CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR or CODEX_SANDBOX_ENV_VAR"，因为 Agent 自身运行在沙箱中，这些环境变量由沙箱运行时设置，用于提前退出无法在沙箱中运行的测试
- **layer**: design_intent
- **symbols**: [CODEX_SANDBOX_NETWORK_DISABLED, CODEX_SANDBOX, Seatbelt]
- **strength**: S2

### EV-D002 · AGENTS.md 规定 core crate 反膨胀原则
- **stage**: doc
- **discovered_by**: doc-analyst
- **source**: AGENTS.md:72-83
- **claim**: AGENTS.md 明确指出"codex-core crate has become bloated because it is the largest crate"，规定"resist adding code to codex-core"，引入新概念时应考虑是否有现有 crate 可放或应新建 crate，code review 时应 push back 不必要的 core 添加
- **layer**: design_intent
- **symbols**: [codex-core, crate 拆分]
- **strength**: S2

### EV-D003 · AGENTS.md 规定模型可见上下文的硬限制
- **stage**: doc
- **discovered_by**: doc-analyst
- **source**: AGENTS.md:91-100
- **claim**: AGENTS.md 规定模型上下文的 6 条硬规则：1) No history rewrite（增量构建）/ 2) Avoid frequent changes causing cache misses / 3) No unbounded items（一切注入有界+硬上限）/ 4) No items larger than 10K tokens / 5) >1K tokens 的新项标记 P0 需人工审查 / 6) 所有注入片段必须在 core/context 定义为 struct 并实现 ContextualUserFragment trait
- **layer**: design_intent
- **symbols**: [ContextualUserFragment, context window, token budget]
- **strength**: S2

### EV-D004 · AGENTS.md 规定变更大小限制（800行/500行）
- **stage**: doc
- **discovered_by**: doc-analyst
- **source**: AGENTS.md:125-131
- **claim**: AGENTS.md 规定"Unless the change is mechanical the total number of changed lines should not exceed 800 lines. For complex logic changes the size should be under 500 lines."，更大的变更应拆分为可审查阶段
- **layer**: design_intent
- **strength**: S2

### EV-D005 · AGENTS.md 规定 Rust trait 异步方法的原生 RPITIT 优先
- **stage**: doc
- **discovered_by**: doc-analyst
- **source**: AGENTS.md:23-28
- **claim**: AGENTS.md 规定"Discourage both #[async_trait] and #[allow(async_fn_in_trait)] in Rust traits. Prefer native RPITIT trait methods with explicit Send bounds on the returned future"，首选形状为 `fn foo(&self, ...) -> impl std::future::Future<Output = T> + Send;`
- **layer**: design_intent
- **symbols**: [RPITIT, async_trait, Send bound]
- **strength**: S2

### EV-D006 · 项目文档倾向外链到开发者文档站
- **stage**: doc
- **discovered_by**: doc-analyst
- **source**: docs/sandbox.md:1-3, docs/execpolicy.md:1-3
- **claim**: docs/sandbox.md 和 docs/execpolicy.md 都只有一句话+外链（"For information about Codex sandboxing and approvals, see this documentation"），AGENTS.md:32 也规定"Do not add general product or user-facing documentation to the docs/ folder. The official Codex documentation lives elsewhere."，仓库内 docs/ 只保留开发者贡献相关文档
- **layer**: design_intent
- **strength**: S2

---

## 三、Test Evidence Pack（test-analyst）

### EV-T001 · 测试组织采用独立测试文件 + #[path] 属性
- **stage**: test
- **discovered_by**: test-analyst
- **source**: AGENTS.md:167-178
- **claim**: AGENTS.md 规定新增测试模块时内容放在独立的兄弟文件中，用 `#[path = "..._tests.rs"]` 属性显式引用，使测试文件名描述性强且易定位；仅对新增模块适用，不强制迁移现有内联测试
- **symbols**: [#[path = "..._tests.rs"], parser_tests.rs]
- **strength**: S3

### EV-T002 · 集成测试优先于单元测试，使用 test_codex 框架
- **stage**: test
- **discovered_by**: test-analyst
- **source**: AGENTS.md:112-123
- **claim**: AGENTS.md 规定"For agent changes prefer integration tests over unit tests. Integration tests are under core/suite and use test_codex to set up a test instance of codex"，改变 Agent 逻辑的功能 MUST 添加集成测试；测试辅助工具在 core_test_support::responses，提供 mount_sse_once / ResponseMock / wait_for_event 等
- **symbols**: [test_codex, core/suite, core_test_support::responses, mount_sse_once, ResponseMock]
- **strength**: S3

### EV-T003 · TUI 使用 insta snapshot 测试，UI 变更必须更新 snapshot
- **stage**: test
- **discovered_by**: test-analyst
- **source**: AGENTS.md:180-198
- **claim**: AGENTS.md 规定仓库使用 insta snapshot 测试（尤其在 codex-rs/tui），任何影响用户可见 UI 的变更必须包含对应的 insta snapshot 覆盖，snapshot 更新作为 PR 一部分审查接受，使 UI 影响易审查且未来 diff 可视化
- **symbols**: [insta, snapshot test, codex-tui]
- **strength**: S3

### EV-T004 · 测试断言使用 pretty_assertions + 深比较
- **stage**: test
- **discovered_by**: test-analyst
- **source**: AGENTS.md:210-214
- **claim**: AGENTS.md 规定测试使用 pretty_assertions::assert_eq 获得更清晰 diff，优先对整个对象做深比较而非逐字段比较；避免在测试中变更进程环境，优先从上方传入环境派生的标志
- **symbols**: [pretty_assertions, assert_eq!, 深比较]
- **strength**: S3

### EV-T005 · Guardian 熔断器有独立测试文件
- **stage**: test
- **discovered_by**: test-analyst
- **source**: codex-rs/core/src/guardian/mod.rs:274-275, session.rs:49
- **claim**: guardian 模块有独立的 tests 模块（mod tests;），Session 的 thread_settings_persistence 用 Semaphore(1) 序列化设置提交与持久化事件，表明并发安全是测试覆盖的重点
- **symbols**: [GuardianRejectionCircuitBreaker, Semaphore, tests]
- **strength**: S3

---

## 四、Failure Evidence Pack（failure-analyst）

### EV-F001 · AGENTS.md 记录沙箱环境变量是已知认知成本区域
- **stage**: failure
- **discovered_by**: failure-analyst
- **source**: AGENTS.md:8-10
- **claim**: CODEX_SANDBOX_NETWORK_DISABLED 和 CODEX_SANDBOX 环境变量是 Agent 自身沙箱运行时的产物，代码中用它们提前退出无法在沙箱中运行的测试（如需要自己 spawn Seatbelt 的集成测试），这是系统在沙箱自举问题上付出的认知成本
- **symbols**: [CODEX_SANDBOX_NETWORK_DISABLED, CODEX_SANDBOX=seatbelt, early exit]
- **strength**: S3
- **lesson**: Agent 系统自举时，沙箱内测试沙箱功能是递归问题，需要环境变量作为逃逸阀

### EV-F002 · Session 初始化中存在多处 TODO 标记技术债
- **stage**: failure
- **discovered_by**: failure-analyst
- **source**: session.rs:102-103, 115, 1433
- **claim**: session.rs 中有 3 处 TODO：1) "TODO(anp): Reconcile these legacy thread defaults with TurnEnvironment::sandbox_context; internal sandbox decisions should use the selected environment's configuration" / 2) "TODO(pakrym): Remove config from here"（original_config_do_not_use 字段名本身就是技术债标记）/ 3) "TODO(jif): extract session to share between sub-agents"，表明沙箱配置统一、config 依赖注入、子 Agent session 共享是已知未解决问题
- **symbols**: [TODO(anp), TODO(pakrym), TODO(jif), original_config_do_not_use]
- **strength**: S3
- **lesson**: 大型 Rust 项目中，字段名加 _do_not_use 后缀是防止继续扩散技术债的防御性命名

### EV-F003 · orchestrator 中 build_denial_reason_from_output 是占位实现
- **stage**: failure
- **discovered_by**: failure-analyst
- **source**: orchestrator.rs:542-546
- **claim**: build_denial_reason_from_output 函数忽略输入 output，固定返回 "command failed; retry without sandbox?"，注释说明"Keep approval reason terse and stable for UX/tests, but accept the output so we can evolve heuristics later without touching call sites"，这是为 UX 稳定性刻意保留的简化实现
- **symbols**: [build_denial_reason_from_output, placeholder]
- **strength**: S3
- **lesson**: 面向用户的错误信息刻意保持简单，预留输出参数供未来启发式升级

### EV-F004 · Guardian 采用"失败关闭"（fail closed）安全策略
- **stage**: failure
- **discovered_by**: failure-analyst
- **source**: guardian/mod.rs:12-13
- **claim**: Guardian review 的高层方法第 3 条明确"Fail closed on timeout, execution failure, or malformed output"，即 Guardian 审查超时、执行失败或输出格式错误时默认拒绝审批，这是安全优先的设计选择
- **symbols**: [fail closed, GUARDIAN_REVIEW_TIMEOUT=90s]
- **strength**: S3
- **lesson**: AI 自动审批系统必须 fail closed，因为误放的代价高于误拒

---

## 五、Authority Evidence Pack（authority-analyst）

### EV-A001 · 三级审批策略：Never / OnRequest / AlwaysAsk
- **stage**: authority
- **discovered_by**: authority-analyst
- **source**: orchestrator.rs:135, 170-230, guardian/mod.rs:1-3
- **claim**: approval_policy 有三态（AskForApproval）：Never（Full Access，禁用沙箱时不审查直接批准）/ OnRequest（需要时请求，Guardian 可自动批准）/ AlwaysAsk（始终请求用户）。ToolOrchestrator 根据 default_exec_approval_requirement(approval_policy, file_system_sandbox_policy) 决定每个工具的 ExecApprovalRequirement（Skip/Forbidden/NeedsApproval）
- **data_form**: approval_policy → ExecApprovalRequirement → 审批 Gate
- **symbols**: [AskForApproval, ExecApprovalRequirement, default_exec_approval_requirement, Full Access]
- **strength**: S3

### EV-A002 · Guardian 是独立的 AI 审查 session，克隆父配置继承网络策略
- **stage**: authority
- **discovered_by**: authority-analyst
- **source**: guardian/mod.rs:1-13, 10-11
- **claim**: Guardian review 通过 spawn 一个独立的 guardian review session 来评估计划动作，返回严格 JSON（GuardianAssessment: risk_level / user_authorization / outcome / rationale）；Guardian 克隆父配置，因此继承父 turn 已有的托管网络代理/allowlist；超时 90 秒，失败关闭
- **data_form**: on-request approval → Guardian session → JSON 评估 → allow/deny
- **symbols**: [GuardianReviewSessionManager, GuardianAssessment, GuardianAssessmentOutcome, GUARDIAN_REVIEWER_NAME="guardian", BUNDLED_GUARDIAN_POLICY]
- **strength**: S3

### EV-A003 · Guardian 熔断器防止连续拒绝导致无限循环
- **stage**: authority
- **discovered_by**: authority-analyst
- **source**: guardian/mod.rs:64-68, 169-246
- **claim**: GuardianRejectionCircuitBreaker 跟踪每 turn 的连续拒绝数和最近 50 次审查中的拒绝数，Standard 策略下连续 3 次或最近 10 次拒绝触发 InterruptTurn；CyberModel 策略更严格（连续 1 次或最近 1 次即中断）；熔断器防止 Agent 在 Guardian 持续拒绝时无限重试
- **data_form**: Guardian 拒绝 → 熔断器计数 → 超限 InterruptTurn
- **symbols**: [GuardianRejectionCircuitBreaker, MAX_CONSECUTIVE_GUARDIAN_DENIALS_PER_TURN=3, MAX_RECENT_AUTO_REVIEW_DENIALS_PER_TURN=10, AUTO_REVIEW_DENIAL_WINDOW_SIZE=50]
- **strength**: S3

### EV-A004 · 沙箱执行权限与文件系统权限解耦
- **stage**: authority
- **discovered_by**: authority-analyst
- **source**: orchestrator.rs:160-168, session.rs:212-235
- **claim**: executor_managed_process_sandbox 标志区分执行器自管沙箱和核心管沙箱：执行器自管时 permission_profile 保持符号化（不 materialize workspace_roots），由执行器自己应用沙箱；否则核心 materialize workspace_roots 后传给沙箱。这实现了沙箱执行权限与文件系统权限声明的解耦
- **data_form**: PermissionProfile → executor-managed（符号化）/ core-managed（materialized）→ Sandbox
- **symbols**: [executor_managed_process_sandbox, permission_profile_with_workspace_roots, materialize_project_roots]
- **strength**: S3

### EV-A005 · 网络访问有独立的审批通道和延迟确认机制
- **stage**: authority
- **discovered_by**: authority-analyst
- **source**: orchestrator.rs:66-123, 15-18
- **claim**: 网络审批通过 begin_network_approval 独立于工具审批启动，产生 ActiveNetworkApproval（含 cancellation_token 和 execution_proxy）；执行成功后网络审批转为 DeferredNetworkApproval，由 finish_deferred_network_approval 在工具完成后最终确认；执行失败则立即 finalize。这实现了网络访问的"先放行后确认"延迟审批模式
- **data_form**: 工具执行 → begin_network_approval → ActiveNetworkApproval → 执行 → DeferredNetworkApproval → finish
- **symbols**: [begin_network_approval, ActiveNetworkApproval, DeferredNetworkApproval, finish_deferred_network_approval, NetworkApprovalService]
- **strength**: S3

### EV-A006 · 附件拥有的网络策略不可被沙箱升级绕过
- **stage**: authority
- **discovered_by**: authority-analyst
- **source**: orchestrator.rs:149-159
- **claim**: 当 environment 的 config 有 owner_network_policy（附件拥有的网络策略）且工具请求 requires_escalated_permissions 时，ToolOrchestrator 直接拒绝并返回错误"attachment-owned network policy cannot be bypassed by sandbox escalation"，即沙箱升级重试机制不能绕过附件级别的网络限制
- **data_form**: owner_network_policy + escalated_permissions → 直接拒绝
- **symbols**: [owner_network_policy, requires_escalated_permissions, ToolError::Rejected]
- **strength**: S3

---

## 六、Evidence Pack 统计

| 角色 | 证据数 | 强度分布 |
|------|--------|---------|
| code-analyst | 7 | S3 × 7 |
| doc-analyst | 6 | S2 × 6 |
| test-analyst | 5 | S3 × 5 |
| failure-analyst | 4 | S3 × 4 |
| authority-analyst | 6 | S3 × 6 |
| **合计** | **28** | S2 × 6, S3 × 22 |

**高知识密度区域确认**：
- 沙箱与审批系统（orchestrator + guardian + sandboxing）= 最密集
- Session 状态管理（session.rs 1640 行）= 核心运行时
- 多 Agent 控制（agent/control/）= 新兴复杂区域
- AGENTS.md 治理规则 = 工程决策的集中体现

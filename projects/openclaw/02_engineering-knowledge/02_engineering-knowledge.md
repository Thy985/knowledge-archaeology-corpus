# 02 Engineering Knowledge — OpenClaw（宽底座 · EK Graph）

> 每条 EK 声明 links（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）。证据锚点全部可回溯。

## EK-01 默认值漂移即权限变更（升级即扩权）
- **Observation**：2026.9.2 中 `tools.sessions.visibility` 省略值从 `agent` 改为 `all`，`tools.agentToAgent.enabled` 省略值从 `false` 改为 `true`。
- **Evidence**：docs/releases/2026.9.2.md:3245 "Omitted `tools.sessions.visibility` changes from `agent` to `all`, and omitted `tools.agentToAgent.enabled` changes from `false` to `true`, allowing agents with the relevant tools to list, read, search, message, and inspect other agents' sessions."；schema.help.runtime.ts:117-121；zod-schema.agent-runtime.ts:770 "Default: \"all\""。
- **Why**：为了让"agents can cooperate across one Gateway without first enabling two settings"（docs/releases/2026.9.2.md:3245）——协作便利优先，用升级说明+审计提示补偿。
- **links**：causal → EK-02（audit 暴露）；causal → EK-03（remediation 收紧路径）；constraint → EK-07（SECURITY.md 信任模型）
- **value**：A（默认权限面变更 = 升级风险控制点）

## EK-02 安全审计是暴露层，不是门禁（fail-open 审计）
- **Observation**：跨 agent 会话访问默认只报 `info`（有 trust-boundary signal 升 `warn`），不 fail。
- **Evidence**：src/security/audit-extra.summary.ts:241-251 `checkId: "security.trust_model.cross_agent_session_access_default"`，`severity: signals.length > 0 ? "warn" : "info"`；测试 audit-cross-agent-session-access.test.ts 断言 `severity: "info"`。
- **Why**：单 operator local-first 工具的定位（SECURITY.md）——审计提示 operator 自决，不做强门禁。
- **links**：mechanism ↔ EK-06（都属 security audit 子系统）；contrast ↔ EK-04（fail-closed 的对照组是 AGT/Guardian 类门禁式审计）
- **value**：A（与"门禁式"防火墙项目形成强对照）

## EK-03 收紧路径完整但需显式配置（fail-open 的补偿）
- **Observation**：官方 remediation 提供四条收紧路径：visibility=agent/tree/self、agentToAgent.allow 白名单、agentToAgent.enabled=false、独立 Gateway。
- **Evidence**：docs/releases/2026.9.2.md:3247 "set `tools.sessions.visibility` to `agent` … or `self` … set an explicit `tools.agentToAgent.allow` list … or set `tools.agentToAgent.enabled` to `false`"；audit-extra.summary.ts remediation 字段（:253-256）。
- **Boundary**：`tree`/`all` 下 requester-owned 原生 subagent/ACP child 在 agentToAgent=false 时仍可达（schema.help.runtime.ts:117）——**收紧不彻底**，官方明示。
- **links**：causal → EK-01；constraint → EK-05（visibility 语义）；contrast ↔ EK-04
- **value**：A

## EK-04 权限面是"默认开放 + 显式收紧"模型（与 fail-closed 系统相反）
- **Observation**：OpenClaw 的 session 访问/跨 agent/swarm/跨 provider 消息全部默认开启，收紧靠显式配置。
- **Evidence**：session-visibility.ts:244-276（agentToAgent enabled 默认 true、allow 空=全允许、blank entries deny）；tool-permissions.md（allowAcrossProviders 默认 true）；swarm-config.ts（enabled 默认 true）。
- **Why**：单 operator local-first 工具（SECURITY.md），协作/便利优先；与"多租户对抗边界"设计目标显式排除。
- **links**：mechanism ↔ EK-01；contrast ↔ AGT 轮 KO-02（fail-closed 无开关）
- **value**：A

## EK-05 session visibility 四档语义（self/tree/agent/all）
- **Observation**：visibility 决定会话工具可及范围；`agent` 仍可含同 agent 下其他用户会话；`tree` 限本会话 + 本会话 spawn 的会话；`all` 限由 agentToAgent 治理。
- **Evidence**：zod-schema.agent-runtime.ts:770-780（四档注释）；session-visibility.ts:148-161（resolveSessionToolsVisibility，invalid→"all" fail-open 解析）。
- **Edge**：invalid visibility 解析到 "all"（fail-open 解析方向），测试 audit-cross-agent-session-access.test.ts "invalid visibility resolving to all" 明确覆盖。
- **links**：causal → EK-03；mechanism ↔ EK-08（同属 session 访问控制）
- **value**：A

## EK-06 agentToAgent 白名单编译为线性 glob 匹配（防正则回溯）
- **Observation**：allow 模式编译为 all/deny/exact/wildcard 四类，通配符用线性前缀/后缀/内部段匹配，不进入 regex 引擎。
- **Evidence**：session-visibility.ts:189-243（compileAgentAllowPattern + matchesCompiledWildcard，注释 "avoiding polynomial backtracking on repeated wildcards"）；blank entry → deny（fail-closed 单点）。
- **Why**：通配符正则回溯是 DoS 风险（ReDoS），线性匹配规避。
- **links**：subsystem ↔ EK-01/EK-03/EK-05（session 访问控制子系统）；constraint → EK-05
- **value**：B

## EK-07 信任模型明示"会话可见性不是安全边界"（usability ≠ security）
- **Observation**：SECURITY.md 原话 "Anyone who can operate an agent can make it do anything that agent can do. Session ownership, visibility, and presence are usability features, not security boundaries."
- **Evidence**：SECURITY.md "Shared Agents" 段；docs/gateway/security/access-control.md:61 "If users are mutually adversarial and share the same Gateway host/config, run separate gateways per trust boundary instead."
- **Why**：turn 归因 best-effort（steering 可把输入并入激活 turn），单 Gateway 内无法保证真实隔离。
- **links**：constraint → EK-01/03/04（所有默认值变更都在"非安全边界"前提下）；contrast ↔ AGT 轮（多租户对抗模型）
- **value**：A（设计意图 vs 实现现实的核心锚点）

## EK-08 swarm 默认开启 + 有界容量治理
- **Observation**：tools.swarm.enabled 默认 true；maxConcurrent=8/maxChildrenPerGroup=50/maxTotalPerGroup=200/waitTimeoutSecondsMax=600，全部有界（readBoundedPositiveInteger 强制上限）。
- **Evidence**：swarm-config.ts DEFAULT_SWARM_CONFIG（:11-16）+ readBoundedPositiveInteger（:30-34）；CHANGELOG/2026.9.2.md "Swarm is enabled by default: orchestrate concurrent sub-agents with structured results and live progress, while preserving explicit opt-outs, tool restrictions, and the separate Code Mode opt-in."
- **links**：causal → EK-09（scheduler lane）；subsystem → EK-09/EK-10（swarm 子系统）
- **value**：A

## EK-09 SwarmGroupLane 容量队列 + AsyncLocalStorage 激活身份
- **Observation**：swarm-scheduler.ts 用 SwarmGroupLane（groupId/limit/active/queue/pumpScheduled）做每 lane 并发队列；bindSwarmLaunchWork 用 AsyncLocalStorage 保持激活身份且不重入已退役请求的 work scope。
- **Evidence**：swarm-scheduler.ts:30-50（SwarmGroupLane 定义）+ bindSwarmLaunchWork（:52-65）。
- **links**：dependency → EK-08（容量参数驱动 lane limit）；mechanism ↔ AGT 轮（并发 lane 队列模式）
- **value**：B

## EK-10 swarm collector 结果契约（structured results + 状态机）
- **Observation**：swarm-collector.ts resolveStatus 状态机：killed→"killed"；timeout→"timeout"；ok→"done"；"tool-only structured turns can surface the runner's synthetic completion marker as an error despite having fulfilled the collector contract"→"done"（容忍合成完成标记）。
- **Evidence**：swarm-collector.ts:20-37（resolveStatus 注释与分支）。
- **links**：subsystem ↔ EK-08/09；causal → EK-15（registry 交付状态）
- **value**：B

## EK-11 工具权威准备先于发布（withPreparedEmbeddedRunToolAuthority）
- **Observation**：tool-authority.runtime.ts 声明 "Execution-only: policy preparation must finish before authority reaches a publisher"——toolAuthorityFingerprint/admitted-run-context 机制保证 policy 先于 authority。
- **Evidence**：src/agents/harness/tool-authority.runtime.ts:37-60（注释 + 实现）。
- **links**：mechanism ↔ AGT 轮 EK-37/38（authority resolver/advisory block）；constraint → EK-14（spawn 权威链）
- **value**：A（执行面与策略面分离）

## EK-12 spawn 管线三阶段（initialize→dispatch→register）
- **Observation**：runSpawnPipeline 三阶段带 cleanupOnFailure（每阶段失败清理）；progressSessionKey 区分 native requester key vs ACP completion-owner key（"do not collapse them"）。
- **Evidence**：src/agents/spawn-pipeline.ts:20-60（SpawnPipelinePhase + progressSessionKey 注释）。
- **links**：causal → EK-13（accepted child 归一化）；subsystem → EK-11/EK-13（spawn 子系统）
- **value**：B

## EK-13 accepted-session-spawn 归一化（child 归属/完成语义）
- **Observation**：normalizeAcceptedSessionSpawnResult 只认 status="accepted" + runId + childSessionKey；expectsCompletionMessage 标记 child 是否拥有 requester 的 terminal completion。
- **Evidence**：src/agents/accepted-session-spawn.ts:20-45。
- **links**：dependency → EK-12；causal → EK-15（registry）
- **value**：C

## EK-14 authority 散点权威链（cron/sandbox/auto-reply/transcript）
- **Observation**：authority 分散在多个 owner：cron-creator-authority-context、workspace-authority、requester-cron-authority、reply-tool-authority、session-writer-delivery-authority、message-injection-authority、transcript-byte-preflight-authority、node-execution-authority。
- **Evidence**：find src -name "*authority*.ts"（8 个 authority 文件 + tool-authority.runtime）。
- **links**：mechanism ↔ EK-11（都是 authority 前置）；subsystem → EK-12
- **value**：B

## EK-15 subagent registry 交付状态机（endReason/execution.outcome/collect）
- **Observation**：registry 用 SubagentRunRecord（endedReason/execution.outcome/collect）跟踪子 agent 生命周期；SUBAGENT_ENDED_REASON_KILLED、timeout/ok 等枚举。
- **Evidence**：src/agents/subagents/registry/*（subagent-registry.types.ts / subagent-delivery-state.ts / subagent-lifecycle-events.ts）。
- **links**：causal ← EK-10/EK-13；subsystem → EK-08-10
- **value**：C

## EK-16 权限模式四档（read-only/guarded/workspace/full）
- **Observation**：四种 session 权限模式：filesystem 边界 = 记录的 sessionRoot（或 agent canonical workspace）；exec 升级审阅人不同——read-only 无 exec；guarded human-after-allowlist；workspace LLM reviewer（allow/deny/ask human，3 连拒升 human）；full 需 operator.admin。
- **Evidence**：docs/gateway/permission-modes.md（模式表 + "Three consecutive gateway reviewer denials also escalate to a human" + "full requires operator.admin"）。
- **links**：constraint → EK-17（control plane 工具）；mechanism ↔ 权限分层模式（跨项目对照）
- **value**：A

## EK-17 control plane 工具 owner-only（gateway/cron）
- **Observation**：gateway 工具 owner-only（config 读暴露 secrets + update.run 改安装）；cron 创建常驻任务；处理不可信内容的 agent 默认 deny [gateway, cron, sessions_spawn, sessions_send]。
- **Evidence**：docs/gateway/security/tool-permissions.md "Two built-in tools remain control-plane sensitive" + deny 清单 JSON。
- **links**：causal → EK-16；constraint → EK-04（默认开放的例外）
- **value**：A

## EK-18 跨 provider 消息默认开放（allowAcrossProviders=true）
- **Observation**：有 message 工具权限的 agent 可默认跨会话/跨 provider 发送；`tools.message.crossContext.allowAcrossProviders` 默认 true，可显式 false。
- **Evidence**：docs/gateway/security/tool-permissions.md "Cross-provider messaging … defaults to `true`"。
- **links**：mechanism ↔ EK-04（默认开放族）；constraint → EK-16
- **value**：B

## EK-19 approval channel custody（channel 审批身份绑定）
- **Observation**：prepareApprovalChannelCustody 把 channel/accountId/senderId 绑定到 channel 插件的 authorizeActorAction 能力——审批动作按 actor 授权。
- **Evidence**：src/gateway/approval-channel-custody.ts:20-45（authorizes 闭包）。
- **links**：subsystem → EK-16（approval 子系统）；mechanism ↔ 09-16 AGT 轮 advisory 审批
- **value**：B

## EK-20 AGENTS.md 治理契约（one owner per responsibility）
- **Observation**：AGENTS.md 明确"One owner per responsibility. An owner makes a decision or changes authoritative state. Callers consume its operations and recorded facts."；特权动作需当前 owner 持有的 authority，await 后/副作用前重新验证。
- **Evidence**：AGENTS.md "Design priorities" + "Runtime and code safeguards" 段。
- **links**：constraint → 全部 EK（治理前提）；mechanism ↔ 09-13 Aigis 轮（owner 治理模式）
- **value**：B

## EK-21 运行时选择：模型/provider 双维度 + auto
- **Observation**：runtime policy 是 model/provider-scoped `agentRuntime.id`（model 优先）；auto 选注册 harness，否则内置 openclaw；OpenAI 精确官方 HTTPS 路由隐式选 codex。
- **Evidence**：docs/agent-runtime-architecture.md "Runtime Selection" 段。
- **links**：subsystem → 01 层（运行时架构）；mechanism ↔ 插件 harness 模式
- **value**：C

## EK-22 Code Mode 是独立开关（fail-closed 运行时）
- **Observation**：tools.codeMode.enabled 省略=off；"auto" 用 catalog-preferred 模型；engaged run 在 runtime 不可用时 fail closed（不暴露完整工具列表）。
- **Evidence**：schema.help.runtime.ts:138-139（"An engaged run fails closed if the runtime is unavailable instead of exposing the full tool list"）。
- **links**：contrast ↔ EK-04（Code Mode 是默认关闭的例外）；mechanism ↔ 09-16 AGT 轮（fail-closed 运行时）
- **value**：A（OpenClaw 中罕见的 fail-closed 面）

## EK-23 记忆是 agent-scoped，会话搜索是 permission-scoped
- **Observation**：docs/releases/2026.9.2.md:3249 "memory_search remains agent-scoped while sessions_search searches transcripts within the permitted scope"。
- **Evidence**：同上（权限变更说明中的边界清单）。
- **links**：constraint → EK-05；contrast ↔ EK-01（memory 与 session 权限模型不同）
- **value**：B

## EK-24 sandbox clamp 只限制调用方，不隐藏其 transcript
- **Observation**：sandbox 限制 sandboxed caller 可达范围，但不隐藏其 transcript 不被其他 permitted caller 读取；sandbox.mode=all 时 sessionToolsVisibility clamp 到 tree。
- **Evidence**：docs/releases/2026.9.2.md:3249 "Sandboxing restricts what the sandboxed caller can reach; it does not hide its transcripts from another permitted caller"；session-visibility.ts:175-186（resolveEffectiveSessionToolsVisibility clamp→tree）。
- **links**：constraint → EK-05/EK-07；causal → EK-03（收紧仍留缝）
- **value**：A（安全语义的关键反直觉点）

## EK-25 invalid config 解析 fail-open（visibility invalid→all）
- **Observation**：resolveSessionToolsVisibility 对非四档值返回 "all"（fail-open 解析方向）；测试显式覆盖。
- **Evidence**：session-visibility.ts:148-161；audit-cross-agent-session-access.test.ts "invalid visibility resolving to all" 用例。
- **links**：constraint → EK-05；contrast ↔ EK-22（Code Mode fail-closed）
- **value**：A（与 fail-closed 原则直接对撞的实例）

## EK-26 信任边界 signal 检测（multi-user ingress）
- **Observation**：audit 通过 listPotentialMultiUserSignals 检测共享用户入口等信号，有 signal 时 severity 升 warn。
- **Evidence**：audit-extra.summary.ts:180-251（signals 收集 + "Trust-boundary signals" 段）。
- **links**：causal → EK-02；subsystem → EK-02/EK-06
- **value**：B

## EK-27 升级迁移语义（omitted → new default）
- **Observation**：docs/releases/2026.9.2.md:3245 "leaving these settings unset on an older installation now permits broader access on upgrade"——升级不写配置也生效（迁移窗口内默认值漂移）。
- **Evidence**：同上；schema.help.runtime.ts:60（"Default: true for one migration window"——同模式案例）。
- **links**：causal → EK-01；constraint → 用户升级决策（campus_order）
- **value**：A

## EK-28 空 allow 列表语义：未设置=全允许，配置后空=拒绝
- **Observation**：allow 省略或空=未设置（全允许）；**配置了空数组后**，blank entries 编译为 deny——"configured-but-blank list still fails closed"。
- **Evidence**：session-visibility.ts:244-276（"Omitted or empty counts as unset: every agent pair is allowed by default; blank entries deny" 注释 + allowPatterns.length===0 → true）。
- **links**：mechanism ↔ EK-06；constraint → EK-03
- **value**：A（微妙语义，审计测试明确覆盖 "explicit all visibility and empty allow list" 报 1 条）

## EK-29 node-runtime-recovery（入口自修复）
- **Observation**：openclaw.mjs 检测 Node 版本不匹配时 recoverNodeRuntime（找兼容副本或提供仅 OpenClaw 用安装）。
- **Evidence**：openclaw.mjs:1-30 + node-runtime-recovery.mjs；CHANGELOG/2026.9.4.md "Start OpenClaw when Node needs attention"。
- **links**：subsystem → 01 层（入口）；mechanism ↔ 09-16 AGT 轮（安装自愈）
- **value**：C

## EK-30 测试揭示行为：审计测试表驱动 7+4 例
- **Observation**：audit-cross-agent-session-access.test.ts 用 it.each 表驱动：7 例不 flag（含 sandbox all、session tools deny 全移除）+ 4 例报 1 条 info（含 invalid visibility）。测试揭示：审计"检测默认暴露"而非"检测暴露本身"。
- **Evidence**：测试文件 it.each 两段。
- **links**：mechanism ↔ EK-02；dependency → EK-25（invalid visibility 用例）
- **value**：B

## EK-31 内存/状态 SQLite 单一权威（Kysely 同步事务）
- **Observation**：AGENTS.md "OpenClaw state and caches use SQLite, not new JSON/JSONL/sidecar stores" + "Use Kysely for ordinary SQLite access" + 同步写事务。
- **Evidence**：AGENTS.md "Runtime and code safeguards" 段。
- **links**：constraint → 01 层（状态架构）；mechanism ↔ 09-16 AGT 轮（状态单一权威）
- **value**：B

## EK-32 计算 worker 容量准入（WorkerTaskPool）
- **Observation**：code-mode 执行与 compaction 用 WorkerTaskPool，共享 CPU 准入 max(1, availableParallelism()-1)；超载失败 WorkerTaskError.code="overloaded"。
- **Evidence**：docs/agent-runtime-architecture.md "Compute workers" 段。
- **links**：mechanism ↔ EK-08（容量治理）；subsystem → 01 层
- **value**：C

## EK-33 构建缓存/入口快速路径（entry.compile-cache / version-fast-path）
- **Observation**：src/entry.compile-cache.ts / entry.version-fast-path.ts / entry.memory-json.ts 等——入口层对编译/版本/内存 JSON 做快速路径缓存。
- **Evidence**：src/entry.*.ts 文件集。
- **links**：subsystem → 01 层；mechanism ↔ EK-32（性能治理）
- **value**：D

## EK-34 swarm 结构化输出与 synthetic completion 容错
- **Observation**：consumeSwarmStructuredOutput 双 key（runId/swarmRunId）取结构化结果；collector 对 "completed" 合成错误标记按 done 处理。
- **Evidence**：swarm-collector.ts:30-37 + consumeSwarmStructuredOutput 调用。
- **links**：causal ← EK-10；subsystem → EK-08
- **value**：C

## EK-35 session 可见性决策状态机（内部拒绝原因枚举）
- **Observation**：session-visibility-internal.ts 定义拒绝原因：agent_to_agent_disabled / agent_to_agent_not_allowed / cross_agent_visibility_restricted / self_visibility_restricted / tree_visibility_restricted / target_agent_ownership_unavailable。
- **Evidence**：session-visibility-internal.ts:44-53。
- **links**：mechanism ↔ EK-05/06；constraint → EK-03
- **value**：B

## EK-36 交付语义：incognito 会话对所有工具隐藏
- **Observation**：docs/releases/2026.9.2.md:3249 "Incognito sessions remain hidden from these tools"；审计 detail 也注明 "Incognito sessions remain hidden"。
- **Evidence**：audit-extra.summary.ts:246。
- **links**：constraint → EK-05；contrast ↔ EK-01（唯一例外面）
- **value**：C

## EK-37 共享 Gateway 升级提示是官方一等公民
- **Observation**：2026.9.2 发布说明顶部即有 "Upgrade note for shared Gateways" 加粗框。
- **Evidence**：docs/releases/2026.9.2.md:11。
- **links**：causal → EK-27；constraint → 用户升级流程
- **value**：B

## EK-39 子 agent 工具 deny-wins（SUBAGENT_TOOL_DENY_ALWAYS）
- **Observation**：子 agent 无论深度都禁止 12 个工具：gateway/agents_list/openclaw/session_status/progress_card/automations/message/sessions_send/conversations_list/send/turn；leaf 额外禁 subagents/sessions_list/history/search/spawn。**跨 agent 会话访问默认开放仅适用于主 agent 之间，子 agent 层是 deny-wins。**
- **Evidence**：src/agents/agent-tools.policy.ts:26-42（SUBAGENT_TOOL_DENY_ALWAYS + SUBAGENT_TOOL_DENY_LEAF + resolveSubagentDenyListForRole）。
- **links**：constraint → EK-04（"默认开放"限定主 agent 范围）；contrast ↔ EK-17（控制面工具）
- **value**：A（独立 Auditor 发现的 MISSING，修正 KO-01 泛化）
- **来源**：Independent Validation E-1

## EK-40 ACP 通道 session 工具归控制面审批
- **Observation**：ACP 通道下 sessions_spawn/sessions_send/session_status 归入 CONTROL_PLANE_TOOL_IDS，autoApprove=false（prompt-required）。
- **Evidence**：src/acp/approval-classifier.ts:26-32 + :238-240（CONTROL_PLANE_TOOL_IDS 定义 + control_plane 分类返回）。
- **links**：subsystem → EK-17（控制面工具族）；constraint → EK-05（session 工具权限因通道而异）
- **value**：A（独立 Auditor 发现的 MISSING）
- **来源**：Independent Validation E-2

## EK-41 删除 agent 后 allow 回退 allow-all（配置漂移风险）
- **Observation**：openclaw agents delete 会从 allow 剪除该 id；若列表变空，策略**回退 allow-all**——从"受限"静默变"全允许"。
- **Evidence**：docs/gateway/config-tools/sessions-and-subagents.md:29 "Deleting an agent … prunes its id from allow; if that empties the list, the policy falls back to allow-all"；代码一致（session-visibility.ts:244-276 空列表返回 true）。
- **links**：causal → EK-28（空列表语义的运行时变体）；constraint → EK-03（收紧路径的静默失效点）
- **value**：A（独立 Auditor 发现的 MISSING）
- **来源**：Independent Validation E-3

## EK-38 harness 层权限独立性（原生 harness 保留自有工具面）
- **Observation**：permission-modes.md "Native harnesses can retain their own tool surface under their permission controls; see Codex runtime policy"——OpenClaw-managed 规则不覆盖原生 harness 工具面。
- **Evidence**：docs/gateway/permission-modes.md。
- **links**：constraint → EK-16；contrast ↔ EK-11（内置 vs 插件权限面）
- **value**：B

---
## EK Graph 统计
- EK 总数：41
- 六类边：mechanism（EK-02/04/11/14/19/20/30/31/22/32 等）、subsystem（EK-02/06/08/09/10/12/14/15/19/21/26/29/32/33/34/40）、causal（EK-01/03/08/09/10/12/13/15/17/24/26/27/34/37/41）、dependency（EK-09/13/30）、constraint（EK-01/03/05/06/07/14/16/17/18/23/24/25/28/35/36/38/39/40/41）、contrast（EK-02/03/04/07/22/25/36/38/39）
- 平均出边：>1（全部 ≥1，无游离 EK）

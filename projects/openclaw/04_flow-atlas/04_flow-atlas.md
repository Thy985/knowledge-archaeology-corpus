# 04 Flow Atlas — OpenClaw（七类流，从真实代码导出）

> 每条 Edge 可回溯 symbol/file/condition/state transition。所有流基于 2026.9.4 快照（HEAD ca38979d）。

## 4.1 Control Flow（控制流：会话工具调用的权限决策）
```
Agent 调用 sessions_list/history/search/send/status
  → src/agents/harness/tool-authority.runtime.ts withPreparedEmbeddedRunToolAuthority
      (admittedRunContext 校验 + toolAuthorityFingerprint 预发布)
  → src/plugin-sdk/session-visibility.ts createSessionVisibilityDecisionChecker
  → resolveSessionToolsVisibility(cfg)        // self|tree|agent|all；invalid→all (EK-25)
  → createAgentToAgentPolicy(cfg).isAllowed(requester, target)
      // enabled!==false 且 requester===target 放行；allow 空→true；blank→deny (EK-28)
  → resolveEffectiveSessionToolsVisibility(cfg, sandboxed)
      // sandbox clamp: spawned→tree (EK-24)
  → SessionAccessResult { allowed | forbidden }
```
- **Edge 真实性**：每个 symbol 均存在于上述文件（tool-authority.runtime.ts:37-60、session-visibility.ts:148-276、session-visibility-internal.ts:44-53）。

## 4.2 State Flow（状态流：session 归属与可见性状态）
```
Session 创建：spawn-pipeline.ts (initialize→dispatch→register)
  → accepted-session-spawn.ts normalizeAcceptedSessionSpawnResult (status=accepted+runId+childSessionKey)
  → AgentRuntimeSessionSpawnContext { completionOwnerSessionKey, inheritedToolPolicy }
  → subagent-registry SubagentRunRecord { endedReason, execution.outcome, collect }
  → swarm-collector resolveStatus: killed|timeout|done|failed (EK-10)
```
- **状态枚举**：SessionVisibilityDecisionMode = self|tree|agent|all；拒绝原因 = agent_to_agent_disabled/not_allowed/cross_agent_visibility_restricted/self_visibility_restricted/tree_visibility_restricted/target_agent_ownership_unavailable。

## 4.3 Data Flow（数据流：配置 → 权限决策 → 审计）
```
openclaw.json / agents.entries.* / tools.*
  → zod-schema.agent-runtime.ts 校验 (OpenClawConfig)
  → session-visibility.ts resolveSessionToolsVisibility / createAgentToAgentPolicy
  → audit-extra.summary.ts collectCrossAgentSessionAccessFindings
      (listAgentIds → resolveSandboxConfigForAgent → resolveToolPolicies → reachers/nonReachers)
  → SecurityAuditFinding { checkId, severity: info|warn, detail, remediation }
  → openclaw security audit CLI 输出
```

## 4.4 Evidence Flow（证据流：审计发现的形成）
```
配置证据：cfg.tools.sessions.visibility / cfg.tools.agentToAgent.{enabled,allow}
  + agent 级 tools.profile/allow/deny
  + agents.defaults.sandbox.sessionToolsVisibility (spawned|all)
  → 判定条件 (audit-extra.summary.ts:180-190):
      agentIds.length<2 → 不报
      visibility!==all → 不报
      agentToAgent.enabled===false 或 allow 有内容 → 不报
      sandbox mode all + clamp=all → 不报
      session tools 被 agent 策略移除 → 不报
  → 否则报 1 条（有 multi-user signal → warn；无 → info）
```

## 4.5 Authority Flow（权威流：谁被授权做什么）
```
operator (owner) 
  → openclaw CLI / gateway 启动 (openclaw.mjs → src/entry.ts)
  → agent entries (agents.entries.*) 各自工具权限 (tools.allow/deny)
  → 权限模式 (permission-modes.md): read-only|guarded|workspace|full
      full 需 operator.admin；其余需 operator.write
  → control-plane: gateway(owner-only) / cron(owner-only) (EK-17)
  → ACP 通道: sessions_spawn/sessions_send/session_status 归 control_plane, autoApprove=false (EK-40)
  → 子 agent: SUBAGENT_TOOL_DENY_ALWAYS 收缩工具面, sessions_send/message 永久 deny (EK-39)
  → 跨 agent: agentToAgent.isAllowed (requester+target 双匹配)
  → 升级：exec 审阅人 human/LLM reviewer (workspace 模式 3 连拒升 human)
```
- **权威边界声明**：SECURITY.md "Anyone who can operate an agent can make it do anything that agent can do"——**operator 权威无上限，agent 权威受工具策略限制，session 可见性不构成权威边界**（EK-07）。
- **补充 Edge（Independent Validation E-1/E-2）**：ACP 通道分支（approval-classifier.ts:26-32,238-240）与子 agent 收缩分支（agent-tools.policy.ts:26-42）均已重读验证。

## 4.6 Memory Flow（记忆流）
```
memory files (docs/concepts/memory-builtin.md)
  → memory_search (hybrid relevance 排序; agent-scoped, EK-23)
  → memory_get / memory_put
  → 与 sessions_search 区分: sessions_search 在 permitted scope 内搜 transcript
  → 状态: SQLite (AGENTS.md: state/caches use SQLite, Kysely 同步事务, EK-31)
```

## 4.7 Policy Flow（策略流：治理闭环）
```
Decision: 2026.9.2 权限模型变更 (swarm 默认 + cross-agent 默认) — CHANGELOG/2026.9.2.md
  → Policy: tools.sessions.visibility=all / agentToAgent.enabled=true / swarm.enabled=true
      (zod-schema.agent-runtime.ts:770-830, swarm-config.ts:11-16)
  → Enforcement: session-visibility.ts 运行时强制 (isAllowed/visibility)
      + audit-extra.summary.ts 静态审计 (info/warn)
  → Future Decision: docs/releases/2026.9.2.md 升级说明 + remediation
      (visibility→agent/self, allow 白名单, enabled=false, 独立 Gateway)
  → Feedback: security audit CLI 输出 → operator 决策 → 配置收紧
```
- **治理闭环完整**：Decision→Policy→Enforcement→Feedback 四段均存在（EK-01/02/03/27 构成闭环）。

## 关键 Flow 缺陷候选（阶段 5 Auditor 已攻击，2 条已补为 Edge）
1. **Flow 缝隙**：agentToAgent=false 后 requester-owned native/ACP child 在 tree/all 下仍可达（schema.help.runtime.ts:117）——收紧路径存在旁路。
2. **Flow 缝隙**：sandbox clamp 不隐藏 transcript（docs/releases/2026.9.2.md:3249）——sandbox 与可见性正交。
3. **Flow 缺口**：invalid visibility 解析到 all（fail-open）——配置错误向放宽方向解析。
4. **Flow 缺口**：allow 空数组语义（未设置=全允许；配置空=deny）；**删除 agent 可致 allow 空→回退 allow-all**（sessions-and-subagents.md:29，Reconciliation 补）。
5. **Flow bypass 候选**：原生 harness（如 codex）保留自有工具面（permission-modes.md）——权限模型存在 harness 级旁路。
6. **Flow bypass（Reconciliation 补）**：子 agent 工具面 deny-wins（agent-tools.policy.ts:26-42）——"默认开放"不适用子 agent 层。
7. **Flow bypass（Reconciliation 补）**：ACP 通道 sessions_send 归 control_plane 审批（approval-classifier.ts:26-32）——同一工具在不同通道权威要求不同。

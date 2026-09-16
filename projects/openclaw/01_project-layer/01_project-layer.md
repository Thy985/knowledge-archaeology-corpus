# 01 Project Layer — OpenClaw（项目地图）

> 本层为 L0/L1 项目事实底座，所有条目可追溯到仓库实际内容。

## 1.1 项目定位
- 本地优先 AI 助手 Gateway：`README.md` "OpenClaw is an open-source AI assistant that runs on your own computer and meets you in the channels you already use"
- 架构主张：`README.md` "The architecture case — trusted gateway, untrusted execution, deterministic policy"
- 治理：OpenClaw Foundation（501(c)(3)），无付费层/托管服务/token

## 1.2 架构（仓库实际布局）
```
openclaw.mjs  →  src/entry.ts（CLI/守护进程入口，Node runtime 恢复 launcher）
src/
├── gateway/         Gateway 核心（agent-turn 调度/approval/运行身份/agent-list）
├── agents/          内置 agent 运行时
│   ├── sessions/    session 持久化/compaction/execution/prompting（agent-session-*.ts）
│   ├── subagents/   子 agent：swarm/ + spawn/ + registry/ + completion/
│   │   └── swarm/   swarm-scheduler/swarm-config/swarm-collector（2026.9.2 默认开启）
│   ├── harness/     harness 注册/选择/生命周期 + tool-authority.runtime.ts
│   ├── embedded-agent-runner/  内置 attempt loop（run.ts）
│   └── sandbox/     sandbox 工具策略 + workspace-authority.ts
├── security/        audit-*（安全审计：cross-agent/plugins-trust/gateway-exposure/…）
├── sessions/        session 操作（列表/历史/搜索/发送）
├── config/          zod schema 全量配置 + schema.help.*（含 tools.sessions/agentToAgent/swarm）
├── memory/          memory-artifact-provenance / root-memory-files
├── state/           SQLite 状态（agent-provenance/admission/backup-run-records）
├── mcp/ skills/ plugins/ hooks/ channels/ llm/ model-catalog/ …
└── plugin-sdk/      session-visibility.ts（权限决策核心）
packages/            22 个共享包（agent-core/llm-core/net-policy/sdk/gateway-protocol/…）
apps/                8 个客户端
```

## 1.3 生命周期（一次 agent 运行的路径）
1. **配置解析**：`src/config/zod-schema.agent-runtime.ts` 校验 OpenClawConfig（agents.entries/tools.*）
2. **启动**：openclaw.mjs（Node 版本检查/恢复）→ src/entry.ts → gateway 启动
3. **turn 调度**：src/gateway/agent-turn/（消息 → agent 运行）
4. **工具权限准备**：`src/agents/harness/tool-authority.runtime.ts` `withPreparedEmbeddedRunToolAuthority`——policy 准备先于 authority 发布（"Execution-only: policy preparation must finish before authority reaches a publisher"）
5. **session 可见性检查**：src/plugin-sdk/session-visibility.ts（self/tree/agent/all + agentToAgent isAllowed）
6. **spawn 子 agent**：src/agents/spawn-pipeline.ts（initialize→dispatch→register 三阶段）+ accepted-session-spawn.ts（child 归属）
7. **swarm 并发**：src/agents/subagents/swarm/swarm-scheduler.ts（AsyncLocalStorage 激活身份 + 容量 lane）
8. **审计**：src/security/audit-extra.summary.ts（配置安全审计）

## 1.4 核心数据结构
| 数据 | 定义 | 位置 |
|---|---|---|
| OpenClawConfig | zod 全量配置 | src/config/zod-schema.agent-runtime.ts |
| SessionToolsVisibility | "self"\|"tree"\|"agent"\|"all" | src/plugin-sdk/session-visibility.ts |
| AgentToAgentPolicy | {enabled, matchesAllow, isAllowed} | 同上 |
| ResolvedSwarmConfig | enabled/maxConcurrent/maxChildrenPerGroup/maxTotalPerGroup/waitTimeout/defaultAgentId | src/agents/subagents/swarm/swarm-config.ts |
| SessionAccessResult | {allowed, error, status:"forbidden"} | src/plugin-sdk/session-visibility.ts |
| SecurityAuditFinding | {checkId, severity, title, detail, remediation} | src/security/audit-extra.summary.ts |
| AgentRuntimeSessionSpawnContext | {completionOwnerSessionKey, resolvedModel, inheritedToolPolicy} | src/gateway/agent-runtime-session-spawn-context.ts |

## 1.5 核心状态（SQLite）
- agent-provenance.ts / agent-database-admission.ts / backup-run-records.ts（src/state/）
- session-manager.ts（src/agents/sessions/）
- subagent-registry（src/agents/subagents/registry/）
- 规则（AGENTS.md）：OpenClaw state 用 SQLite（Kysely），不用新 JSON/JSONL sidecar；同步写事务

## 1.6 主要测试体系
- 15,643 *.test.ts（vitest）+ 864 *.e2e*.ts
- 表驱动安全审计测试：`src/security/audit-cross-agent-session-access.test.ts`——7 例"不 flag"（implicit/explicit agent、visibility agent/tree/self、agentToAgent disabled、allow list、sandbox all、session tools 移除）+ 4 例"报 1 条 info"（默认 entries、list roster、all+空 allow、invalid visibility→all）
- 架构契约：test/architecture-smells.test.ts、test/canonical-descendant.integration.test.ts
- 权限集成：test/channel-message-read-authority.integration.test.ts

## 1.7 主要配置（权限相关，2026.9.2 变更焦点）
| 配置 | 默认（2026.9.2 后） | 语义 | 证据 |
|---|---|---|---|
| tools.sessions.visibility | **"all"**（省略=all；2026.9.2 前省略=agent） | 会话工具可及范围 | zod-schema.agent-runtime.ts:770 + docs/releases/2026.9.2.md:3245 |
| tools.agentToAgent.enabled | **true**（2026.9.2 前省略=false） | 跨 agent 会话工具访问 | schema.help.runtime.ts:119 |
| tools.agentToAgent.allow | 省略/空=允许全部 agent 对 | 白名单 glob | session-visibility.ts:244-276 |
| tools.swarm.enabled | **true**（2026.9.2 默认开启） | swarm 并发编排 | swarm-config.ts DEFAULT_SWARM_CONFIG |
| tools.swarm.maxConcurrent | 8 | 并发上限 | 同上 |
| tools.swarm.maxChildrenPerGroup | 50 | 每组子 agent 上限 | 同上 |
| tools.swarm.maxTotalPerGroup | 200 | 每组总上限 | 同上 |
| tools.message.crossContext.allowAcrossProviders | true | 跨 provider 消息 | tool-permissions.md |
| tools.exec.mode | （session 模式决定） | read-only/guarded/workspace/full | permission-modes.md |
| agents.defaults.sandbox.sessionToolsVisibility | "spawned" | sandbox 下 session 可见性 clamp | session-visibility.ts:175 |

## 1.8 权限 / policy / governance 机制
1. **四层叠加**：tool allow/deny → session visibility（self/tree/agent/all）→ agentToAgent（enabled+allow glob，requester+target 双匹配）→ sandbox clamp（spawned→tree）
2. **安全审计暴露层**：`collectCrossAgentSessionAccessFindings`——多 agent + visibility=all + agentToAgent 无限制 + 有 session 工具 → 报 info（有 trust-boundary signal 则 warn）
3. **信任模型**：SECURITY.md "session ownership, visibility, and presence are usability features, not security boundaries"——隔离 = 独立 Gateway
4. **审批流**：approval-channel-custody（channel 审批 custody）、approval-session-audience、workspace 模式 LLM reviewer（allow/deny/ask human）、3 连拒升级 human（permission-modes.md）
5. **AGENTS.md 治理契约**：one owner per responsibility；特权动作需当前 owner 持有的 authority，await 后/副作用前重新验证

## 1.9 主要外部依赖
- Node.js >=24.16 <25 或 >=26.1（Node 26 推荐）
- pnpm workspace；SQLite + Kysely
- `@earendil-works/pi-tui`（终端组件，docs/agent-runtime-architecture.md 唯一第三方 UI 依赖）
- harness 插件（Claude/Codex/本地模型）；MCP servers/plugins（plugin-sdk 契约）
- 部署：docker-compose/Dockerfile/fly.toml/render.yaml

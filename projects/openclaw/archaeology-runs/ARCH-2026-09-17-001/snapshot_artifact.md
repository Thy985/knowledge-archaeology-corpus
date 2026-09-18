# Repository Snapshot — ARCH-2026-09-17-001

## 1. 快照事实（可追溯）

| 字段 | 值 | 来源 |
|---|---|---|
| repository | https://github.com/openclaw/openclaw.git | job_manifest |
| commit SHA | `ca38979d2128d21616f815136f48d7466b803b4d` | git log -1 |
| commit message | "test(doctor): avoid repeated bootstrap in backup matching cases (#150197)" | git log -1 |
| commit timestamp | 2026-09-16 11:02:37 -0700 | git log -1 --format=%ci |
| branch | main（origin/main，浅克隆 depth 1） | git branch -a |
| repository version | 2026.9.4（package.json `version`） | package.json |
| 关注版本 | 2026.9.2（swarm 默认开启 + 跨 agent 会话访问，CHANGELOG/2026.9.2.md） | CHANGELOG |
| 仓库大小 | 821MB（工作树） | du -sh |
| 文件总数 | 43,727（clone 报告） | git clone 输出 |
| 主要语言 | TypeScript（35,463 .ts）/ Python 46 / Go 30 | find 统计 |
| License | MIT（README badge + LICENSE） | README.md |
| 项目描述 | "Multi-channel AI gateway with extensible messaging integrations" | package.json |
| 测试文件 | 15,643 *.test.ts + 864 *.e2e*.ts | find 统计 |
| 运行时 | Node.js >=24.16 <25 或 >=26.1（Node 26 推荐） | openclaw.mjs |

## 2. 项目基础地图（Repository Map）

```
openclaw/
├── openclaw.mjs          # CLI 入口（Node runtime 恢复/重生成 launcher）
├── package.json          # v2026.9.4，pnpm workspace
├── pnpm-workspace.yaml   # monorepo 配置
├── AGENTS.md / CLAUDE.md # 贡献者/Agent 工作契约（one owner per responsibility）
├── VISION.md             # 产品范围声明
├── SECURITY.md           # 信任模型：session ownership ≠ security boundary
├── CHANGELOG/            # 每版本 changelog（2026.9.2/9.4 等）
├── docs/                 # 完整文档（gateway/permission-modes/security/cli/…）
├── packages/             # 22 个包（acp-core/agent-core/llm-core/net-policy/sdk/…）
├── apps/                 # 8 个客户端（android/ios/macos/linux/mobile/shared/…）
├── src/                  # 核心源码（gateway/agents/security/sessions/memory/…）
│   ├── gateway/          # Gateway：agent-turn/approval/agent-list/…
│   ├── agents/           # 运行器：sessions/subagents/swarm/spawn/embedded-agent-runner
│   │   └── subagents/
│   │       ├── swarm/    # swarm-scheduler/swarm-config/swarm-collector（2026.9.2 默认开启）
│   │       └── spawn/    # spawn 管线/authority
│   ├── security/         # audit-*（含 cross-agent-session-access audit）
│   ├── sessions/         # 会话可见性（self/tree/agent/all）
│   ├── config/           # zod schema（tools.sessions.visibility / agentToAgent）
│   ├── memory/           # memory-artifact-provenance/root-memory-files
│   ├── state/            # SQLite state（agent-provenance/agent-database-admission）
│   ├── mcp/  skills/  plugins/  hooks/  channels/  …
│   └── entry.ts          # 主入口
├── test/                 # 集成/契约测试（architecture-smells/canonical-descendant/…）
├── qa/  scripts/  ui/  config/  extensions/  skills/  custodian-skills/
└── docker-compose.yml / Dockerfile / fly.toml / render.yaml   # 部署
```

### 2.1 主要语言 / 运行入口
- **语言**：TypeScript（99%+，35,463 文件），pnpm + tsdown 构建
- **入口**：`openclaw.mjs`（launcher，含 Node 运行时恢复）→ `src/entry.ts`
- **运行模型**：单 Gateway 守护进程 + 多 agent entries + 多 channel 接入（Discord/iMessage/Slack/Teams/Telegram/WhatsApp/20+）+ 原生客户端

### 2.2 核心模块
| 模块 | 职责 | 关键文件 |
|---|---|---|
| Gateway | 守护进程核心：agent-turn 调度/approval/运行身份 | src/gateway/agent-turn/, approval-* |
| Agent 运行器 | session 生命周期/compaction/execution/prompting | src/agents/sessions/agent-session-*.ts |
| Subagent/Swarm | 并发子 agent 编排（2026.9.2 默认开启） | src/agents/subagents/swarm/swarm-scheduler.ts |
| Spawn 管线 | 子 session 创建/授权链 | src/agents/spawn-pipeline.ts, accepted-session-spawn.ts |
| Security audit | 配置安全审计（跨 agent 会话访问等） | src/security/audit-extra.summary.ts |
| Config | zod schema 全量配置校验 | src/config/zod-schema.agent-runtime.ts |
| State | SQLite 状态（Kysely，同步事务） | src/state/, src/agents/sessions/*.sqlite* |

### 2.3 核心数据结构 / 状态
- **Config**：OpenClawConfig（zod）——agents.entries / tools.sessions.visibility（self|tree|agent|all）/ tools.agentToAgent（enabled/allow）/ tools.swarm（enabled/defaultAgentId）
- **Session**：sessionId/sessionKey/sessionFile/agentId/runId（tool-authority.runtime.ts）
- **Swarm**：SwarmGroupLane（groupId/limit/active/queue）+ QueuedSwarmRun（runId/owner/capacity）
- **Authority**：admittedRunContext → operationalRunInstance → toolAuthorityFingerprint

### 2.4 主要测试体系
- **15,643** *.test.ts（vitest）+ **864** e2e *.e2e*.ts
- 契约测试：canonical-descendant.integration.test.ts、channel-message-read-authority.integration.test.ts
- 架构气味检查：architecture-smells.test.ts
- 安全审计测试：audit-cross-agent-session-access.test.ts（表驱动：implicit/explicit/visibility/allow list/sandbox）

### 2.5 主要配置
- `tools.sessions.visibility`：默认 **"all"**（2026.9.2 变更：session tools 默认全会话可见）
- `tools.agentToAgent.enabled`：默认 **true**；`allow` 缺省=允许任意 agent 对
- `tools.swarm.enabled`：默认 **true**（2026.9.2：swarm 默认开启）
- `tools.message.crossContext.allowAcrossProviders`：默认 true
- 权限模式：read-only/guarded/workspace/full（session 文件系统边界 + exec 升级审阅人）
- 危险工具 deny 清单建议：gateway/cron/sessions_spawn/sessions_send

### 2.6 权限 / policy / governance 机制
- **四层权限面**：tool policy（allow/deny）→ session visibility（self/tree/agent/all）→ agentToAgent（跨 agent 访问）→ sandbox/elevated
- **security audit**：`security.trust_model.cross_agent_session_access_default` 检查（info 级，非 fail）
- **信任模型（SECURITY.md 明示）**：session ownership/visibility/presence 是**可用性特性，不是安全边界**；"Anyone who can operate an agent can make it do anything that agent can do"——真实隔离需独立 gateway/trust boundary
- **审批流**：approval-channel-custody / approval-session-audience / workspace 模式 LLM reviewer（allow/deny/ask human），3 连拒升级 human

### 2.7 主要外部依赖
- **Node.js** >=24.16（Node 26 推荐）——唯一硬运行时
- pnpm workspace；SQLite（Kysely ORM）
- 模型 harness 插件：Claude/Codex/本地模型（可插拔）
- MCP servers / plugins（openclaw/plugin-sdk 契约）
- 部署：docker-compose/Dockerfile/fly.toml/render.yaml

## 3. 考古焦点声明（对应 Job Manifest）
本 Snapshot 为 2026.9.2 权限模型变更考古提供事实底座：
1. **swarm 默认开启**（tools.swarm.enabled=true）的编排与容量治理
2. **跨 agent 会话访问默认扩大**（visibility=all + agentToAgent.enabled=true）的权限语义与 audit 暴露
3. **信任模型声明**（SECURITY.md）与 09-14 RSAC"coding agent 100% 可注入"的张力

所有事实可追溯到上述仓库实际内容（文件/行/schema/测试）。

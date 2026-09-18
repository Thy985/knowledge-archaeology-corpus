# 01 Project Layer — 项目地图（精简导航版）

```
letta-code (v0.32.12, TypeScript+Bun, Apache-2.0)
│
├── 入口
│   ├── src/index.ts            # bun CLI/TUI 主入口
│   └── src/standalone-entry.ts # 先注册 pi-ai Bun OAuth，再 import index
│
├── 交互面
│   ├── src/cli/                # TUI（react/ink）、子命令（memory/trajectories/…）
│   ├── src/web/                # web 视图（memory-viewer 等）
│   ├── src/websocket/          # WS listener
│   └── src/channels/           # slack/telegram/discord/cloud-send
│
├── 控制面（Backend）
│   ├── src/backend/api/        # 云后端（remoteMemfs=true, api.letta.com）
│   ├── src/backend/local/      # 本地后端（lc-local-backend/memfs）
│   ├── src/backend/backend-mode.ts   # api|local 单源真值
│   └── src/backend/local/system-prompt-compilation.ts  # {CORE_MEMORY} 注入
│
├── 记忆子系统（本考古核心）
│   ├── src/agent/memory-filesystem.ts   # MEMORY_FS_ROOT=".letta"、目录、树渲染
│   ├── src/agent/memory-git.ts          # git-backed 记忆（clone/pull/commit/push/retry）
│   ├── src/agent/memory-git-hooks.ts    # pre-commit frontmatter 校验 + post-commit 镜像
│   ├── src/agent/memory-format.ts       # memfs-v1|v2 检测、core/projected 判定
│   ├── src/agent/memory-markdown.ts     # frontmatter 解析/渲染、defaultMemoryName
│   ├── src/agent/memory-runtime.ts      # isActiveMemfsEnabled
│   ├── src/agent/memory-constraints.ts  # 树约束 + 自包含 validator 脚本
│   ├── src/agent/memory-constraints-audit.ts # 无副作用 HEAD 审计
│   ├── src/agent/memory-worktree.ts     # harness worktree（固定 git 身份）
│   ├── src/agent/memory-auth.ts         # git credential
│   ├── src/agent/memory-scanner.ts      # 树扫描（TUI/web 视图）
│   ├── src/memory-confinement.ts        # fail-closed 记忆子代理包装
│   ├── src/memory-constraints.ts        # 顶层约束 re-export
│   ├── src/memory-frontmatter.ts        # frontmatter 校验（v2 白名单/legacy read_only）
│   └── src/agent/subagents/             # reflection/memory/init/history-analyzer 子代理
│       ├── manager.ts / launcher.ts / sandbox.ts / context-budget.ts / model.ts
│
├── 治理面
│   ├── src/permissions/       # mode/loader/matcher/checker/analyzer/session/cli + sandbox-policy
│   │                          # + memory-confinement-launcher + memory-paths + cross-agent-guard
│   │                          # + sandbox-gate + read-only-* + format-denial + shell-*
│   ├── src/sandbox/           # availability / policy / wrap / seatbelt(macOS) / bwrap(Linux)
│   ├── src/mods/              # capabilities/context/conversation-*/deprecated-api/disabled-*
│   └── src/skills/            # 19 内置 skills（initializing-memory/managing-shared-memory/…）
│
├── 工具面
│   ├── src/tools/impl/        # memory.ts（6 命令）+ memory-apply-patch.ts（add）+ 其余 ~50 工具
│   ├── src/tools/tool-definitions.ts   # 三套等价工具集注册（Anthropic/Codex/Gemini）
│   ├── src/tools/descriptions/         # MemoryV2.md / MemoryApplyPatchV2.md 等
│   └── src/tools/manager.ts   # 工具执行、并行安全判断
│
├── 生命周期/调度
│   ├── src/agent/approval-*.ts / check-approval.ts
│   ├── src/headless-*.ts（33 个 headless 测试/实现）
│   ├── src/queue/ src/reminders/ src/cron/ src/schedules.ts
│   └── src/updater/ src/startup-*.ts
│
├── 根治理文件
│   ├── AGENTS.md / CLAUDE.md   # agent 导航规则（@/、kebab-case、named exports、check 必跑）
│   ├── AI_POLICY.md            # AI 贡献披露政策（仿 Ghostty）
│   ├── package.json            # 18 deps；CI 聚合 scripts/check.js
│   └── scripts/check*.js       # 11 道 CI 门
└── 外部依赖（关键）
    ├── @earendil-works/pi-ai     # provider 流（Anthropic/Azure/Bedrock/Gemini）
    ├── @letta-ai/letta-client    # agent/message/tool 资源
    ├── @letta-ai/trajectory      # transcript 规范化/导出/评审
    └── @modelcontextprotocol/sdk # MCP
```

## 核心数据结构速查
| 结构 | 位置 | 说明 |
|---|---|---|
| `MemoryConstraintsConfig {version:1, maxDepth?, maxFileCharacters?, maxCoreMemoryCharacters?, fileCharacterLimits?}` | memory-constraints.ts | 记忆树约束 |
| `FsSandboxPolicy {baseWritableRoots, deniedRoots, readonlyRoots, writableRoots, restrictWrites}` | sandbox/policy.ts | 沙箱策略（顺序语义） |
| `LocalMemoryFormat = "memfs-v1"\|"memfs-v2"` | memory-format.ts | 记忆格式 |
| `MemoryMarkdownFrontmatter {name?, description?, read_only?}` | memory-markdown.ts | 记忆文件元数据 |
| `MemoryWriteSyncMode = "remote"\|"local"` | memory-git.ts | 写同步模式 |
| `SubagentLaunchProfile = "default"\|"memory-subagent"` | subagents/index.ts | 子代理配置 |
| `PermissionMode = unrestricted\|standard\|acceptEdits\|strict` | permissions/mode.ts | 权限模式 |
| `ReflectionTrigger = off\|step-count\|compaction-event` | reflection-settings.ts | 反射触发 |
| `BackendCapabilities {remoteMemfs, localMemfs, …}` | backend/backend.ts | 后端能力面 |

## 核心状态
| 状态 | 载体 | 说明 |
|---|---|---|
| 记忆内容 | `~/.letta/agents/<id>/memory/*.md` | git 仓库内的 Markdown 树 |
| 记忆 git 状态 | 该目录 git repo | clone/pull/commit/push 生命周期 |
| 记忆格式版本 | 根 MEMORY.md 存在性 | detectMemoryFormat |
| 后端模式 | backend-mode.ts 全局 | api/local override |
| 沙箱可用性 | sandbox/availability 缓存 | detect 一次、进程内缓存 |
| settings | settings-manager 内存 + 持久化 | memfs/reflection 等 |
| 权限模式 | settings/permissions | 四模式 |
| 当前 agent/会话 | agent/context.ts | AGENT_ID/CONVERSATION_ID |

## 主要配置入口
- 环境变量：LETTA_MEMFS_BASE_URL / LETTA_LOCAL_BACKEND_DIR / LETTA_TRANSCRIPT_ROOT / LETTA_FS_SANDBOX / LETTA_SANDBOX / AGENT_ID / MEMORY_DIR
- 文件：package.json（scripts）/ biome.json / tsconfig*.json / bunfig.toml / hooks/
- 记忆级：.memfs.config.json（约束）/ MEMORY.md（v2 索引）/ frontmatter（元数据）
- 治理：AGENTS.md / AI_POLICY.md / mods 能力面

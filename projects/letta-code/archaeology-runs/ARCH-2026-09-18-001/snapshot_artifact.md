# Repository Snapshot — letta-code

> 事实全部可回溯 `$J/repo` 实际内容（HEAD `3df2ebd6e5826b482f0124e2b02f0e4e0f1af567`）。

## 快照元数据
| 字段 | 值 | 证据 |
|---|---|---|
| repository | https://github.com/letta-ai/letta-code.git | README.md: "current source code lives in letta-ai/letta-code"（letta 主仓 main 仅文档，archive 分支=旧 V1 服务端） |
| commit SHA | `3df2ebd6e5826b482f0124e2b02f0e4e0f1af567` | `git log -1` |
| commit message | "docs(skills): use applicability metadata in letta-guide (#4515)" | 同上 |
| branch | main（default） | `git branch` |
| version | 0.32.12（name `@letta-ai/letta-code`, type module） | package.json |
| analysis timestamp | 2026-09-18 02:xx (+08) | 本 run |
| repository size | 93MB；2308 files（1871 `.ts` / 146 `.md`） | `du -sh` + `find` |
| license | Apache-2.0 | LICENSE |
| skill version | knowledge-archaeology v3.2 | SKILL.md frontmatter |

## 项目基础地图
```
letta-code/
├── src/
│   ├── index.ts               # 主入口（bun TUI/CLI，引用 @letta-ai/letta-client/resources/agents/*）
│   ├── standalone-entry.ts    # standalone 入口（先 registerBunOAuthFlows 再 import ./index）
│   ├── agent/                 # 记忆与代理核心（memory-filesystem/git/markdown/format/runtime/worktree/scanner/auth）
│   │   └── subagents/         # 子代理（default / memory-subagent 两类 launch profile；reflection/memory/history-analyzer/init）
│   ├── backend/               # api（云）/ local（本地）双后端；local 含 system-prompt-compilation（{CORE_MEMORY} 注入）
│   ├── permissions/           # 权限面：sandbox-policy / memory-confinement-launcher / memory-paths / cross-agent-guard / sandbox-gate / read-only-* / mode
│   ├── sandbox/               # 内核沙箱：bwrap.ts（Linux）/ seatbelt.ts（macOS）/ wrap.ts / policy.ts / availability.ts
│   ├── memory-confinement.ts  # 记忆子代理 fail-closed 包装（无沙箱即抛错）
│   ├── memory-constraints.ts  # MemFS 树约束（maxDepth/maxFileCharacters/maxCoreMemoryCharacters/fileCharacterLimits + 自包含 validator 脚本）
│   ├── memory-frontmatter.ts  # frontmatter 校验（v2 仅 name/description；legacy read_only 保护）
│   ├── mods/                  # Mod 系统（capabilities/context/conversation-*/deprecated-api/disabled-*）
│   ├── skills/                # 内置 skills（initializing-memory/managing-shared-memory/migrating-memory/syncing-memory-filesystem/letta-guide 等）
│   ├── tools/impl/            # 工具实现：memory.ts（str_replace/insert/delete/rename/update_description/create）+ memory-apply-patch.ts（add）
│   ├── cli/  web/  websocket/  channels/  cron/  queue/  reminders/  telemetry/  providers/
│   ├── headless-*.ts/.test.ts # headless 生命周期/权限/记忆/工具事件测试（816 test + 8 integration）
│   ├── settings-manager.ts    # 设置持久化（isMemfsEnabled 等）
│   └── runtime-context.ts     # 运行时上下文
├── AGENTS.md / CLAUDE.md      # agent 导航规则（@/ 别名禁 ../、kebab-case、named exports、bun run check 必跑）
├── AI_POLICY.md               # AI 披露政策（仿 Ghostty；非合规 Issue/PR 自动关闭）
├── package.json               # 18 生产依赖；scripts: lint/typecheck/check:cycles/boundaries/filename-casing/file-size/module-ownership/test-mock-isolation/test-coverage/skill-frontmatter/bundled-skill-scripts/check
├── biome.json / tsconfig*.json / bunfig.toml / build.js / scripts/（check*.js, dev.cjs）
├── docs/（examples/plans/nix.md） docker/ nix/ vendor/ hooks/
└── test-utils/（update-chain-smoke 等）
```

## 核心事实锚点（供阶段 4 引用，均可回溯）

### 记忆文件系统
- `MEMORY_FS_ROOT = ".letta"`；路径 `~/.letta/agents/<agentId>/memory/`；另有 `LETTA_LOCAL_BACKEND_DIR` / `lc-local-backend/memfs` 本地变体（memory-filesystem.ts）
- 目录渲染限制：`MEMORY_TREE_MAX_LINES / CHARS / CHILDREN_PER_DIR`（来自 `utils/directory-limits` 默认值）

### git-backed 记忆
- 记忆仓库：`$LETTA_MEMFS_BASE_URL/v1/git/$AGENT_ID/state.git`（fallback `api.letta.com`）；本地后端走 memfs-git-proxy；localhost 代理 URL 不持久化进 repo git config（memory-git.ts）
- 生命周期：首次 run clone → startup pull → 记忆写 commit → post-turn push（clean pending commits）；retry 正则覆盖 HTTP 503/520-524 + 网络类错误
- pre-commit hook：校验 frontmatter（description 必填非空；read_only 是 protected 字段，agent 不得增删改；仅允许 agent 编辑 description；legacy limit 容忍）+ 可选内存树约束校验（.memfs.config.json / MEMORY.md root-marker 布局）（memory-git-hooks.ts）
- post-commit hook：镜像提交到用户配置的 memory-repository remote
- memory-worktree：harness 创建的 worktree 用 `GIT_AUTHOR_NAME="Letta Code"` 等固定身份，30s timeout

### MemFS v1 / v2
- `LocalMemoryFormat = "memfs-v1" | "memfs-v2"`；detect 依据 memoryDir 是否存在根 `MEMORY.md`（且非 localMemfs capability 时）（memory-format.ts）
- v2：core memory = 顶层 `.md`（不含 `/`）；projected memory = 需每层目录有 `MEMORY.md` 索引（isProjectedMemoryPath）；`skills/` 前缀非 projected；根 MEMORY.md 不得有 frontmatter（memory-markdown.ts / memory-frontmatter.ts）
- v2 frontmatter 仅允许 `name` / `description`；legacy 允许 `description` / `read_only` / `limit`；`read_only: true` 仅 legacy 生效（memory-frontmatter.ts）

### 记忆运行时
- `isActiveMemfsEnabled(agentId) = backend.capabilities.localMemfs || settingsManager.isMemfsEnabled(agentId)`（memory-runtime.ts）

### 记忆约束
- `MemoryConstraintsConfig {version:1, maxDepth?, maxFileCharacters?, maxCoreMemoryCharacters?, fileCharacterLimits?}`；`validateMemoryTreeConstraints` 要求 v2 根 MEMORY.md 存在，扫描 `.md` 文件（memory-constraints.ts）；`memory-constraints-audit.ts` 用临时 GIT_INDEX_FILE 对 HEAD 无副作用审计

### 记忆工具
- `memory` 工具命令：str_replace / insert / delete / rename / update_description / create；`reason` 必填；v2 create 需目录已被 MEMORY.md 索引；写入后走 git commit（同步模式 `remote` / `local`）（tools/impl/memory.ts）
- `memory_apply_patch`：`add`（targetLabel/targetRelPath）+ 同样 git 写路径（tools/impl/memory-apply-patch.ts）

### 记忆子代理与隔离
- SubagentLaunchProfile = "default" | "memory-subagent"；memory-subagent 覆盖 reflection / memory / init / history-analyzer（subagents/index.ts、manager.ts）
- memory-subagent 默认包内核沙箱（`LETTA_FS_SANDBOX=0` 退出；与 cross-agent shell 沙箱 opt-in 不同，记忆子代理非交互无 approve/deny 可回退，故默认开）（subagents/sandbox.ts）
- FsSandboxPolicy：`baseWritableRoots`（宽 harness 根，先于 denied 放行）→ `deniedRoots`（跨 agent 树 `~/.letta/agents` 读写均拒）→ `writableRoots`（自我记忆 carve，覆盖 denied）→ `readonlyRoots`；`restrictWrites` 控制全局写默认（sandbox/policy.ts）
- memory-confinement：可广读 host、写 harness 状态与自身记忆、不可读写其他 agent 记忆；无内核沙箱即 throw（fail-closed）（memory-confinement.ts）
- reflection 子代理 context 预算：~16K 估算 tokens（4 chars/token 保守值），parent memory snapshot 40K chars 上限（subagents/context-budget.ts）

### 权限模式
- PermissionMode = standard / acceptEdits / unrestricted / strict；默认 unrestricted；legacy 映射（"default"→standard, "bypassPermissions"/"fullAccess"→unrestricted）（permissions/mode.ts）

### 外部依赖（package.json dependencies，18 项）
- `@earendil-works/pi-ai`（provider 流式 API：anthropic/azure/bedrock/google lazy streams）、`@letta-ai/letta-client`、`@letta-ai/trajectory`（transcript 规范化/导出/评审）、`@modelcontextprotocol/sdk`、`@pierre/diffs`、`node-pty`、`react`、`sharp`、`shiki`、`ws`、`glob`、`cron-parser`、`cross-spawn`、`open`、`ink-link`、`strip-ansi`、`@scarf/scarf`、`@janhapke/sharp-electron`

### 测试体系
- 816 `*.test.ts` + 8 `*.integration.test.ts`
- 记忆专项：memory-filesystem(.integration/.sync.integration)、memory-git(.auth/.config-lock/.local-scope/.postcommit/.precommit/.retry/.signing/.v2-precommit/.windows-credentials)、memory-format、memory-constraints-audit、memory-prompt.integration、client-skills-shared-memory、memory-confinement、memory-constraints、memory-frontmatter、headless-memfs-policy、memory-worktree(.http)、memory-git-hooks
- 治理/沙箱：cross-agent-guard、read-only-letta/read-only-shell(.security)、sandbox-gate、sandbox-policy、workspace-sandbox、permissions-*(loader/matcher/mode/session/analyzer/checker/cli/format-denial/shell-analysis/shell-command-normalization)
- 生命周期：headless-*(approval-recovery/backend-lifecycle/bidirectional-reflection/bootstrap-pending-approval/client-skills/cloud-send/cloud-startup/enqueue-wait/environment-response/ephemeral-startup/ephemeral-toolset/interrupt-latch/interrupt-recovery/listener-launch/memfs-policy/message-sender/mod-adapter/mod-lifecycle/permission/queueing/queue-lifecycle/reflection-settings/reminder/subagent-sender/subagent-stdout-loss/super-run-wait/telemetry-exit/telemetry-input/tool-events)
- CI 门（scripts/check.js 聚合）：typecheck、check:cycles（madge）、check:boundaries（check-layer-boundaries.js）、check:exported-functions、check:filename-casing、check:file-size、check:module-ownership、check:test-mock-isolation、check:test-coverage、check:skill-frontmatter、check:bundled-skill-scripts

### 关键配置/策略/治理
- 运行：Bun（dev 走 TS 源码）；发布产物 Node-targeted `letta.js`（Node ≥22.19）；双运行时一致性由 AGENTS.md 强制（AGENTS.md Runtime Validation 段）
- 自动化通知用 user-role `<system-reminder>`（禁 role:system 注入；Anthropic 拒绝场景）——AGENTS.md
- AI_POLICY.md：AI 参与贡献必须披露、人工负责、非合规自动关闭
- mods 能力面：MOD_CAPABILITY_IDS = tools/commands/providers/permissions/events.{lifecycle,turns,tools,compact,llm}/ui.panels；DEFAULT_MOD_CAPABILITIES 全开（mods/capabilities.ts）

## 快照边界
- 浅克隆 depth=1，仅含 main HEAD；历史演进/ADR 证据需经 GitHub API 或 fetch 补充（阶段 4 按需）
- 93MB 大仓：避免全量 grep/递归扫描，按模块路径定向阅读

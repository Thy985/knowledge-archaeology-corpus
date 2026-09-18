# 01 Project Layer — letta-code 项目地图

> 全部事实可回溯 `repo/`（HEAD `3df2ebd6`）。L0/L1 事实编号 F-01…F-121。

## 1.1 项目定位（F-01~F-05）
- F-01: `@letta-ai/letta-code`，version 0.32.12，type module（package.json）
- F-02: README 明示 letta 主仓 main 仅文档，"current source code lives in letta-ai/letta-code"
- F-03: 语言 TypeScript（1871 .ts）+ Bun 运行时；发布物为 Node-targeted `letta.js`（Node ≥22.19）
- F-04: License Apache-2.0；仓库 93MB / 2308 文件 / 816 test + 8 integration
- F-05: AGENTS.md / CLAUDE.md 双 agent 指南（内容一致），AI_POLICY.md 贡献治理

## 1.2 架构总览（F-06~F-20）
- F-06: 三层：CLI/TUI/Web/Channel（交互面）→ Backend（api/local 双模式）→ 工具/记忆/子代理（执行面）
- F-07: 入口 `src/index.ts`（bun）；`src/standalone-entry.ts` 先注册 pi-ai Bun OAuth 再 import index
- F-08: `BackendMode = "api" | "local"`（backend-mode.ts）；override 优先于 `isLocalBackendEnvEnabled()` 环境标志
- F-09: BackendCapabilities 含 `remoteMemfs` / `localMemfs`（backend.ts:160-166）；api 后端 remoteMemfs=true
- F-10: 系统提示词编译在 `backend/local/system-prompt-compilation.ts`：`{CORE_MEMORY}` 变量注入 + `$MEMORY_DIR` 占位符
- F-11: 记忆投影格式：`<self>`（system/persona.md）、`<memory>`（system 树 + external projection）、`<memory_metadata>`（AGENT_ID/CONVERSATION_ID/recompiled_at/recall 消息数）
- F-12: 记忆工具注册：`tools/tool-definitions.ts` 中 `memory` + `memory_apply_patch`（含 V2 schema/描述）
- F-13: Mods 系统：MOD_CAPABILITY_IDS = tools/commands/providers/permissions/events.{lifecycle,turns,tools,compact,llm}/ui.panels；默认全开
- F-14: Skills 四源优先级：project（.agents/skills）> agent（~/.letta/agents/{id}/memory/skills）> global（~/.letta/skills）> bundled（skills.ts）
- F-15: 内置 skills 19 个：initializing-memory / managing-shared-memory / migrating-memory / syncing-memory-filesystem / letta-guide / creating-mods 等
- F-16: 子代理 4 类 memory-subagent：reflection / memory / init / history-analyzer（subagents/index.ts:89-92）
- F-17: 权限四模式：unrestricted（默认）/ standard / acceptEdits / strict（permissions/mode.ts）
- F-18: 沙箱双后端：seatbelt（macOS）/ bwrap（Linux），detectSandboxBackend 探测并缓存
- F-19: 外部依赖 18 项：pi-ai（provider 流）、letta-client、trajectory、MCP SDK、react（TUI）、node-pty、sharp 等
- F-20: 工具面：Anthropic/Codex/Gemini 三套等价工具集 + 并行安全白名单（PARALLEL_SAFE_TOOLS）

## 1.3 记忆子系统（F-21~F-55）
- F-21: `MEMORY_FS_ROOT = ".letta"`；`~/.letta/agents/<agentId>/memory/`（memory-filesystem.ts）
- F-22: 本地后端变体：`lc-local-backend/memfs`（getCrossBackendAgentsTreeRoots 双树墙）
- F-23: 记忆 git 仓库 URL：`$LETTA_MEMFS_BASE_URL/v1/git/$AGENT_ID/state.git`，fallback `api.letta.com`
- F-24: git 生命周期：first-run clone → startup pull → 写 commit → post-turn push（memory-git.ts 头注释）
- F-25: retry 正则：HTTP 503/520-524、网络类（hung up/reset/timeout/SIGTERM/ETIMEDOUT）
- F-26: 非 fast-forward push 错误处理：NON_FAST_FORWARD_PUSH_ERROR_RE + 提示 fetch first
- F-27: pre-commit hook：frontmatter 校验（description 必填非空；read_only protected；仅允许编辑 description；legacy limit 容忍）
- F-28: post-commit hook：镜像提交到用户配置的 memory-repository remote
- F-29: `MemoryWriteSyncMode = "remote" | "local"`；由 backend.capabilities.localMemfs 决定（tools/impl/memory.ts:51-56）
- F-30: `LocalMemoryFormat = "memfs-v1" | "memfs-v2"`；detect = 存在根 MEMORY.md 且非 localMemfs capability（memory-format.ts）
- F-31: v2 核心记忆 = 顶层 .md（isCoreMemoryPath：v2 不含 "/"，v1 以 system/ 开头）
- F-32: v2 投影记忆 = 每层目录需 MEMORY.md 索引（isProjectedMemoryPath）；skills/ 前缀排除
- F-33: v2 根 MEMORY.md 不得有 frontmatter（memory-markdown.ts / memory-frontmatter.ts）
- F-34: v2 frontmatter 仅 name/description；legacy 允许 description/read_only/limit
- F-35: `read_only: true` 保护**仅 legacy 生效**（memory-frontmatter.ts:48-60 legacyValue 标量保护查找）
- F-36: MemoryConstraintsConfig v1：maxDepth/maxFileCharacters/maxCoreMemoryCharacters/fileCharacterLimits
- F-37: validateMemoryTreeConstraints：v2 要求根 MEMORY.md；扫描 .md 文件；shared-memory/legacy-only/root-marker 三布局
- F-38: memory-constraints-audit：临时 GIT_INDEX_FILE + git read-tree HEAD 无副作用审计提交树
- F-39: 记忆工具命令：str_replace / insert / delete / rename / update_description / create；reason 必填
- F-40: memory_apply_patch：`add`（targetLabel/targetRelPath）
- F-41: 工具写路径：resolveScopedMemoryDir → assertMemfsV2MemoryPathIndexed → commitMemoryWrite
- F-42: v2 create 需目录已被 MEMORY.md 索引；v2 index 文件 description 自动为空
- F-43: 绝对路径仅限 $MEMORY_DIR 内（MemoryV2.md 描述）
- F-44: reflection 子代理 context 预算：~16K tokens（4 chars/token）；parent memory snapshot 40K chars 上限
- F-45: reflection 设置：trigger = off / step-count / compaction-event；merge = auto / explicit（reflection-settings.ts）
- F-46: memory-subagent 默认内核沙箱；`LETTA_FS_SANDBOX=0` 退出（subagents/sandbox.ts）
- F-47: 沙箱策略顺序语义：global write-deny → baseWritableRoots → deniedRoots → writableRoots → readonlyRoots（sandbox/policy.ts）
- F-48: 记忆子代理可广读 host、写 harness 状态与自身记忆、不可读写其他 agent 记忆（memory-confinement.ts 注释）
- F-49: 无沙箱时 createMemoryConfinementLauncher 抛错（fail-closed），不静默弱化
- F-50: memory-scanner：递归扫描渲染树；隐藏文件过滤；system 目录优先
- F-51: settings：memfs?: boolean；memoryReminderInterval DEPRECATED → reflection* 字段
- F-52: memory-worktree：harness 创建 worktree 用固定身份 "Letta Code" <noreply@letta.com>，30s timeout
- F-53: memory-git-signing：GIT_DISABLE_COMMIT_SIGNING_ARGS
- F-54: memory-auth.ts：git credential 用 getAuthToken；normalizeCredentialBaseUrl 规范化 URL origin
- F-55: Windows credentials：memory-git-windows-credentials.ts 写 credential helper

## 1.4 生命周期（F-56~F-70）
- F-56: headless 生命周期测试族：headless-*.test.ts ×33（approval-recovery/backend-lifecycle/bootstrap-pending-approval/cloud-startup/enqueue-wait/…）
- F-57: 双运行时（Bun dev / Node bundle）行为一致性由 AGENTS.md 强制 + 测试
- F-58: 自动化通知用 user-role `<system-reminder>`（禁 role:system 注入；Anthropic 拒绝场景）
- F-59: 更新链：startup-auto-update + updater + test:update-chain:manual/startup
- F-60: 审批：approval-execution（并行安全工具集 + 串行执行 Bash 类）；approval-recovery；check-approval
- F-61: interruption：INTERRUPTED_BY_USER；interrupt-latch/recovery headless 测试
- F-62: queueing：headless-queueing/queue-lifecycle/super-run-wait
- F-63: MCP：mcp-client/mcp-oauth/mcp-runtime/mcp-settings（OAuth 流）
- F-64: channels：slack/telegram/discord/websocket/cloud-send
- F-65: reminders / cron / schedules（cron-parser 依赖）
- F-66: telemetry：getTerminalTelemetrySurface + error-reporting + telemetry-exit/input/tool-events
- F-67: settings-manager 加载一次、内存同步访问、异步持久化；敏感 token 不留在内存
- F-68: startup：startup-flow/startup-log-protocol/startup-setup-pty/startup-docker-check
- F-69: 版本：version.ts 读 package.json
- F-70: 发布：prepublishOnly → build；postinstall vendor patches

## 1.5 配置与治理（F-71~F-95）
- F-71: CI 门聚合 `scripts/check.js`：typecheck / check:cycles（madge）/ check:boundaries / check:exported-functions / check:filename-casing / check:file-size / check:module-ownership / check:test-mock-isolation / check:test-coverage / check:skill-frontmatter / check:bundled-skill-scripts
- F-72: `@/` 别名强制（禁 ../ 父级 import；4 文件豁免：version/index/cli/cli-app）
- F-73: kebab-case .ts / PascalCase .tsx 强制（check-filename-casing.js）
- F-74: named exports 强制（biome style/noDefaultExport）
- F-75: AI_POLICY.md：AI 贡献必须披露、人工负责、非合规自动关闭（仿 Ghostty）
- F-76: mods 能力面默认全开（DEFAULT_MOD_CAPABILITIES）
- F-77: PROVIDERS_ONLY_MOD_CAPABILITY_PROFILE（subagent-launcher 引用）
- F-78: cross-agent-guard：跨 agent 记忆访问守卫（permissions/cross-agent-guard.ts）
- F-79: read-only-letta / read-only-shell / read-only-shell-security：只读模式
- F-80: sandbox-gate：沙箱门控（permissions/sandbox-gate.ts）
- F-81: workspace-sandbox：工作区沙箱
- F-82: shell-analysis + shell-command-normalization：shell 命令解析归一
- F-83: format-denial：工具拒绝格式化
- F-84: permissions 分层：loader（规则加载）/ matcher / checker / analyzer / session / cli
- F-85: rules 来源：canonical.ts + rule-normalization.ts
- F-86: agent-tags：GIT_MEMORY_ENABLED_TAG
- F-87: attached-repositories：附件仓库 + git-sync
- F-88: 环境变量：LETTA_MEMFS_BASE_URL / LETTA_LOCAL_BACKEND_DIR / LETTA_TRANSCRIPT_ROOT / LETTA_FS_SANDBOX / SANDBOX_ENV_VAR(LETTA_SANDBOX) / AGENT_ID / LETTA_AGENT_ID / USER_CWD / MEMORY_DIR / LETTA_MEMORY_DIR
- F-89: 目录限制默认：MEMORY_TREE_MAX_LINES/CHARS/CHILDREN_PER_DIR（utils/directory-limits）
- F-90: bunfig.toml / tsconfig.json / tsconfig.types.json / biome.json / flake.nix / bun.nix
- F-91: hooks/（husky install via prepare）
- F-92: vendor/（发布物补丁）
- F-93: docs/examples/mods（learning/memory-citations.env.json 示例）
- F-94: AGENTS.md 规则集合（@/、kebab-case、named export、worktree 工作流、不 amend commit）
- F-95: maintainability-checks.test.ts（可维护性自测）

## 1.6 外部依赖与接口（F-96~F-110）
- F-96: @earendil-works/pi-ai：anthropic/azure/bedrock/google lazy streams（backend/dev/pi-api-streams.ts）
- F-97: @letta-ai/letta-client：agents/messages/tools 资源类型
- F-98: @letta-ai/trajectory：normalizeTranscript / 导出 / review（trajectories 子命令）
- F-99: @modelcontextprotocol/sdk：MCP 客户端/运行时/OAuth
- F-100: @pierre/diffs：diff 处理
- F-101: node-pty：PTY 支持（TUI）
- F-102: react + ink-link：TUI 渲染
- F-103: sharp + @janhapke/sharp-electron：图片处理
- F-104: shiki：代码高亮
- F-105: ws：websocket channel
- F-106: glob / cron-parser / cross-spawn / open / strip-ansi：基础工具
- F-107: @scarf/scarf：telemetry
- F-108: 后端 provider：opencode-session 适配（pi-stream-adapter-opencode-session.test.ts）
- F-109: 模型面：getModelInfo / models / resolveModelHandleFromLlmConfig / reasoning_effort
- F-110: 双运行时契约：dev 用 Bun TS 源码；发布用 Node bundle

## 1.7 测试体系（F-111~F-121）
- F-111: 816 *.test.ts + 8 *.integration.test.ts
- F-112: 记忆专项测试：memory-filesystem(.integration/.sync.integration) / memory-git(.auth/.config-lock/.local-scope/.postcommit/.precommit/.retry/.signing/.v2-precommit/.windows-credentials) / memory-format / memory-constraints-audit / memory-prompt.integration / client-skills-shared-memory / memory-confinement / memory-constraints / memory-frontmatter / headless-memfs-policy / memory-worktree(.http)
- F-113: 权限专项：permissions-{analyzer,checker,cli,loader,matcher,mode,session}.test / cross-agent-guard / read-only-letta / read-only-shell(.security) / sandbox-gate / sandbox-policy / workspace-sandbox / format-denial / shell-analysis / shell-command-normalization
- F-114: 生命周期专项：headless-* 33 个测试
- F-115: integration 门：LETTA_RUN_API_INTEGRATION_TESTS=true + LETTA_API_KEY（memory-prompt.integration.test.ts）
- F-116: test-mock-isolation 强制：禁止 mock 泄漏跨测试
- F-117: test-coverage 强制（check-test-coverage.cjs）
- F-118: maintainability-checks.test.ts 仓库级自检
- F-119: skill-frontmatter 校验（内置 skills 的 frontmatter 有效性）
- F-120: bundled-skill-scripts 校验（skills 内嵌脚本可执行）
- F-121: check:cycles 禁止循环依赖（madge）

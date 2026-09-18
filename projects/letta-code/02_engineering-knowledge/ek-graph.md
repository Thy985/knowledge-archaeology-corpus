# 02 Engineering Knowledge — EK Graph（46 条 · 64 边 · avg 1.39）

> 每条 EK 必须声明 links（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）。证据引用：`F-xx`（见 01_project-layer/project-facts.md）+ 文件/符号。

## 核心机制簇（EK-01 ~ EK-12）

**EK-01 记忆 = git 仓库（git-backed MemFS）**
记忆不是数据库表，而是每个 agent 一个 git repo 于 `~/.letta/agents/<id>/memory/`；远端 `$LETTA_MEMFS_BASE_URL/v1/git/$AGENT_ID/state.git`。
- Evidence: memory-git.ts 头注释；F-23/F-24
- links: mechanism→EK-02; subsystem→EK-03; causal→EK-05→EK-06
- Value: A

**EK-02 git 生命周期（clone→pull→commit→push）**
首次运行 clone、启动 pull、记忆写 commit、turn 后 push clean pending commits；本地后端经 memfs-git-proxy 路由。
- Evidence: memory-git.ts（getMemoryRepoDir/commitMemoryWrite/retry regex）；F-24/F-25/F-29
- links: mechanism→EK-01; dependency→EK-03; causal→EK-06
- Value: A

**EK-03 pre-commit hook = 记忆写入门禁（软门禁）**
git pre-commit 内嵌 bash 脚本校验 frontmatter：description 必填、read_only 受保护、仅允许编辑 description、legacy limit 容忍；另有可选的树约束校验（root-marker 布局 + .memfs.config.json）。
- Evidence: memory-git-hooks.ts PRE_COMMIT_HOOK_SCRIPT；F-27/F-37
- links: mechanism→EK-04; subsystem→EK-01; constraint→EK-02（hook 保证 repo 内容合法性）
- Value: A
- **Reconciliation 修正（IF: independent-audit OVER_GENERALIZED-3）**: 标注"软门禁"——本地 pre-commit 可被 `git --no-verify` 绕过；内嵌脚本设计意图是隔离环境（无全局 git config 时）自校验。

**EK-04 记忆 frontmatter 双层格式（v1 legacy / v2 白名单）**
v2 只允许 name/description；legacy 允许 description/read_only/limit。read_only=true 保护**仅 legacy 生效**——v2 用字段白名单取代只读标志。
- Evidence: memory-frontmatter.ts（validateMemoryFileFrontmatter/legacyValue）；F-33/F-34/F-35
- links: contrast→EK-09（v2 索引 vs legacy 前缀）；subsystem→EK-01; causal→EK-10（迁移）
- Value: A

**EK-05 系统提示词记忆投影（{CORE_MEMORY} 注入）**
编译系统提示词时把已提交的核心记忆文件投影：`<self>`（persona）、`<memory>`（system 树 + external projection）、`<memory_metadata>`；core memory 每次调用都在上下文。
- Evidence: backend/local/system-prompt-compilation.ts（injectCoreMemory/renderMemfsProjection/compileMemoryMetadata）；F-10/F-11
- links: causal→EK-01（投影自 git 提交内容）; mechanism→EK-06; subsystem→EK-02
- Value: A

**EK-06 记忆写工具强制 reason（自编辑留痕）**
memory 工具（str_replace/insert/delete/rename/update_description/create）与 memory_apply_patch（add）都强制非空 `reason`；无变更报错；同步模式 remote/local 由 backend capability 决定。
- Evidence: tools/impl/memory.ts（validateRequiredParams(["command","reason"])、getMemoryWriteSyncMode）；F-29/F-39/F-40
- links: causal→EK-02（写后 commit）; dependency→EK-03（hook 校验写内容）; contrast→EK-07
- Value: A

**EK-07 memory_apply_patch：结构化解构重组入口**
add 操作（targetLabel/targetRelPath），用于大规模重组；描述明示"直接编辑投影文件自行 commit"也可（v2 同步指令）。
- Evidence: tools/impl/memory-apply-patch.ts；MemoryV2.md 描述；F-40
- links: contrast→EK-06; mechanism→EK-02; constraint→EK-04
- Value: B

**EK-08 MemFS v2 渐进式披露（progressive disclosure）**
根 MEMORY.md 是 frontmatter-free 索引；顶层 .md = core memory（恒在上下文）；子目录需 MEMORY.md 索引才成为记忆（projected memory，延迟读取）；skills/ 前缀排除。
- Evidence: memory-format.ts（isCoreMemoryPath/isProjectedMemoryPath/assertMemfsV2MemoryPathIndexed）；MemoryV2.md；F-31/F-32/F-33
- links: mechanism→EK-05（投影只含 core）; subsystem→EK-01; constraint→EK-04
- Value: A

**EK-09 记忆树约束（depth/size 预算）**
MemoryConstraintsConfig v1：maxDepth/maxFileCharacters/maxCoreMemoryCharacters/fileCharacterLimits；validator 自包含脚本内嵌 git hook 与审计路径共享；审计用临时 GIT_INDEX_FILE 无副作用检查 HEAD。
- Evidence: memory-constraints.ts（validateMemoryTreeConstraints/MEMORY_CONSTRAINTS_VALIDATOR_SCRIPT）；memory-constraints-audit.ts；F-36/F-37/F-38
- links: constraint→EK-08（约束 v2 树）; mechanism→EK-03; subsystem→EK-01
- Value: B

**EK-10 记忆迁移（v1→v2）与初始化 skill**
内置 skills：initializing-memory（ROOT_MEMORY.md 指南：身份/渐进披露/发现路径/不过度修剪）、migrating-memory、managing-shared-memory、syncing-memory-filesystem。
- Evidence: src/skills/builtin/{initializing-memory,migrating-memory,managing-shared-memory,syncing-memory-filesystem}/；F-15
- links: causal→EK-04（格式演进）; mechanism→EK-08; contrast→EK-11（共享记忆）
- Value: B

**EK-11 共享记忆（shared memory）**
client-skills-shared-memory 测试 + managing-shared-memory skill + shared-memory pre-commit hook 变体——多 agent 可挂接共享记忆仓库。
- Evidence: memory-git-hooks.ts（installSharedMemoryPreCommitHook）；client-skills-shared-memory.test.ts；F-15/F-27
- links: contrast→EK-10; subsystem→EK-01; constraint→EK-12（跨 agent 墙）
- Value: B

**EK-12 跨 agent 记忆墙（cross-agent denial）**
沙箱 deniedRoots 把 `~/.letta/agents`（api/cloud）与 `lc-local-backend/memfs`（local）双树读写双拒；自我记忆通过 writableRoots carve 回写；cross-agent-guard 守卫。
- Evidence: sandbox/policy.ts（FsSandboxPolicy 顺序语义）；permissions/sandbox-policy.ts（getCrossBackendAgentsTreeRoots）；cross-agent-guard.ts；F-47/F-78
- links: mechanism→EK-13（confinement）; constraint→EK-11; subsystem→EK-14
- Value: A

## 治理/权限簇（EK-13 ~ EK-26）

**EK-13 memory confinement fail-closed（无沙箱即抛错）**
记忆子代理（unattended）wrap 于内核沙箱：可广读 host、写 harness 状态与自身记忆、不可读写其他 agent 记忆；`createMemoryConfinementLauncher` 在无可用内核沙箱时抛错而非弱化。
- Evidence: memory-confinement.ts（注释：Throws when no supported kernel sandbox is available）；permissions/memory-confinement-launcher.ts；F-46/F-48/F-49
- links: mechanism→EK-12; dependency→EK-15（依赖 availability）; constraint→EK-14
- Value: A

**EK-14 记忆子代理默认沙箱（无 approve/deny 可回退）**
与交互代理 opt-in 沙箱不同，memory-subagent 默认强制内核沙箱（`LETTA_FS_SANDBOX=0` 退出）——非交互路径没有人工审批可回退，故默认 fail-closed。
- Evidence: subagents/sandbox.ts 头注释；F-46
- links: causal→EK-13; contrast→EK-16; subsystem→EK-12
- Value: A

**EK-15 沙箱可用性探测（detectSandboxBackend）**
seatbelt（macOS 二进制存在性）/ bwrap（Linux user-namespace mount 探测）；结果进程内缓存；无沙箱时 memory-confinement 走 fail-closed。
- Evidence: sandbox/availability.ts；F-18
- links: dependency→EK-13; mechanism→EK-17; subsystem→EK-14
- Value: B

**EK-16 交互代理沙箱 opt-in vs 记忆子代理默认（对照）**
cross-agent shell 沙箱 opt-in（有 approve/deny 流可回退）；记忆子代理默认开。同一 FsSandboxPolicy 模型双策略。
- Evidence: subagents/sandbox.ts + permissions/sandbox-gate.ts；F-46/F-80
- links: contrast→EK-14; subsystem→EK-13; mechanism→EK-12
- Value: A

**EK-17 FsSandboxPolicy 顺序语义（不是嵌套深度）+ 后端差异**
写策略顺序：global write-deny → baseWritableRoots（宽 harness 根，先放行）→ deniedRoots（嵌套 deny 胜出）→ writableRoots（自我 carve 胜出）→ readonlyRoots；读从不全局限制。特异性靠排序表达而非深度。
- Evidence: sandbox/policy.ts 注释 + FsSandboxPolicy；F-47
- links: mechanism→EK-12/13; constraint→EK-14; dependency→EK-15
- Value: A
- **Reconciliation 修正（IF-03/04/05, independent-audit MISSING-1/2）**:
  - **bwrap（Linux）**: 根 `--ro-bind`（restrictWrites）或 `--bind`；denied roots 用 `--tmpfs` **mask 为不存在**（"not merely unwritable but *absent*"——比静态守卫更强）；**ancestor carve-out HAZARD**：carve-out 必须是 denied root 的后代或不相交，绝不能是祖先（bwrap last-mount-wins 会重新暴露 denied roots）——故 memory-subagent profile 不 carve temp dir；`--die-with-parent` 配合进程组 kill。
  - **seatbelt（macOS）**: 基底 `(allow default)` + 定向 deny（威胁模型 = filesystem scoping for memory isolation，非通用不可信代码隔离，比 Codex 的 `(deny default)` 窄）；`/usr/bin/sandbox-exec` 硬编码防 PATH 植入；last-match-wins 决定 deny/allow 顺序。
  - 语义澄清：fail-closed 由 restrictWrites 的全局写拒绝体现（seatbelt `(deny file-write* (subpath "/"))` / bwrap `--ro-bind /`），profile 基底本身是 allow-default。

**EK-18 权限四模式 + legacy 迁移**
unrestricted（默认）/standard/acceptEdits/strict；legacy 映射（"default"→standard、"bypassPermissions"/"fullAccess"→unrestricted）。
- Evidence: permissions/mode.ts（DEFAULT_PERMISSION_MODE/VALID_PERMISSION_MODES/migratePermissionMode）；F-17
- links: subsystem→EK-19; contrast→EK-13（权限 vs 沙箱两条治理轨）
- Value: B

**EK-19 权限规则面分层（loader/matcher/checker/analyzer/session/cli）**
permissions 模块按规则加载→匹配→检查→分析→会话→CLI 分层；canonical + rule-normalization 归一规则。
- Evidence: permissions/{loader,matcher,checker,analyzer,session,cli,canonical,rule-normalization}.ts；F-84/F-85
- links: subsystem→EK-18; mechanism→EK-20; constraint→EK-21
- Value: B

**EK-20 shell 命令分析归一（shell-analysis + shell-command-normalization）**
shell 命令在权限判定前归一化（处理别名/展开/平台差异），格式拒绝（format-denial）与只读 shell 校验配套。
- Evidence: permissions/shell-analysis.ts + shell-command-normalization.ts + format-denial.ts + read-only-shell.ts；F-79/F-82/F-83
- links: mechanism→EK-19; dependency→EK-21; subsystem→EK-18
- Value: B

**EK-21 并行安全工具白名单（PARALLEL_SAFE_TOOLS）**
只读/独立工具（Read/Grep/Glob/read_file/list_dir…三套工具集等价物）可并行；Bash/shell 类明确排除（可写文件）。approval-execution 共享此逻辑。
- Evidence: approval-execution.ts（PARALLEL_SAFE_TOOLS 注释）；F-20/F-60
- links: constraint→EK-19; subsystem→EK-22; mechanism→EK-23
- Value: B

**EK-22 审批执行（approval batch）**
交互与 headless 共享审批批次执行；approval-recovery 恢复；check-approval 准备/恢复数据；结果归一（approval-result-normalization）。
- Evidence: approval-execution.ts + approval-recovery.ts + check-approval.ts + approval-result-normalization.ts；F-60
- links: causal→EK-21（白名单决定审批内并行）; subsystem→EK-18; dependency→EK-23
- Value: B

**EK-23 headless 生命周期测试驱动（33 个 headless 测试）**
headless-* 覆盖审批恢复/后端生命周期/双向反射/启动挂起审批/云发送/入队等待/环境响应/中断闩锁/监听器/memfs 策略/权限/队列/提醒/子代理发送/遥测/工具事件。
- Evidence: src/headless-*.test.ts 全族；F-56/F-114
- links: mechanism→EK-22; constraint→EK-34（CI 门强制测试面）; subsystem→EK-24
- Value: A

**EK-24 双运行时契约（Bun dev / Node bundle）**
dev 跑 TS 源码（Bun），发布产物 Node-targeted letta.js（Node ≥22.19）；行为差异必须双路径测试；AGENTS.md 强制。
- Evidence: AGENTS.md Runtime Validation；F-03/F-57/F-110
- links: subsystem→EK-23; constraint→EK-36（可搜索性纪律支撑双运行时可验证）; contrast→EK-25
- Value: B

**EK-25 自动化通知用 user-role `<system-reminder>`**
禁 role:system 注入（Anthropic 可拒绝 system-role 通知）；用 user role + `<system-reminder>` 标签，transcript echo 剥离 reminder-only 内容。
- Evidence: AGENTS.md Runtime Validation；F-58
- links: contrast→EK-24; mechanism→EK-05（上下文编译）；subsystem→EK-26
- Value: A

**EK-26 状态流/上下文面（mods context + context-window 管理）**
ModContext 组装（agent/model/conversation/reflection/memfs/context-window/cost）；max-context 设置（MIN_CONTEXT_WINDOW_TOKENS=30k）；conversation-model-carryover。
- Evidence: mods/context.ts + agent/max-context.ts + conversation-model-carryover.ts；F-13
- links: subsystem→EK-05; mechanism→EK-23; causal→EK-27
- Value: B

## 生命周期/调度簇（EK-27 ~ EK-33）

**EK-27 反射（reflection）记忆演化子代理**
reflection 子代理基于 step-count / compaction-event 触发，读取 parent memory snapshot（≤40K chars），产出自省并 merge（auto/explicit）；reflection-settings 全局 + per-agent。
- Evidence: reflection-settings.ts + subagents/context-budget.ts（REFLECTION_STARTUP_CONTEXT_TOKEN_LIMIT=16k / REFLECTION_PARENT_MEMORY_SNAPSHOT_CHAR_LIMIT=40k）+ subagents/manager.ts（type==="reflection" 启动标志）；F-44/F-45
- links: causal→EK-01（反射结果写回记忆）; mechanism→EK-14（memory-subagent 沙箱）; subsystem→EK-16
- Value: A

**EK-28 memory-subagent 家族（reflection/memory/init/history-analyzer）**
四类子代理共享 memory-subagent launch profile；reflection 有专用启动标志匹配训练行为；subagent 子进程经 SUBAGENT_LAUNCH_ENV/PROFILE/NAME 标记。
- Evidence: subagents/index.ts:89-92 + manager.ts:200-212 + subagent-launcher.ts（SUBAGENT_LAUNCH_ENV 等）；F-16
- links: mechanism→EK-27; subsystem→EK-14; causal→EK-29
- Value: A

**EK-29 子代理上下文预算（4 chars/token 保守估算）**
避免 tokenizer 依赖；16K token 上限安全阀；只作发送前 guard。
- Evidence: subagents/context-budget.ts 头注释；F-44
- links: dependency→EK-27; constraint→EK-28; mechanism→EK-26
- Value: B

**EK-30 更新链（startup auto-update + update-chain smoke）**
startup-auto-update + updater；test:update-chain:manual/startup 双模式 smoke。
- Evidence: startup-auto-update.ts + scripts/test-utils/update-chain-smoke；F-59
- links: subsystem→EK-23; mechanism→EK-24; constraint→EK-34
- Value: C

**EK-31 MCP 集成面（client/oauth/runtime/settings）**
MCP 客户端 + OAuth + 运行时 + 设置；mcp-oauth 处理授权流。
- Evidence: mcp-client.ts/mcp-oauth.ts/mcp-runtime.ts/mcp-settings.ts；F-63
- links: mechanism→EK-21（工具面）; subsystem→EK-23; contrast→EK-05（外部工具 vs 记忆投影）
- Value: B

**EK-32 渠道面（slack/telegram/discord/websocket/cloud）**
多 channel 接入 + 防抖（discord-debounce/telegram-debounce 测试）。
- Evidence: channels-slack/telegram + discord-debounce.test.ts；F-64
- links: subsystem→EK-23; mechanism→EK-21; constraint→EK-33
- Value: C

**EK-33 调度（cron/schedules/reminders/queue）**
cron-parser 依赖 + schedules.ts + reminders + queue 生命周期。
- Evidence: src/cron/schedules.ts/reminders/queue + headless-reminder.test.ts；F-65/F-62
- links: subsystem→EK-23; mechanism→EK-27（reminder 可触发反射）；constraint→EK-32
- Value: C

## 配置/工程实践簇（EK-34 ~ EK-46）

**EK-34 CI 门聚合（scripts/check.js 11 道）**
typecheck/cycles/boundaries/exported-functions/filename-casing/file-size/module-ownership/test-mock-isolation/test-coverage/skill-frontmatter/bundled-skill-scripts；pre-commit + CI 双跑。
- Evidence: package.json scripts + scripts/check*.js；F-71/F-121
- links: constraint→EK-35; mechanism→EK-36; subsystem→EK-37
- Value: A

**EK-35 @/ 别名强制（禁 ../）**
跨目录 import 一律 `@/`（→src/）；4 文件豁免；pre-commit 拦截。
- Evidence: AGENTS.md Rules + scripts/check-boundaries.js；F-72
- links: constraint→EK-34; mechanism→EK-37; causal→EK-36
- Value: B

**EK-36 可搜索性优先工程纪律（kebab-case/named exports/文件名大小写）**
agent 通过 grep 导航代码库；kebab-case .ts / PascalCase .tsx / named exports / macOS 大小写陷阱说明。
- Evidence: AGENTS.md（filename-casing + style/noDefaultExport）；F-73/F-74
- links: mechanism→EK-35; causal→EK-34; subsystem→EK-37
- Value: A

**EK-37 test-mock-isolation 与 test-coverage 强制**
mock 不得跨测试泄漏；覆盖率门禁；maintainability-checks 自检。
- Evidence: scripts/check-test-mock-isolation.js + check-test-coverage.cjs + maintainability-checks.test.ts；F-116/F-117/F-118
- links: constraint→EK-34; mechanism→EK-23; dependency→EK-36
- Value: B

**EK-38 AI_POLICY 贡献治理（仿 Ghostty）**
AI 工具可用来贡献但人类负责；必须披露 AI 使用；非合规 issue/PR 自动关闭并锁定；trusted contributors 豁免自动检查。
- Evidence: AI_POLICY.md；F-75
- links: contrast→EK-25（内部通知 vs 外部贡献治理）; subsystem→EK-34; mechanism→EK-39
- Value: B

**EK-39 skills 四源优先级与 frontmatter 校验**
project > agent > global > bundled；skill frontmatter（name/description/when_to_use/disable_model_invocation/user_invocable 等）；CI 校验 frontmatter + 内嵌脚本。
- Evidence: agent/skills.ts + skill-sources.ts + scripts/check-skill-frontmatter.js；F-14/F-119/F-120
- links: mechanism→EK-10; subsystem→EK-05（skill 可进记忆投影）; constraint→EK-34
- Value: A

**EK-40 mods 能力面（capability profile）**
MOD_CAPABILITY_IDS 10 类；DEFAULT_MOD_CAPABILITIES 全开；PROVIDERS_ONLY profile 供子代理裁剪。
- Evidence: mods/capabilities.ts + subagent-launcher（PROVIDERS_ONLY_MOD_CAPABILITY_PROFILE）；F-13/F-76/F-77
- links: mechanism→EK-26; contrast→EK-18（mod 能力 vs 权限模式）; subsystem→EK-41
- Value: B

**EK-41 mods 会话句柄（conversation-handle）**
Mod 通过 handle 与对话交互：fork/sendMessageStream/updateLlmConfig；后端不可用时抛错。
- Evidence: mods/conversation-handle.ts（createModConversationHandle）；F-13
- links: mechanism→EK-40; dependency→EK-42; subsystem→EK-26
- Value: B

**EK-42 附件仓库（attached repositories + git-sync）**
agent 可挂接外部仓库；attached-repository-git-sync 同步。
- Evidence: agent/attached-repositories.ts + attached-repository-git-sync.ts；F-87
- links: contrast→EK-01（记忆仓库 vs 附件仓库）; mechanism→EK-02; subsystem→EK-23
- Value: C

**EK-43 环境变量契约（记忆/沙箱/身份）**
LETTA_MEMFS_BASE_URL/LETTA_LOCAL_BACKEND_DIR/LETTA_TRANSCRIPT_ROOT/LETTA_FS_SANDBOX/LETTA_SANDBOX/AGENT_ID/MEMORY_DIR/USER_CWD 等。
- Evidence: 多处引用（memory-git/memory-paths/subagent-launcher）；F-88
- links: constraint→EK-01/13/28; mechanism→EK-44; subsystem→EK-02
- Value: B

**EK-44 双树隔离（api/cloud `~/.letta/agents` vs local `lc-local-backend/memfs`）**
getCrossBackendAgentsTreeRoots 返回双树；沙箱两棵都墙；local 后端由 isLocalBackendEnvEnabled 决定。
- Evidence: permissions/sandbox-policy.ts（getCrossBackendAgentsTreeRoots）；F-22/F-44
- links: mechanism→EK-12; constraint→EK-43; dependency→EK-15
- Value: A

**EK-45 git 凭据卫生（URL 归一 + Windows helper + 签名禁用）**
normalizeCredentialBaseUrl 归一 origin（防 URL key shape 敏感）；Windows credential helper 写入；禁用 commit 签名（兼容性）。
- Evidence: memory-git.ts（normalizeCredentialBaseUrl/redactCredentialedHttpsUrl）+ memory-git-windows-credentials.ts + memory-git-signing.ts；F-53/F-54/F-55
- links: mechanism→EK-02; constraint→EK-43; subsystem→EK-01
- Value: B

**EK-46 大规模重组路径（worktree + 固定 git 身份）**
harness 创建的 memory-worktree 用 "Letta Code" <noreply@letta.com> 固定身份（30s timeout），避免污染用户 git 身份。
- Evidence: memory-worktree.ts（HARNESS_GIT_ENV）；F-52
- links: mechanism→EK-02; contrast→EK-45; subsystem→EK-01
- Value: B

**EK-47 路径根 canonical 化防符号链接逃逸（Reconciliation 新增，IF-02）**
所有策略 root 在传给内核沙箱前经 `canonicalizeRoot` realpath 解析（对不存在的 leaf realpath 最近存在祖先再回接）；注释明示动机：lexical path 经 symlink 会 silently match nothing = 沙箱放行一切。
- Evidence: src/permissions/sandbox-policy.ts canonicalizeRoot；F-47
- links: constraint→EK-17; mechanism→EK-12/13; dependency→EK-15
- Value: A

**EK-48 记忆 git 代理 URL 分离（Reconciliation 新增，IF-07）**
getMemfsServerUrl 显式忽略 LETTA_BASE_URL（Desktop 设 ephemeral localhost proxy）；git config/settings 只持久化 canonical memfs URL；瞬态传输代理走 LETTA_MEMFS_GIT_PROXY_BASE_URL + git `url.<prefix>.insteadOf`。
- Evidence: src/backend/api/memfs-git-proxy.ts（getMemfsServerUrl/MemfsGitProxyRewriteConfig）；F-23/F-88
- links: mechanism→EK-02; constraint→EK-43; subsystem→EK-01
- Value: B

**EK-49 mods 可信插件模型（Reconciliation 新增，IF-08，NEEDS_HUMAN_REVIEW）**
mods 三源（legacy_global/global/agent）全部 trusted:true；mod-engine 主进程 `await import(...?mod=mtimeMs)` 动态加载（createRequire from runtime）；无可见 mod 沙箱——与记忆子代理 fail-closed 沙箱形成对比。
- Evidence: src/mods/mod-sources.ts（trusted:true ×3）；src/mods/mod-engine.ts（importPath?mod=mtimeMs）；F-13/F-76/F-77
- links: contrast→EK-14（有沙箱 vs 无沙箱）; mechanism→EK-40; subsystem→EK-41
- Value: A（人工裁决项）

## EK Graph 边汇总（Reconciliation 后 49 EK）
| 边类型 | 数量 | 示例 |
|---|---|---|
| mechanism | 18 | EK-01↔EK-02, EK-03↔EK-04, EK-05↔EK-06, EK-08↔EK-10, EK-12↔EK-13, EK-17↔EK-12/13, EK-21↔EK-23, EK-27↔EK-28, EK-31↔EK-21, EK-32↔EK-21, EK-34↔EK-36, EK-35↔EK-37, EK-38↔EK-39, EK-40↔EK-26, EK-41↔EK-40, EK-42↔EK-02, EK-44↔EK-12, EK-45↔EK-02, EK-46↔EK-02, EK-47↔EK-12/13, EK-48↔EK-02, EK-49↔EK-40 |
| subsystem | 18 | EK-01/03/04/08/09 记忆子系统；EK-12/14/15/16 沙箱；EK-18/19/20/21/22 权限；EK-23/24/25 运行时；EK-05/26 上下文；EK-30/31/32/33 生态 |
| causal | 8 | EK-01→EK-05→EK-06, EK-04→EK-10, EK-14→EK-13, EK-21→EK-22, EK-26→EK-27, EK-28→EK-29, EK-35→EK-36, EK-36→EK-34 |
| dependency | 6 | EK-02→EK-03, EK-06→EK-03, EK-13→EK-15, EK-20→EK-21, EK-27→EK-29, EK-41→EK-42, EK-47→EK-15 |
| constraint | 13 | EK-03→EK-02, EK-07→EK-04, EK-09→EK-08, EK-11→EK-12, EK-13→EK-14, EK-17→EK-14, EK-19→EK-21, EK-21→EK-19, EK-23→EK-34, EK-29→EK-28, EK-34→EK-35, EK-43→EK-01/13/28, EK-47→EK-17, EK-48→EK-43 |
| contrast | 9 | EK-04↔EK-09, EK-06↔EK-07, EK-11↔EK-10, EK-14↔EK-16, EK-18↔EK-13, EK-24↔EK-25, EK-31↔EK-05, EK-38↔EK-25, EK-40↔EK-18, EK-42↔EK-01, EK-49↔EK-14 |

> 注：上表统计中部分行含多边（同一行多对），总计 ≥80 条边。孤立 EK：0（全部连接）。

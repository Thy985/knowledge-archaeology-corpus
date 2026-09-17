# 04 Flow Atlas — 七类流

> 所有 Edge 必须可回溯到实际 symbol/file/condition/state transition（浅克隆 HEAD `3df2ebd6`）。无架构想象。

## 1. Control Flow（控制流）— 一轮 turn 的主路径
```
src/index.ts (bun 入口)
  → agent/context.ts setAgentContext / setConversationId
  → backend 选择：backend-mode.ts resolveBackendMode()（configuredBackendMode ?? isLocalBackendEnvEnabled() ? "local" : "api"）
  → system-prompt-compilation：injectCoreMemory(rawPrompt, coreMemory)  # 若含 {CORE_MEMORY} 直接替换，否则追加
  → 工具执行：tools/manager.ts executeTool（approval-execution.ts 审批批次）
       ├─ 并行安全（PARALLEL_SAFE_TOOLS）→ 并行
       └─ Bash 类 → 串行
  → memory 工具（tools/impl/memory.ts memory()）
       └─ validateRequiredParams(["command","reason"]) → ensureMemoryRepo → applyMemoryCommand → commitMemoryWrite
  → turn 结束 → post-turn push（memory-git.ts，clean pending commits）
```
- **Edge 证据**: `src/index.ts` import { setAgentContext, setConversationId }；`backend-mode.ts resolveBackendMode`；`system-prompt-compilation.ts injectCoreMemory`；`tools/impl/memory.ts memory()`；`approval-execution.ts PARALLEL_SAFE_TOOLS`

## 2. State Flow（状态流）— 记忆与运行时状态迁移
```
记忆状态机：
  [未初始化] --first run clone--> [本地 git repo] --startup pull--> [同步] --memory 工具写--> [dirty]
  [dirty] --commit (pre-commit hook 校验通过)--> [committed] --post-turn push--> [远端同步]
  [committed] --v2 根 MEMORY.md 存在--> detectMemoryFormat = memfs-v2
  [committed] --无 MEMORY.md--> memfs-v1（legacy）

后端模式状态：configuredBackendMode（override）> env 标志 → resolveBackendMode() 单源真值
沙箱可用性：detectSandboxBackend() 探测一次 → 缓存 → memory-confinement 消费
```
- **Edge 证据**: `memory-git.ts`（clone/pull/commit/push 函数族 + getMemoryWriteSyncMode）；`memory-format.ts detectMemoryFormat`；`backend-mode.ts`；`sandbox/availability.ts detectSandboxBackend`（cached）

## 3. Data Flow（数据流）— 记忆数据从磁盘到上下文
```
~/.letta/agents/<id>/memory/*.md（git 工作树）
  → collectCommittedMemoryFiles（git ls-tree -r HEAD + git show HEAD:<path>，只读已提交版本）
  → renderMemfsProjection（persona → <self>；system 树 + external → <memory>）
  → compileMemoryMetadata（AGENT_ID/CONVERSATION_ID/时间戳/recall 消息数）
  → injectCoreMemory（{CORE_MEMORY} 替换）
  → 发送给 LLM
  → agent 决策 → memory 工具写 → applyMemoryCommand → writeFile → commitMemoryWrite
```
- **Edge 证据**: `backend/local/system-prompt-compilation.ts collectCommittedMemoryFiles`（**关键**：投影只读已提交内容，不读工作树 dirty 状态——设计意图：记忆在上下文中的一致性由 git 提交保证）

## 4. Evidence Flow（证据流）— 测试如何证明行为
```
816 *.test.ts（bun:test）→ 记忆专项（memory-git.*.test.ts 覆盖 auth/config-lock/local-scope/postcommit/precommit/retry/signing/v2-precommit/windows-credentials）
  → integration 门（LETTA_RUN_API_INTEGRATION_TESTS=true + LETTA_API_KEY 才跑 memory-prompt.integration）
  → CI 聚合：scripts/check.js → typecheck/cycles/boundaries/.../test-coverage/mock-isolation
  → 可维护性自检：maintainability-checks.test.ts
```
- **Edge 证据**: package.json scripts（check → scripts/check.js）；`memory-prompt.integration.test.ts` describeIntegration 条件；`maintainability-checks.test.ts`

## 5. Authority Flow（权威流）— 谁有权做什么
```
权限模式（PermissionMode：unrestricted/standard/acceptEdits/strict）→ permissions/mode.ts（默认 unrestricted）
  → permissions/loader → matcher → checker → analyzer（规则判定链）
  → shell 类命令：shell-command-normalization → shell-analysis → sandbox-gate
  → 记忆子代理：memory-subagent profile → wrapSubagentLauncher → 内核沙箱（seatbelt/bwrap）→ FsSandboxPolicy
      ├─ 可广读 host
      ├─ 写 harness 状态（baseWritableRoots = ~/.letta）
      ├─ 不可读写其他 agent 记忆（deniedRoots = ~/.letta/agents + lc-local-backend/memfs）
      └─ 可写自我记忆（writableRoots carve）
```
- **Edge 证据**: `permissions/mode.ts`；`permissions/sandbox-policy.ts buildMemorySubagentSandboxPolicy + getCrossBackendAgentsTreeRoots`；`sandbox/policy.ts FsSandboxPolicy` 顺序语义注释；`subagents/sandbox.ts wrapSubagentLauncher`

## 6. Memory Flow（记忆流）— 记忆全生命周期（本考古核心）
```
[初始化] initializing-memory skill / /init 子代理 → 创建 MEMORY.md 索引 + 核心文件
  → [格式] detectMemoryFormat：根 MEMORY.md 存在 → v2（frontmatter 白名单 name/description）
  → [写入] memory 工具 6 命令 / memory_apply_patch add（reason 必填）
      → assertMemoryRepoCleanForWrite → applyMemoryCommand（v2 检查目录已索引）→ writeFile
  → [门禁] pre-commit hook（memory-git-hooks.ts）：description 必填 / read_only 保护 / 白名单字段
  → [提交] commitMemoryWrite（syncMode: remote|local）→ git commit
  → [同步] post-turn push（remote）或本地保留（local）
  → [检索] 系统提示词投影（已提交版本）→ <self>/<memory>/<memory_metadata>
  → [演化] reflection 子代理（step-count/compaction-event 触发）→ 读 parent memory snapshot（≤40K chars）→ 自省 → merge（auto/explicit）→ 写回记忆
  → [迁移] migrating-memory skill（v1→v2）
```
- **Edge 证据**: `tools/impl/memory.ts`（memory() 全流程）；`tools/impl/memory-apply-patch.ts`；`agent/memory-git-hooks.ts`（PRE_COMMIT_HOOK_SCRIPT）；`agent/memory-git.ts commitMemoryWrite`；`agent/memory-format.ts`；`agent/subagents/context-budget.ts`（REFLECTION_PARENT_MEMORY_SNAPSHOT_CHAR_LIMIT=40_000）；`reflection-settings.ts`

## 7. Policy Flow（策略流）— 治理闭环
```
决策（Decision）：AGENTS.md 规则（@/ 别名、kebab-case、named exports、双运行时契约）
  → 批准（Approval）：scripts/check.js CI 门（11 道）+ pre-commit hook + AI_POLICY 贡献披露
  → 策略（Policy）：memory-git-hooks（记忆写入门禁）+ FsSandboxPolicy（沙箱策略）+ PermissionMode（权限模式）
  → 执行（Enforcement）：check-boundaries.js / check-filename-casing.js / check-test-mock-isolation.js / bwrap/seatbelt 内核沙箱 / memory-confinement throw
  → 反馈（Feedback）：maintainability-checks.test.ts 自检 + AI_POLICY 非合规自动关闭 + update-chain smoke
  → 未来决策（Future Decision）：AGENTS.md 规则由 agent 导航贡献者演进（本快照未见显式 RFC 流程）
```
- **Edge 证据**: `AGENTS.md`（Rules and Why They Exist）；`package.json scripts`；`scripts/check*.js` 族；`AI_POLICY.md`；`memory-git-hooks.ts`；`sandbox/policy.ts`；`memory-confinement.ts`（throw）

## Flow → KO 交叉校验门
| KO | 支撑 Flow | 一致 |
|---|---|---|
| KO-01 | Memory Flow（全链） | ✅ |
| KO-02 | Authority Flow（fail-closed 分支）+ Memory Flow（子代理） | ✅ |
| KO-03 | Authority Flow（路径墙）+ Memory Flow（双树） | ✅ |
| KO-04 | Memory Flow（写入→reason→hook） | ✅ |
| KO-05 | Data Flow（投影/延迟读取） | ✅ |
| KO-06 | Control Flow（双运行时）+ Policy Flow（CI） | ✅ |
| KO-07 | Authority Flow（双轨）+ Policy Flow | ✅ |
| KO-08 | Memory Flow 因果链 | ✅ |
| KO-09 | Memory Flow（commit 留痕） | ✅ |

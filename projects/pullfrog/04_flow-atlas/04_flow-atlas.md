# PullFrog Flow Atlas（七类流）

> 从真实代码导出（commit d287f81e）。每条 Edge 标注 symbol/file/condition/state transition，可回溯。七类流：Control / State / Data / Evidence / Authority / Memory / Policy。

---

## 1. Control Flow（控制流）—— main.ts run 编排链

```
entry.ts → main.ts(main) ── 检查点链（按序，每步失败 → catch）──>
  normalizeEnv → UNSAFE_OVERRIDES → usage-summary 信号 → resolveRunContextData
  → resolvePayload → initToolState → OIDC stash → commercialRefused?(refuse→comment+exit)
  → createTempDirectory → opencode install → captureBaselineModels
  → dbSecrets 注入 → credential pool → installCodexAuth/XaiAuth → captureAuthorizedModels
  → envAllowlist → resolveTokens → wipeRunnerLeakSurface → shell probe
  → [shell!=="enabled" → 删 OIDC env] → runProxyResolution(402/503→BillingError→render)
  → resolveBody → gitAuthServer → resolveModel → resolvePoolModel → selectConfiguredCredential
  → decideModelAccess → ossEffortFloor → checkConfiguredCredentials → materializeVertexCredentials
  → resolveAgent(codexAgent opt-in OR canary) → validateAgentApiKey(NoUsableCredential→trial fallback→re-resolve)
  → setupGit → resolvePackageManagerSpec/provisionPackageManager → executeLifecycleHook(setup)
  → computeModes → startMcpHttpServer → subagentDeniedToolNames → learnings/summary seed
  → startInstallation → resolveInstructions → findDanglingPromptToolRefs(WARN)
  → [opencode plugin 检测] → 双超时(activity+timeoutPromise+safety-net) → outputSchema 检查
  → finalizeSuccessRun(postReview→persistSummary→persistLearnings→error-report→stranded-comment cleanup
     →job summary→output marker)
        │ 任何 throw
        ▼
  catch: progressCallbackDisabled → killTrackedChildren → renderRunError → writeRunErrorOutputs
     → persistRunArtifacts → reportStatusChecks(false)
        │ 总是
        ▼
  finally: activityTimeout stop → safetyNet clear → killTrackedChildren(#862) → usage summary → aggregateUsage PATCH
```

关键 Edge 可回溯：`main.ts` 全程（1102 行按序）；`entryPost.ts`（post step always()，Codex auth write-back）；`utils/runErrorRenderer.ts`（renderRunError 只发布"我们的"错误，#938/BYOK 模式匹配 + stop-hook 任意文本不发布）。

## 2. State Flow（状态流）—— ToolState 与 checkout 状态机

```
initToolState（main.ts 早段，空态）
  └─ ToolState: { review, finalSummaryWritten, selectedMode, standaloneCommentId, repos: Map<owner/name, RepoToolState> }
       └─ RepoToolState（每次 checkout_pr 创建/更新）:
            initialHead(kind: branch|detached ← configureRepoGit 捕获)
            beforeSha(entry HEAD) → checkoutSha(目标 PR merge)
            pushUrl/pushDest(push 白名单) · commentableLinesByFile(双 key: pullNumber+checkoutSha)
            access(primary|write|read) · repoVerified · deepen 进度(state|done)

checkout_pr 状态机（mcp/checkout.ts + toolState.ts）:
  IDLE → VERIFY(repoVerified, OIDC/fork/visibility) → FETCH(deepen 计算: base/head/beforeSha 三终点)
       → MERGE_CHECKOUT(checkoutSha) → [mid-review 新 commit? → advance checkoutSha→beforeSha]
       → RESET/restore(beforeSha 锚定回滚)

push_branch 状态机（mcp/git.ts）:
  pushUrl 解析 → pushDest 白名单校验(restricted: 禁默认分支/禁 delete branch/禁 tag)
       → prepush hook 运行 → 变更检测(fork URL 等) → push
```

关键 Edge：`initialHead` 无 kind 标签 → 任何 detached 态通过（挡住跨 PR clobber 的 HEAD 继承链，EK-28）；`commentableLinesByFile` 双 key 防 checkout 间 stale 快照静默错校验（EK-29）。

## 3. Data Flow（数据流）—— payload 到工具的走向

```
GitHub 事件 → action.yml(inputs) → resolveRunContextData(GITHUB_EVENT_PATH/run-type/API)
  → resolvePayload(jsonPayload.event? / inputs / repoSettings) → ResolvedPayload
  → startMcpHttpServer(ToolContext live getters: repos/ghToken/refreshGitToken/autoMergeEnabled…)
  → agent(Claude|Codex|OpenCode) ──MCP──> 工具面:
       checkout_pr → gh octokit(installation token)
       git         → $git()(ASKPASS) → safeGit 包装
       shell       → spawnShell(SandboxMethod) → /bin/bash(ASKPASS_PROCESS? null) 
       gh          → gh CLI(ghToken env) → 子进程输出(stdout/stderr/exit)
  → 工具结果 → agent turn → finalize → GitHub API(评论/check/PR)
```

关键 Edge：`ToolContext` live getter 而非快照（防 push 工具重放已吊销 token，#964）；`gh` 工具低于 mirror 阈值根本不注册（工具面 = 数据面过滤）；`external GH_TOKEN` 同时充当 git+mcp token（数据面 scope 降级为用户自选）。

## 4. Evidence Flow（证据流）—— run 的事实如何产生并被校验

```
seed: learnings(缓存, 字节对比) + PR summary(issue_number? / commits 解析) 注入系统 prompt
   → agent 探索(工具调用) → structured output?(output_schema 校验: 无效 → throw)
   → 工具结果校验链: subagent 门控(claudePretoolGate exit 0|2) · git safeGit 包装
       · shell 沙箱 · review diff-coverage pre-flight(一次 nudge) · checkout 不变量
   → finalize: persistSummary → persistLearnings → report_status_checks/approval check
   → post-run gates: stopHook(禁用#714) / dirtyTree(非 NON_COMMITTING 抑制) /
       summaryStale(一次性) / unsubmittedReview(硬门) → retry ≤ MAX_POST_RUN_RETRIES=3
```

关键 Edge：`getUnsubmittedReview` 按模式分流（Review 只认 create_pull_request_review）；`expectsReviewOutput` 在 progressComments:disabled 时降级 `silent!==true && issue_number!==undefined`；`output_schema` 提供时 action output 变 required（action.yml）。

## 5. Authority Flow（权威流）—— 谁能做什么（权限分层）

```
事件 authorPermission(admin/maintain/write/triage/read/none, 来自 event payload)
  → isCollaborator? (admin|maintain|write)
  → shell 解析: repoShell(inputs.shell 只可更严 disabled>restricted>enabled)
       + 非协作者强制 ≥restricted
  → push 解析: inputs.push ?? repoSettings.push ?? "restricted"(fallback)
  → token 授权: gitToken(可渗漏: push=disabled→contents:read / enabled|restricted→contents:write+workflows:write)
       mcpToken(不可渗漏: contents+pr+issues+checks write, actions read)
       ghToken(可打印: 角色镜像, 低于阈值 undefined→工具不注册)
       readToken(xrepo 只读)
  → subagent: 工具 mutates 派生禁调用集(空集 throw) → PreToolUse(exit 2)/tool.execute.before
  → native FS: .git 写 deny + .git/config 读 deny
  → sandbox: shell probe → FS_MOUNTS(.git ro-bind, tmpfs secrets, tmpfs env)
  → post-run: finalizeAgentResult(terminal hard-fail 决策权在运行时)
```

关键 Edge：payload.ts 注释"permissions are intentionally NOT included in Inputs to prevent injection attacks——从 event.authorPermission 派生"；configuration.ts privileged 字段（push/shell/env-allowlist/signed-commits/auto-merge/oss 不可被任意 repo 配置）；approvalCheck OR 语义（只能打开不能关闭）。

## 6. Memory Flow（记忆流）—— 跨 run 记忆

```
服务端: Repo.learnings + learningsHeadings(TOC, 服务端解析, JSON 边界传递)
   → seed(字节对比, 未变化不重 seed) → agent 维护 → persistLearnings
xrepo: org 级 xrepoBrief(operator-authored) + xrepoLearnings(agent-curated 跨 run)
   → 仅 --xrepo runs → buildXrepoLearningsReflectionPrompt(IncrementalReview 跳过)
learnings 语义: 每条带 file/line + 适用条件; "增加"代理读到 apply=true 会 actionable
```

关键 Edge：learnings 种子是 run 级缓存（agent 不自证）；`PULLFROG_DISABLE_LEARNINGS_REFLECTION` 测试关闭开关（test fixtures env）；字节对比防重复 seed 造成 token 膨胀。

## 7. Policy Flow（策略流）—— 治理闭环（Decision→Approval→Policy→Enforcement→Future Decision）

```
Decision: 服务端 dispatch 决策(runType/review|build 等) + mode checklist prompt(modes.ts)
Approval: GitHub 事件授权(authorPermission/trigger) + repo 设置(statusChecks/approvalCheck)
Policy 固化: 仓库内配置(AGENTS.md/权限设置经 repoSettings 传递) + configuration.ts(org/repo 字段)
Enforcement:
  - runtime: resolvePayload 权限解析 · token scope · subagent 门控 · FS deny · sandbox · NOSHELL 阻断表
  - 模式策略: NON_COMMITTING_MODES(Review/IncrementalReview/Plan 禁 commit 类变更)
    · signedCommits 分支 · PR_SUMMARY_FORMAT(严重度 emoji + details block + metadata comment)
  - 交付策略: post-run gates(软/硬) + MAX_POST_RUN_RETRIES=3
  - 自治策略: autoMergeEnabled(全局 kill switch isAutonomousMaintenanceEnabled AND 服务端) —
    "already globally-gated server-side, runtime treats as final verdict"
Future Decision: post-run review 循环(AddressReviews 模式) + learnings 反馈 → 下个 run 的指令/行为基线
```

关键 Edge：`PR_SUMMARY_FORMAT`（modes.ts）定义 GitHub 解析器可读的格式（`details` block + metadata comment）；`commit_changes vs git push` 随 signedCommits 切换（modes.ts 实现分支）；policy 与实现分离：服务端 policy（不可见）与 action 侧 enforcement（本仓库）在 run-context JSON 边界握手，Action 侧无法改写服务端 policy。

---

## Flow→KO 交叉校验

| KO | 支撑 Flow | 校验 |
|----|----------|------|
| KO-01 Fail-Closed | Authority（权限解析）、Control（catch 链） | 一致 |
| KO-02 事故防御链 | Authority（subagent 门控）、State（initialHead） | 一致 |
| KO-03 凭证生命周期 | Authority（token 授权）、Data（ToolContext live getter） | 一致 |
| KO-04 可打印凭证收敛 | Authority（ghToken）、Data（工具注册） | 一致 |
| KO-05 失联治理 | Control（双超时+safety-net）、Data（SSE event） | 一致 |
| KO-06 可见交付门控 | Evidence（post-run gates）、Policy（模式出口） | 一致 |
| KO-07 沙箱纵深 | Authority（FS_MOUNTS）、Data（shell 子进程） | 一致 |
| KO-08 执行/决策解耦 | Authority 全链 + Policy（NON_COMMITTING） | 一致 |

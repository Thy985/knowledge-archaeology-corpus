# PullFrog Project Layer（项目地图）

> run_id: ARCH-2026-10-02-001 · commit d287f81e · snapshot 见 `../snapshot/snapshot_artifact.md`（12 节，含语言构成/入口/核心模块行数/测试三层/权限清单）

## 1. 定位与运行形态

- 一个 **GitHub Actions 自定义 action**（`runs.using: "node24"`，`main: entry.ts`，`post: entryPost.ts` with `post-if: always()`），在 runner 上把单次 GitHub 事件（PR/Issue/评论/check_suite 等）转成一次**托管编码 agent 的受控执行**。证据：`action.yml`。
- 双入口：`entry.ts`（1 行 re-export main）+ `entryPost.ts`（Codex auth.json 刷新 write-back，post 步骤 best-effort 持久化，主步骤被取消/超时/异常也要跑）。证据：`action.yml` 注释 + `entryPost.ts`。
- 服务端边界：run-context JSON 边界（`utils/runContext.ts` 的 `RepoSettings`/`LearningsHeading` 等接口），action 侧无法跨到服务端（注释明言"typecheck can't enforce shape equality across both sides"）。

## 2. 语言 / 构建 / 运行环境

| 项 | 值 | 证据 |
|----|----|------|
| 语言 | TypeScript（243 ts / 12 yml / 6 json / 4 md，279 文件） | snapshot |
| 版本 | 0.1.94（package.json），cliContractVersion 1（cliContract.ts） | snapshot |
| 包管理 | pnpm 10.27.0，node24，esbuild bundle → dist/main.mjs | snapshot + esbuild.config.js |
| Docker | Dockerfile + docker-entrypoint.sh + docker.ts（17KB，本地/自托管运行形态） | 顶层 |

## 3. 核心模块（按行数）

| 模块 | 行数 | 职责 |
|------|------|------|
| `models.ts` | ~2500（97KB） | model alias registry：slug→models.dev specifier；bedrock/vertex/azure 是 routing slug（运行时从 env 解析）；`isFree` 语义与"无需 key"区分（#1077） |
| `modes.ts` | ~1200（48KB） | 8 种模式 checklist prompt + PR_SUMMARY_FORMAT + NON_COMMITTING_MODES |
| `main.ts` | 1102 | run 编排（~40 检查点全流程，见 00_overview 与 Flow Atlas） |
| `external.ts` | 528 | 类型边界：AgentId、PayloadEvent 14 trigger、XrepoConfig、PayloadRouting、WriteablePayload |
| `mcp/git.ts` | 1334 | git 工具面：push_branch 权限链、NOSHELL 阻断表、symmetric-diff trap |
| `mcp/review.ts` | 1196 | review 提交契约：diff-coverage pre-flight、checkoutSha 锚定、mid-review 新 commit 检测 |
| `mcp/checkout.ts` | 1032 | checkout_pr：initialHead 不变量、beforeSha 增量 range-diff、deepen 计算 |
| `agents/opencode.ts` | 1616 | in-process opencode serve harness（SDK client / event stream / turn accumulator） |
| `agents/claude.ts` | 1425 | Claude Code harness（managed Stop hook + gate server sidecar） |
| `agents/codex.ts` | 998 | Codex harness（auth 物化 + post-hook writeback） |
| `utils/apiKeys.ts` | 914 | 凭证物化到 env：BYOK 错误分类（MISSING_KEY_MARKER / BYOK_SETUP_PATTERN） |
| `utils/runErrorRenderer.ts` | 659 | 失败渲染到 PR comment / job summary（非我们自己的 errorMessage 不发布） |
| `mcp/shell.ts` | 710 | 沙箱 shell：三级 sandbox method 探测 + FS_MOUNTS + TailBuffer |
| `utils/github.ts` | 664 | OIDC 换 installation token、repo context、OctokitWithPlugins |
| `configuration.ts` | ~300 | org/repo 配置字段（privileged 标志：push/shell/env-allowlist/signed-commits/auto-merge/oss） |
| `toolState.ts` | ~450 | LITERAL 事实记录 + RepoToolState（checkout 状态、评论锚点快照） |

## 4. 核心数据结构

| 结构 | 位置 | 语义 |
|------|------|------|
| `RepoSettings` | utils/runContext.ts | 服务端下发 run 级设置（model/effort/modes/hooks 四件套/push/shell/权限类 toggles/learnings/envAllowlist/xrepo 双件） |
| `ResolvedPayload` | utils/payload.ts | 权限解析结果：push（disabled/restricted/enabled）、shell（resolvedShell 严格化）、modelExplicit、event |
| `ToolState` | toolState.ts | LITERAL 事实记录：review/finalSummaryWritten/selectedMode/standaloneCommentId + repos Map |
| `RepoToolState` | toolState.ts | per-checkout：owner/name/dir/access/initialHead（branch|detached 判别）/pushUrl/pushDest/checkoutSha/beforeSha/commentableLinesByFile（双 key 缓存） |
| `TokenRef` | utils/token.ts | gitToken/mcpToken/readToken/ghToken + refreshGitToken 单飞 + dispose 吊销 |
| `PayloadEvent` | external.ts | trigger 判别联合（14 种）+ authorPermission + comment_type 判别 |
| `AgentRunContext` | agents/shared.ts | secretDenyPaths/subagentDeniedTools/stopScript/apiToken/回调三件套 |
| `PostRunIssues` | agents/shared.ts | stopHook/dirtyTree/summaryStale/unsubmittedReview 四类 |
| `SandboxMethod` | mcp/shell.ts | unshare/sudo-unshare/userns-unshare/none |

## 5. 生命周期（单次 run 的完整序列）

见 `04_flow-atlas/flow-atlas.md`（Control Flow 全链）与 `02_engineering-knowledge/ek-graph.md` EK-01（main.ts 检查点流水线）。摘要：

```
entry.ts → normalizeEnv → UNSAFE_OVERRIDES → usage-summary 信号 → resolveRunContextData
→ resolvePayload → initToolState → OIDC stash → commercialRefused 门 → createTempDirectory
→ opencode install + captureBaselineModels → dbSecrets 注入 → credential pool 初始化
→ installCodexAuth/XaiAuth（须在 captureAuthorizedModels 前）→ envAllowlist
→ resolveTokens（4 类 token）→ wipeRunnerLeakSurface → shell probe（#1093）
→ shell!=enabled 删 OIDC env → proxy resolution（BillingError 402/503）
→ resolveBody → gitAuthServer → resolveModel → decideModelAccess（#938）
→ checkConfiguredCredentials → vertex 物化 → resolveAgent（codex opt-in/canary）
→ validateAgentApiKey → setupGit → packageManager provision（#844/#1121）
→ setup hook → computeModes → startMcpHttpServer → subagentDeniedToolNames
→ learnings/summary seed → startInstallation → resolveInstructions
→ findDanglingPromptToolRefs（WARNS）→ opencode plugin 检测
→ 双超时（activity + timeoutPromise + safety-net）→ outputSchema 检查
→ finalizeSuccessRun（postReview→persistSummary→persistLearnings→error-report
   →stranded-comment cleanup→job summary→output marker）
catch → renderRunError → writeRunErrorOutputs → persistRunArtifacts → status check false
finally → killTrackedChildren → usage summary → aggregateUsage PATCH
```

## 6. 测试体系（三层）

| 层 | 目录 | 语义 | 代表 |
|----|------|------|------|
| adhoc（对抗/回归单测） | test/adhoc | 单点攻击面验证 | askpassIntercept（ASKPASS 拦截）、gitExecBypass、gitFlagInjection、nobashcreative、pushRestrictedAdversarial、requirementsTxtAttack |
| agnostic | test/agnostic | 与 agent 无关的机制验证 | gitHooks、gitPerms、missingKeyError、packageJsonScripts、pushDisabled/pushEnabled/pushRestricted、timeout |
| crossagent | test/crossagent | 跨真实 agent 的安全属性 | fsExfil（9 checks）、tokenExfil（passOnTimeout 语义）、gitNativeWrite、nobash、restricted、skillInvokeClaude/Opencode、bedrock/vertexClaude/vertexOpencode、mcpmerge、smoke、codexAuth |

关键对抗测试语义（证据：test/crossagent/fsExfil.ts、tokenExfil.ts）：
- **fsExfil**：agent 视角验证 FS_MOUNTS 三个保护——`/var/lib/pullfrog/sentinel` 对 bash 不可见（tmpfs 覆盖）、`.git/config` 写 EROFS 且 re-read 不变、`.git/hooks/pre-commit` 写失败、`$GITHUB_ENV` baseline 不可见（tmpfs）、`gh` 子进程共享同一 mount ns（`--input` 读 sentinel 失败）。9 个 validation checks。
- **tokenExfil**：`SANDBOX_TEST_TOKEN` 置于进程 env，prompt 鼓励 agent 用一切手段（env/shell/native Read /proc/self/environ /proc/1/environ/nsenter/新 mount /proc）找它；校验 `secret_not_leaked`；`passOnTimeout: true`——持久 agent 跑满预算反而证明沙箱守住（"sandbox held, not broke"）。

## 7. 配置体系

- 顶层 input（action.yml）：prompt/prompt_file/timeout/model/effort/debug/cwd/push/shell/status_checks（DEPRECATED，只增不删）/progress_comments/output_schema/token。
- repo/org 配置（configuration.ts）：model/effort/instructions/status/progress-comments/oss/push/shell/env-allowlist/signed-commits/auto-merge + hooks 四件套（setup/post-checkout/pre-push/stop）+ prompts 各模式追加。
- privileged 字段（push/shell/env-allowlist/signed-commits/auto-merge/oss）——非所有配置都可被 repo 管理员自设。
- 环境变量（运行时）：PULLFROG_TEMP_DIR、PULLFROG_DISABLE_LEARNINGS_REFLECTION、PULLFROG_DISABLE_SECURITY_INSTRUCTIONS（测试用）、GH_TOKEN（外部 override）、BEDROCK_MODEL_ID/VERTEX_MODEL_ID/AZURE_DEPLOYMENT（routing）、LOG_LEVEL。

## 8. 权限与治理机制

| 机制 | 位置 | 语义 |
|------|------|------|
| 分层 token | utils/token.ts | git（可渗漏，contents:read/write+workflows）/ mcp（不可渗漏，contents+pr+issues+checks write, actions read）/ gh（可打印，镜像触发者角色，低于阈值不注册）/ read（xrepo read set） |
| shell 三态 | utils/payload.ts + mcp/shell.ts | disabled（无 shell，git 工具面切断 exec 向量）/ restricted（沙箱 + filterEnv 过滤 env）/ enabled；非协作者至少 restricted；input 只能更严 |
| push 三态 | mcp/git.ts | disabled（只读）/ restricted（禁默认分支 push、禁删分支、禁 tag push）/ enabled（全量）；restricted 下 prepush hook 变更检测 |
| subagent 门控 | agents/subagentToolGates.ts + claudePretoolGate.ts + opencodePlugin.ts | 从 `mutates` 派生禁调用集；agent_id presence 判别；exit 2 阻断 + stderr 给模型 |
| FS deny | agents/nativeFsDenies.ts | native FS 工具（agent 进程内、沙箱外）对 .git 全写 deny + .git/config 读 deny（OpenCode Wildcard 与 Claude glob 双编码） |
| fail-closed 原则 | 多处 | subagentDeniedToolNames 派生空集 → throw；CI 下沙箱不可用 → throw；MCP token 刷新失败不降级 scope（#891） |

## 9. 外部依赖

- 运行时：node24（action）、opencode CLI（`opencode serve` 子进程）、Claude Code（--settings 注册 hook）、Codex CLI（auth 物化）、pnpm（corepack 钉版本）、git（二进制 sha256 指纹化，utils/gitAuth.ts）。
- 服务端（不可见）：PullFrog app（run-context / router / Jev 打分器 / credential store / OSS 资助 / billing），经 run-context JSON 边界 + GitHub App installation token 交互。
- 云侧：OpenRouter（OSS/managed proxy mint，BillingError 402/503）、Bedrock/Vertex/Azure（BYOK routing）、Grok（xai）。

## 10. 考古聚焦（修正后的主线）

1. 权限分层如何把"agent 可写"压到最小必要（token 分四、shell 三态、push 三态、subagent 门控、FS deny、沙箱 mount）。
2. 失败驱动设计：每个 issue 锚点（#844/#860/#862/#891/#906/#938/#964/#1077/#1085/#1093/#1115/#1120/#1121/#1139/#1140/#1146/#1171/#1179）→ 机制。
3. post-run 可见性门控：如何把"无可见产出 = 失败"落地成可执行检查。
4. 双超时 + safety-net + noise 过滤的失联治理。
5. 对抗测试体系如何把安全属性变成 CI 可执行断言。

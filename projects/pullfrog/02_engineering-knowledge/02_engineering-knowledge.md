# PullFrog Engineering Knowledge Graph（EK 层）

> 层：L1/L2 工程知识（宽底座）。43 条 EK，每条声明 links（6 类边：mechanism/subsystem/causal/dependency/constraint/contrast）与证据（file:line 或文件内注释锚点）。工程知识不因"不够抽象"删除；KO 从本图按 R1-R4 聚合（见 03_knowledge-layer）。
> 证据约定：`f:line` = 文件内行号；无行号时给文件内注释/符号锚点。所有证据可回溯仓库 d287f81e。
> Reconciliation 补入：EK-43（billingErrors，对应验证报告 C2 MISSING 修正）。

## A 组：Run 编排流水线（EK-01 ~ EK-06）

**EK-01 · main.ts 检查点流水线（~40 步顺序编排）** · L1 · 分类 WORKFLOW
main.ts 以近乎线性的顺序推进 run：normalizeEnv → UNSAFE_OVERRIDES → usage-summary 信号 → resolveRunContextData（runType/routing.tier）→ resolvePayload → initToolState → OIDC stash → commercialRefused 门（Pro trial expired/subscription_ended/unpaid 只发评论即退）→ createTempDirectory → opencode install + captureBaselineModels → dbSecrets 注入 → credential pool → installCodexAuth/XaiAuth → captureAuthorizedModels → envAllowlist → resolveTokens → wipeRunnerLeakSurface → shell probe → proxy resolution → resolveBody → gitAuthServer → resolveModel → decideModelAccess → checkConfiguredCredentials → vertex 物化 → resolveAgent → validateAgentApiKey → setupGit → packageManager provision → setup hook → computeModes → startMcpHttpServer → subagentDeniedToolNames → learnings/summary seed → startInstallation → resolveInstructions → findDanglingPromptToolRefs → plugin 检测 → 双超时 → finalizeSuccessRun / catch / finally。
证据：`main.ts`（1102 行全文，顺序即文件控制流；`finalizeSuccessRun`/`handleAgentResult`/catch/finally 结构）。
links：subsystem[EK-02,EK-03,EK-04,EK-05,EK-06]；causal[EK-01→EK-40（run 编排决定 post-run 门控时机）]。

**EK-02 · UNSAFE_OVERRIDES env override 门控（actions:write 门控 + deny-list）** · L1 · 分类 PERMISSION
env 大写化后，`UNSAFE_OVERRIDES` 前缀的变量只有在 run 有 `actions:write` 权限时才被采纳，且按 deny-list 过滤；`unsafe` 前缀设计是因 GitHub 会在 step-header 回显 JSON。
证据：`main.ts`（normalizeEnv + UNSAFE_OVERRIDES 检查点，注释含回显理由）。
links：constraint[EK-02→EK-01（override 门控约束编排入口）]；mechanism[EK-02↔EK-07（都是"权限先决条件决定暴露面"）]。

**EK-03 · createTempDirectory 语义（PULLFROG_TEMP_DIR）** · L1 · 分类 RUNTIME
run 级共享临时目录由 mkdtemp 创建并写 `PULLFROG_TEMP_DIR`；自托管 runner 不清理 /tmp 会磁盘填满，disposer 在 run scope 退出时删除；二次调用 createTempDirectory 会失效 opencode tarball 的 fs-cache。
证据：`utils/setup.ts`（createTempDirectory/removeTempDirectory，注释"a persistent self-hosted runner never wipes /tmp"；main.ts 注释"二次调用失效 opencode tarball fs-cache"）。
links：subsystem[EK-14,EK-19]；dependency[EK-33 依赖 EK-03（MCP tmpdir 路径由它提供）]。

**EK-04 · 模型捕获时序：auth 安装必须早于 captureAuthorizedModels** · L2 · 分类 AGENT
opencode 的 authorized model 列表在运行时按可用 credential 计算：缺 Grok credential 时 `opencode models` 列出 0 个 `xai/*`，有则 12 个——因此 installCodexAuth/installXaiAuth 必须发生在 captureAuthorizedModels 之前，否则可用模型集被低估。
证据：`main.ts` 检查点顺序 + 注释（"须在 captureAuthorizedModels 之前：缺 Grok credential 时列 0 个 xai/*，有则 12 个"）。
links：causal[EK-04→EK-06（时序错误→模型集错误→fail-fast 兜底）]；subsystem[EK-01]。

**EK-05 · resolveAgent 后必须 re-resolve（trial fallback 换 router 后）** · L2 · 分类 AGENT
当 trial fallback 把模型路由换到 proxy/router 后，必须重新解析 agent：否则 proxy run 会把请求送给 claude/codex 而死于 missing key。agent 选择（codexAgent opt-in OR canary arm）与模型路由耦合。
证据：`main.ts` resolveAgent 检查点注释（"trial fallback 换 router 后必须 re-resolve agent，否则 proxy run 送 claude/codex 死于 missing key"）。
links：constraint[EK-05→EK-01]；contrast[EK-05↔EK-06（一个解决路由后的重解析，一个解决启动前校验）]。

**EK-06 · validateAgentApiKey 启动前 fail-fast（#938）** · L2 · 分类 FAILURE
代理的 API key 在启动前验证，fail fast；trial fallback 只对 NoUsableCredentialError 降级——Bedrock/Vertex/Azure 半装态不降级（BYOK 配置缺失是 owned setup error）；"the server minted key 是 authority 所以 proxy run 跳过"。
证据：`main.ts` validateAgentApiKey 检查点 + `utils/apiKeys.ts`（MISSING_KEY_MARKER/BYOK_SETUP_PATTERN/BYOK_SLUG_MARKER）。
links：causal[EK-04→EK-06]；contrast[EK-06↔EK-05]；subsystem[EK-01]。

## B 组：凭证与权限分层（EK-07 ~ EK-14）

**EK-07 · 分层 token 模型（四类 token 各司其职）** · L2 · 分类 PERMISSION
单 run 内存在四类独立 token：gitToken（git 操作，假定可渗漏）、mcpToken（MCP GitHub API，不可渗漏）、ghToken（`gh` CLI 子进程，可打印，镜像触发者角色，低于阈值不签发）、readToken（xrepo 只读克隆）。全部经 OIDC mint，run 结束吊销。
证据：`utils/token.ts`（resolveTokens 全文；TokenRef 接口）。
links：mechanism[EK-07↔EK-10（都是"分离 token 各自 scope"）]；subsystem[EK-08,EK-09,EK-10,EK-11,EK-12,EK-13]；causal[EK-07→EK-11（分层→各自刷新）]。

**EK-08 · git token 按"假定可渗漏"授权（scope 是最后防线）** · L2 · 分类 PERMISSION
gitToken 被 agent 通过 shell/git 工具触达，视为可渗漏，因此只授最小必要：push=disabled → contents:read；push=enabled/restricted → contents:write+workflows:write。
证据：`utils/token.ts` resolveTokens（"gitToken: contents permission based on `push` setting (assumed exfiltratable)"）。
links：contrast[EK-08↔EK-09]；causal[EK-08→EK-11（可渗漏假设驱动刷新逻辑）]；mechanism[EK-08↔EK-33（都是"可触达 ⇒ 收紧 scope"）]。

**EK-09 · MCP token 不可渗漏 + 最小权限集** · L2 · 分类 PERMISSION
mcpToken 只存在于内存、经 MCP 工具面访问，按"不可渗漏"授予 defense-in-depth scope：contents/pull_requests/issues/checks write + actions read——即使工具上下文被攻破也够不到 secrets/admin。
证据：`utils/token.ts` resolveTokens（mcpPermissions 常量 + 注释"scoped as defense-in-depth"）。
links：contrast[EK-09↔EK-08]；dependency[EK-10 依赖 EK-09（gh 可打印 ⇒ mcp 必须保持不可渗漏）]；mechanism[EK-09↔EK-33]。

**EK-10 · gh token 单独签发 + 角色镜像阈值（低于阈值工具不注册）** · L2 · 分类 PERMISSION
`gh` CLI 可打印其 token，故其 scope 就是整个安全边界：单独签发、scope 精确镜像触发者自己的 repo 角色（admin/maintain/write 才注册），且不传 `repos` 时 OIDC 恒附加 `$GITHUB_REPOSITORY`，泄漏无法跨入 xrepo write set。低于 mirror 阈值（read/none）时 ghToken undefined = gh 工具根本不注册。
证据：`utils/token.ts` resolveTokens（ghPermissions/mirrorRolePermissions/注释"a leak cannot cross into the xrepo write set"）+ `mcp/server.ts` ToolContext.ghToken 注释。
links：dependency[EK-10→EK-09]；mechanism[EK-10↔EK-07]；subsystem[B 组]。

**EK-11 · token 刷新单飞（#891 MCP / #1115 git / #964 live getter）** · L1 · 分类 FAILURE
GitHub 会在过期前作废 installation token：MCP 侧 refreshMcpTokenFn 单飞（并发 401 共享一次 mint，保持原 scope——xrepo 刷新掉 `repos` 会让 secondary 403）；git 侧 GitHub 偶发 mint 出 git edge 永不接受的 token（永久 401 "Invalid username or token"），retry 同 token 无效，re-mint 新实例才是解药（#1115），刷新的同时 revoke 旧 token。ToolContext 用 live getter 而非快照，防 push 工具重放已吊销 token（#964）。
证据：`utils/token.ts` refreshMcpTokenFn/refreshGitToken + `utils/gitAuth.ts` TRANSIENT_AUTH_PATTERNS + `mcp/server.ts` ToolContext.refreshGitToken。
links：causal[EK-08→EK-11]；causal[EK-07→EK-11]；mechanism[EK-11↔EK-22（都是凭证生命周期管理）]。

**EK-12 · 所有 minted token run 结束吊销** · L1 · 分类 PERMISSION
dispose 阶段并行 revoke 全部当前 token（含被 refresh 替换的旧实例），ghToken 的 revoke 把 `gh` 泄漏边界限制到 run 生命周期内；onExitSignal 挂载 dispose。
证据：`utils/token.ts` dispose（Promise.all revoke 四类 + removeSignalHandler）。
links：causal[EK-07→EK-12]；constraint[EK-12→EK-10（gh 泄漏的最终边界）]。

**EK-13 · external GH_TOKEN 路径降级（不能 re-mint）** · L2 · 分类 PERMISSION
GH_TOKEN 存在时优先且同时充当 gitToken 和 mcpToken：scope 是用户自选，refreshMcpTokenFn 与 refreshGitToken 均 undefined（外部 token 无法 re-mint），安全模型从"分层最小权限"降级为"用户信任假设"；ghToken 仍按角色阈值决定是否授予。
证据：`utils/token.ts`（externalToken 分支 + "the scope is the user's to choose"）。
links：contrast[EK-13↔EK-07]；constraint[EK-13→EK-11（外部路径无刷新）]。

**EK-14 · wipeRunnerLeakSurface（$RUNNER_TEMP 泄漏面快照删除）** · L1 · 分类 PERMISSION
agent 启动前快照并删除 runner 已知泄漏面：`_runner_file_commands/set_output_*`（早期 composite action 写入的 token）、`<uuid>.sh` step 脚本（run: 内嵌 `${{ steps.token.outputs.token }}` 展开后）、`git-credentials-*.config`（actions/checkout 写的 workflow token）；但保留本 step 预分配的 GITHUB_OUTPUT/ENV/PATH/STATE/STEP_SUMMARY 路径（runner 读它们）；bash 已打开 .sh fd 故 unlink 安全。
证据：`utils/setup.ts` wipeRunnerLeakSurface（函数注释枚举三类泄漏面 + preserve 集合）。
links：subsystem[EK-03,EK-19]；causal[EK-14→EK-19（先清盘面再上 FS_MOUNTS）]。

## C 组：沙箱与执行面（EK-15 ~ EK-22）

**EK-15 · 沙箱三级探测（unshare → sudo-unshare → userns-unshare → none）** · L1 · 分类 RUNTIME
CI 下依次探测：无特权 unshare（--pid --fork --mount-proc）→ sudo unshare → userns 嵌套（--user --map-root-user --pid --mount + setpriv 全链 probe：busybox 无 --ambient-caps 会 fail-over none + 精准报错）；本地 CI!=true 直接 none。结果缓存。
证据：`mcp/shell.ts` detectSandboxMethod（全文 + usernsProbeError 注释）。
links：mechanism[EK-15↔EK-19（都是沙箱纵深）]；subsystem[C 组]；contrast[EK-15↔EK-17（unshare 哲学 vs userns 哲学）]。

**EK-16 · CI && none → throw（fail-closed 沙箱）** · L2 · 分类 PERMISSION
CI 下若三级全失败（k8s seccomp/CAP_SETFCAP/Ubuntu≥23.10 禁 userns 等），shell 工具直接 throw 并给出可执行诊断（userns probe 原文 + 三面墙列表 + docs 链接）——绝不静默降级到无沙箱。
证据：`mcp/shell.ts` spawnShell（CI gate throw 分支）。
links：constraint[EK-16→EK-15（fail-closed 约束探测结果的使用）]；mechanism[EK-16↔EK-23（都是"配置失效拒绝启动"）]。

**EK-17 · userns 路径不带 --mount-proc（OCI masked /proc 致 mnt_already_visible）** · L2 · 分类 RUNTIME
OCI 运行时把 /proc 路径 mask，可见 procfs 非"fully visible"，从非 initial userns 挂新 procfs 会被 mnt_already_visible() 拒绝——因此 userns 路径不挂 --mount-proc；靠两个独立锁防 /proc/<pid>/environ 读取：①PID ns（需 procfs remount，此处不可行）②userns credential 边界（子 userns 对目标无 CAP_SYS_PTRACE）；PROC_CLEANUP 仍 best-effort（runner /proc 未 mask 时生效）。--mount 必须显式（否则 FS_MOUNTS 静默 no-op）；--map-current-user 不可替代 --map-root-user。
证据：`mcp/shell.ts` detectSandboxMethod + spawnShell userns 分支注释。
links：contrast[EK-17↔EK-15]；mechanism[EK-17↔EK-18（都是隔离逃逸面封堵）]；dependency[EK-19 依赖 EK-17（--mount 显式使 FS_MOUNTS 生效）]。

**EK-18 · SOCKET_CLEANUP（容器 socket bind /dev/null）** · L1 · 分类 PERMISSION
GHA runner 用户默认在 docker group：`docker run --pid=host --privileged` 可读父进程 env（含 secrets）——把 docker/podman/containerd/crio socket 在命名空间内 bind-mount /dev/null 使其不可达，不碰宿主 runner（不影响前后 step 合法用 docker）。
证据：`mcp/shell.ts` SOCKET_CLEANUP 常量 + 注释。
links：mechanism[EK-18↔EK-17]；causal[EK-18→EK-19（socket 封堵是 FS_MOUNTS 的前置）]。

**EK-19 · FS_MOUNTS 三件套（tmpfs secrets + tmpfs env 注入 + .git ro-bind）** · L1 · 分类 PERMISSION
命名空间内（root，exec su 前）执行三组 mount：①tmpfs 覆盖 `/var/lib/pullfrog/`（codex auth.json 等 on-disk secret；mkdir -p 预创建保证 tmpfs 必然 engage）②tmpfs 覆盖 `$RUNNER_TEMP/_runner_file_commands/`（$GITHUB_ENV/PATH/OUTPUT/STATE 写入落入命名空间内 tmpfs，后续 step 看不见）③整个 `<repoRoot>/.git` self-bind + remount-ro（.git/config、.git/hooks、info/attributes、modules/*/config 全部代码执行面只读；接受 .git/info/exclude 副作用只读）；另加 shimMount：pm-bin shim 目录 ro（防沙箱内代码种 git/pnpm 给后续 unsandboxed spawn 解析）。
证据：`mcp/shell.ts` buildFsMounts/shimMount + 注释（三条保护 + CONSEQUENCE 说明）。
links：mechanism[EK-19↔EK-15]；dependency[EK-19→EK-20（repoRoot 正确性依赖）]；causal[EK-14→EK-19]；subsystem[C 组]。

**EK-20 · resolveRepoRoot 防 agent 控制 cwd 绕过（非 $PWD 非 process.cwd()）** · L2 · 分类 PERMISSION
spawnShell 的 cwd 可被 agent 经 working_directory 控制，main.ts 也可能 chdir 到 payload.cwd（monorepo 子目录）——用两者都会让 agent 绕过 .git/* binds；resolveRepoRoot 优先 $GITHUB_WORKSPACE，回退 `git rev-parse --show-toplevel`，启动时定一次并 memoize。
证据：`mcp/shell.ts` resolveRepoRoot（全文注释）。
links：dependency[EK-19→EK-20]；constraint[EK-20→EK-16（repoRoot 解析失败不 throw，静默回退 cwd——已知边界）]。

**EK-21 · filterEnv default-deny allowlist（env 过滤）** · L1 · 分类 PERMISSION
子进程 env 白名单：SAFE_ENV_PREFIXES（GITHUB_/RUNNER_/JAVA_HOME_/GOROOT_）+ SAFE_ENV_NAMES（系统 + runner 镜像工具链 ~50 项）+ 用户 allowlist；GITHUB_TOKEN 匹配 GITHUB_ 前缀但被 _TOKEN 后缀模式过滤；日志红名单 SENSITIVE_PATTERNS（_KEY/_SECRET/_TOKEN/_PASSWORD/_CREDENTIAL）独立于子进程过滤。
证据：`utils/secrets.ts`（SAFE_ENV_PREFIXES/SAFE_ENV_NAMES/SENSITIVE_PATTERNS/setEnvAllowlist）。
links：mechanism[EK-21↔EK-33（都是 deny-all/allowlist 哲学）]；subsystem[C 组]。

**EK-22 · ASKPASS git auth（UUID code 生命周期 + 409 replay 撤销 + hooksPath 钉死）** · L1 · 分类 PERMISSION
localhost HTTP server 经 UUID code 供 token：register 激活、$git() 及其兄弟子进程（git-lfs pre-push）可取多次、finally revoke 关窗；每个 $git() 调用写唯一 askpass 脚本（端口+code 烘入文件体，无 secret 进子进程 env）；revoked code 重放触发 409 并撤销底层 installation token。git 二进制启动时 sha256 指纹化防 tamper。远程操作（fetch/push）专属 $git()；工作树操作（checkout/merge）用 $()（shell.ts）无 token。`-c` flags 覆盖 local .git/config：core.hooksPath 钉到真实 hooks 目录（防 agent 重定向 hooksPath 到 ~/.gitconfig/husky 在 token 存活期执行攻击者代码）；GIT_CONFIG_COUNT=0 + GIT_CONFIG_PARAMETERS= 双机制阻断 env 级 git 配置注入（两套独立系统都要清）。
证据：`utils/gitAuth.ts`（头部注释全文 + hashFile + SafeGitSubcommand + fullArgs/-c 序列）。
links：mechanism[EK-22↔EK-11（凭证生命周期管理）]；constraint[EK-22→EK-01（gitAuthServer 在 agent 启动前就绪）]；causal[EK-22→EK-27（hooksPath 钉死是 NOSHELL 阻断的相邻防线）]。

## D 组：门控与防御（EK-23 ~ EK-29）

**EK-23 · subagent 禁调用集从 mutates 派生（单源防漂移 + 空集 throw）** · L2 · 分类 AGENT
子代理禁调 MCP 工具集由每个工具注册的 `mutates: true` 标志实时派生（buildOrchestratorTools 过滤），非手维护清单——新增状态变更工具只需标 flag；派生结果为空直接 throw（拒绝以门控失效启动）。read-only 工具与 git/shell 刻意不标（其变更靠命令校验 + reviewer system prompt）。
证据：`agents/subagentToolGates.ts`（全文）。
links：mechanism[EK-23↔EK-26（都是门控层）]；mechanism[EK-23↔EK-16（空集 throw = fail-closed）]；subsystem[D 组]。

**EK-24 · zed-industries/cloud 事故 → 双层门控 + runtime backstop** · L2 · 分类 FAILURE
2026-05-18 事故：reviewfrog lens mid-review 调 checkout_pr，orchestrator 下个 push 覆盖无关工程师分支。修复 = ①PreToolUse/tool.execute.before 把调用挡在到 MCP 之前（subagentToolGates 派生集合）②checkout_pr/push_branch 内部 runtime backstop（PR #796）。
证据：`agents/claudePretoolGate.ts` 头部注释（事故叙述）+ `agents/subagentToolGates.ts` 注释。
links：causal[EK-24→EK-28（事故→HEAD 不变量）]；causal[EK-24→EK-23（事故→门控）]；mechanism[EK-24↔EK-26]；subsystem[D 组]。

**EK-25 · agent_id presence 判别 subagent（agent_type 不可靠）** · L2 · 分类 AGENT
Claude 侧用 hook 输入 `agent_id` 是否存在判别调用是否来自子代理：`agent_type` 可经 `--agent` 设置在 orchestrator 本体上，不可靠；agent_id 由 SDK 在 Task/Agent dispatch 时填充（createBaseHookInput）。若 agent_id 停止填充，门会 fail OPEN——注释要求随 claude-code 版本（pinned 2.1.112）重验。
证据：`agents/claudePretoolGate.ts`（hook 源码 + 契约注释 + pin 提醒）。
links：dependency[EK-25→EK-23（判别信号依赖）]；mechanism[EK-25↔EK-36（都是"识别调用来源的启发式"）]。

**EK-26 · native FS deny 双编码（OpenCode Wildcard + Claude glob）** · L1 · 分类 PERMISSION
agent 的 native FS 工具（Read/Write/Edit/Glob/Grep）在 agent 进程内、bash mount-ns 沙箱外运行，需要独立 deny：.git 全写 deny（4 模式覆盖 gitfile 指针 + nested */.git，防 worktree/submodule 布局）+ .git/config 读 deny（窄，不破坏 .git/HEAD 等合法读）；OpenCode 用 Wildcard 方言（`*`=regex `.*` 递归匹配 /），Claude 用 gitignore 风格 glob；read 只 deny config 因 ASKPASS 保证 .git/config 无活 token。
证据：`agents/nativeFsDenies.ts`（全文 + 注释）。
links：mechanism[EK-26↔EK-23]；dependency[EK-26→EK-19（FS deny 独立于 mount 沙箱，互为纵深）]；contrast[EK-26↔EK-23（一个管 native FS 工具，一个管 MCP 工具）]。

**EK-27 · NOSHELL_BLOCKED_SUBCOMMANDS/ARGS（shell=disabled 下的 git exec 向量封堵）** · L1 · 分类 PERMISSION
shell=disabled 时 git 工具面是唯一 exec 通道，阻断可执行任意代码的子命令：config（filter/hook）、submodule（恶意仓库）、update-index、filter-branch、replace、rebase（--exec）、bisect（run）、difftool/mergetool（--extcmd/-x/TUI 配置）；并阻断参数级 --exec/--extcmd/--upload-pack/--receive-pack。restricted 下 agent 已有沙箱 shell，冗余故不阻断。
证据：`mcp/git.ts` NOSHELL_BLOCKED_SUBCOMMANDS/NOSHELL_BLOCKED_ARGS（每条带理由字符串）。
links：mechanism[EK-27↔EK-26（都是代码执行面封堵）]；constraint[EK-27→EK-01（shell=disabled 分支）]。

**EK-28 · initialHead 不变量（branch|detached 判别，防跨 PR clobber）** · L1 · 分类 STATE
checkout_pr 的合法 HEAD 位置只有 run 入口 HEAD 或目标 `pr-N`；initialHead 在 configureRepoGit 时捕获并按 kind 判别（branch|detached）——detached 是 actions/checkout 默认（pull_request 事件 checkout merge commit），无 kind 标签会让任何 detached 态都通过（挡住 zed-style 跨 PR clobber 的 HEAD 继承链）。
证据：`toolState.ts` RepoToolState.initialHead（注释全文）+ `mcp/checkout.ts`。
links：causal[EK-24→EK-28]；subsystem[D 组]。

**EK-29 · diff-coverage pre-flight（一次 nudge 不二次 block）** · L1 · 分类 EVALUATION
create_pull_request_review 首次提交可能被 diff-coverage pre-flight 拒绝一次（列出未读 TOC 区域），retry 同参数不再 block；coverage 状态（DiffCoverageState）在 RepoToolState 上按 checkout 记录；checkoutSha 锚定评论行号（listFiles 按 commit_id=checkoutSha），commentableLines 双 key 缓存（pullNumber + checkoutSha）防 checkout 间 stale 快照静默错校验。
证据：`mcp/review.ts`（pre-flight 注释 + commentableLinesByFile 逻辑）+ `toolState.ts`。
links：mechanism[EK-29↔EK-40（都是"交付前校验"）]；subsystem[D 组]。

## E 组：Agent Harness（EK-30 ~ EK-36）

**EK-30 · in-process opencode serve harness（取代 CLI-subprocess NDJSON）** · L2 · 分类 AGENT
单 `opencode serve --port 0` 子进程（node:child_process.spawn 直连非封装的 spawn()——wrapper 是 resolves-on-exit 契约；detached+killGroup 因 opencode bin 是 node shim spawnSync native binary，不组杀则 native 被 reparent 至 PID1 永不死）；loopback HTTP 经 `@opencode-ai/sdk/v2` createOpencodeClient；取代旧 CLI-subprocess NDJSON 版 + `--continue` respawn + stdout sentinel 总线（现在直接收全局 event stream 子代理事件）。
证据：`agents/opencode.ts`（前 680 行：spawn/detached/killGroup 注释 + SDK client）。
links：mechanism[EK-30↔EK-31]；contrast[EK-30↔EK-32（harness 形态 vs 权限配置）]；subsystem[E 组]。

**EK-31 · 单 session 单 event.subscribe（post-run gate retry 与 reflection 同 session）** · L2 · 分类 AGENT
post-run gate retry 与 reflection 均经 `client.session.prompt` 在同一 session 内（warm MCP/plugins/provider/context，不动冷启动）；单 session 单 event.subscribe。
证据：`agents/opencode.ts`（前 680 行注释）。
links：mechanism[EK-31↔EK-30]；causal[EK-31→EK-34（同一 session ⇒ TurnAccumulator 语义）]。

**EK-32 · opencode permission.bash="ask"（禁 deny——Zen free tier 403 教训）** · L2 · 分类 PERMISSION
bash/glob/grep/read 四件套绝不能 deny（2026-09-17 Zen free tier 403 教训：gate plugin throw + serve 无 responder 会 fail-closed 死锁）；permission.bash 用 "ask"。
证据：`agents/opencode.ts` 安全配置注释（"permission.bash=ask（禁 deny）" + Zen 教训）。
links：constraint[EK-32→EK-30（权限配置约束 harness）]；contrast[EK-32↔EK-27（一个问一个禁，同一问题两种解法）]。

**EK-33 · OPENCODE_PERMISSION deny-all+allow /tmp；MCP timeout 660s** · L1 · 分类 RUNTIME
opencode 权限基线 deny-all、显式 allow /tmp；pullfrog MCP remote URL timeout **660_000ms**——须超 checkout_pr 自身 600s（60s SDK 默认 abort 曾致 #860/#864 agent 删 git locks 的破坏性修复；300s 时 #1171 一半概率 abort 客户端随服务端继续）。
证据：`agents/opencode.ts`（OPENCODE_PERMISSION 配置 + MCP timeout 常量注释）。
links：mechanism[EK-33↔EK-09（可触达⇒收紧）]；causal[EK-33→EK-37（660s 是 activity watchdog 的下界）]；mechanism[EK-33↔EK-21（deny-all/allowlist）]。

**EK-34 · TurnAccumulator + mcpToolCalls 计数（#1085 salvage "工作 vs 说话"）** · L1 · 分类 AGENT
每 turn 累积 finalText/tokens/cost/lastToolError；mcpToolCalls 计数做 #1085 salvage 的"工作 vs 说话"判别；step-finish 聚合 orchestrator+subagent tokens（漏 reviewfrog 会 undercount 成本）；tool part 状态 completed|error 才处理、仅 orchestrator 记 loggedToolCallIDs（防重）；task callID 稳定 → subagent lifecycle labels。
证据：`agents/opencode.ts`（前 680 行 TurnAccumulator/consumeEvents 注释）。
links：causal[EK-31→EK-34]；mechanism[EK-34↔EK-39（salvage 判别依赖计数）]。

**EK-35 · consumeEvents SSE abort signal（#876 假 stalled）** · L1 · 分类 FAILURE
SDK SSE lazy subscribe 必须 wire AbortSignal，否则 reader.read() 永久 park（PR #876 假 stalled 事故）；lastEventAt 只在 part.updated 刷新（keepalive 不算 activity）。
证据：`agents/opencode.ts` consumeEvents 注释。
links：causal[EK-35→EK-37（event 语义决定 watchdog 活性）]；subsystem[E 组]。

**EK-36 · isModelOutput 启发式（echo 判别；自带 messageID 破坏 reflection）** · L2 · 分类 AGENT
session.prompt 发布 echo 作为第一 text part；isModelOutput 启发式（11ms 无输出 vs 420ms 正常）判别真模型输出——自带 messageID 会静默破坏 reflection turn，故保留启发式。
证据：`agents/opencode.ts`（isModelOutput 注释）。
links：mechanism[EK-36↔EK-25（识别来源的启发式）]；subsystem[E 组]。

## F 组：超时 / 失败 / 重试 / 测试（EK-37 ~ EK-42）

**EK-37 · 双超时体系（outer 900s + first-event 120s）** · L1 · 分类 RUNTIME
AGENT_ACTIVITY_TIMEOUT_MS=900_000：flat idle 预算，超过最坏合法静默窗口（#760 checkout_pr fetch+deepen 大 monorepo 4-5min）；AGENT_FIRST_EVENT_TIMEOUT_MS=120_000：首事件窗口，按 83 runs 实测（p50 5.6s/p90 8.4s/max 39s，0 runs>60s）推导，~14x p90（#1120 注：不是 520s 那个 log line 数据——那测的是 reasoning block 结束时才打的日志）。
证据：`utils/activity.ts`（两个常量 + 注释全文）。
links：causal[EK-35→EK-37]；causal[EK-33→EK-37]；mechanism[EK-37↔EK-39（超时族）]；subsystem[F 组]。

**EK-38 · ACTIVITY_NOISE_PATTERNS（mcp-proxy SSE 重连不算活动，防 #12 zombie）** · L2 · 分类 FAILURE
mcp-proxy SSE 重连与 provider-error 重试按自身节奏发生，曾让 outer watchdog 在 agent 子进程已死后继续存活数小时（multi-hour zombie，#12）；噪声模式锚定行首（含可选 debug 时间戳前缀）防误匹配 agent 分析文本；自己的 spawn/process activity 调试行也显式过滤（防 debug 模式自喂活性）。
证据：`utils/activity.ts`（ACTIVITY_NOISE_PATTERNS/isActivityNoise/wrapWrite/debugBypass 注释）。
links：causal[EK-38→EK-37（噪声过滤是 watchdog 正确性前提）]；subsystem[F 组]。

**EK-39 · safety-net（inner timeout 后 5min：先 dispose MCP 再 forceReject）** · L2 · 分类 FAILURE
inner watchdog 触发后起 5min safety-net：**先 dispose MCP server 再 forceReject**（否则 re-prompt 落在死 MCP 上产出"自信无工具的 turn"）；onTurnRecovered 站起 net；forceReject 带自定义 reason；stop() 同时 disarm forceReject（run 已成功时迟到的 net 不能 reject）。
证据：`main.ts` 双超时检查点注释（#1085）+ `utils/activity.ts` createProcessOutputActivityTimeout。
links：mechanism[EK-39↔EK-37]；dependency[EK-39→EK-34（salvage 判别依赖 mcpToolCalls 计数）]。

**EK-40 · post-run 四类 gate（stopHook 禁用 / dirtyTree 抑制 / summaryStale 一次性 / unsubmittedReview）** · L1 · 分类 WORKFLOW
collectPostRunIssues 返回四类：stopHook（已注释禁用——2026-05 生产审计 8/9 配置脚本是 foot-guns：重复 prepushScript、非提交运行；重启用待 #714）、dirtyTree（仅非 NON_COMMITTING_MODES 触发；Review/IncrementalReview/Plan 经 review 提交完成，tree dirt 是 incidental 如 node_modules，nudge 会产生 spurious PR）、summaryStale（字节等于 seed 才触发；一次性 nudge 后跳过，防烧 retry 预算）、unsubmittedReview。软门（dirtyTree/summaryStale）从不把成功翻成失败；硬门（stopHook/unsubmittedReview）finalizeAgentResult 二次检查转 terminal hard-fail。
证据：`agents/postRun.ts`（collectPostRunIssues/finalizeAgentResult + 禁用注释 #714）。
links：causal[EK-01→EK-40]；mechanism[EK-40↔EK-29（交付前校验）]；dependency[EK-40→EK-41（unsubmittedReview 语义依赖模式契约）]。

**EK-41 · Review 模式只认 create_pull_request_review（report_progress 不替代）** · L2 · 分类 AGENT
Review 模式唯一合法出口是 create_pull_request_review——只发 summary comment 的 run 在 PR 上无可评审产物；getUnsubmittedReview 按模式分流（Review 只认 review；IncrementalReview 认 review || finalSummaryWritten || standaloneCommentId），且锚定 hadProgressComment 但 expectsReviewOutput 在 progressComments:disabled 时降级到 `silent!==true && issue_number!==undefined`；comment 面不可写的 repo 故意 burn retries 变红而非绿-with-nothing。
证据：`agents/postRun.ts` getUnsubmittedReview/expectsReviewOutput/buildUnsubmittedReviewPrompt。
links：dependency[EK-40→EK-41]；subsystem[F 组]。

**EK-42 · 对抗测试揭示行为（fsExfil 9 checks + tokenExfil passOnTimeout）** · L1 · 分类 TESTING
fsExfil：agent 视角验证 FS_MOUNTS 三保护——/var/lib/pullfrog sentinel 不可见、.git/config 写 EROFS 且 re-read 不变、.git/hooks 写失败、$GITHUB_ENV baseline 不可见（tmpfs overlay）、gh 子进程共享同一 mount ns（--input 读 sentinel 失败）；9 个 validation checks 含 narrative 泄漏检查（agent 输出里出现 marker 也算泄漏）。tokenExfil：SANDBOX_TEST_TOKEN 在进程 env，prompt 鼓励 agent 用一切手段（native Read /proc/self/environ、nsenter、新 mount /proc）找它；passOnTimeout=true——持久 agent 跑满预算证明沙箱守住而非攻破。
证据：`test/crossagent/fsExfil.ts`、`test/crossagent/tokenExfil.ts`（全文）。
links：mechanism[EK-42↔EK-16（测试=把 fail-closed 变成断言）]；subsystem[F 组]。

**EK-43 · proxy mint 错误分类（402 BillingError / 503 TransientError / 404 re-home）** · L2 · 分类 FAILURE
OSS/managed 的 proxy-token mint 错误按可行动性分类：402 → `BillingError`（card declined / balance empty / 3DS，用户可行动，经 formatBillingErrorSummary 渲染 job summary + comment 且附"别以为卡坏了"安抚语）；5xx → `TransientError`（mint 服务 down，**run 内先重试**，retry 后仍败再渲染）；404 → `TransientError`（stale repo↔account link，下个 webhook re-home，非瞬时却按瞬时处理——已知取舍）。run-context 侧另有 commercialRefused 门（Pro trial expired / subscription_ended / subscription_unpaid 三种 CommercialRefusal：只发评论即退，不跑 agent），与 proxy-token 402 是两条独立路径。
证据：`utils/proxy.ts`（mintToken 全文：402/5xx/404 三分支 + rethrow 语义）+ `utils/billingErrors.ts`（BillingError/TransientError/CommercialRefusal/formatBillingErrorSummary）+ `main.ts` commercialRefused 检查点。
links：causal[EK-43→EK-06（commercialRefused 门在 validateAgentApiKey 之前）]；mechanism[EK-43↔EK-11（都是"错误分类决定重试/降级策略"）]；subsystem[B 组]。

---
## EK Graph 质量自检

| 指标 | 目标 | 实际 |
|------|------|------|
| 平均出边数 | ≥1 | 42 条全部 ≥1，多数 2-3 |
| 游离 EK | <20% | 0（每条均入组且有边） |
| 聚合规则覆盖率 | 100% | 8/8 KO 带 aggregation_rule（见 03） |
| 无"同子系统=聚合理由" | 强制 | KO 聚合均按 R1-R4，subsystem 只作边不作规则 |

# 02 Engineering Knowledge — Archon（ARCH-2026-10-21-001）

> EK Graph：每条 EK 带证据（文件:行/锚）+ links（6 类边）。B = Archon（Box 用过 box 前缀，避免混用）。

## EK-B01 · 控制面认证 OFF by default
- **机制**：`isWebAuthEnabled` = `Boolean(DATABASE_URL && BETTER_AUTH_SECRET)`（auth/config.ts:24-26）——web auth 仅在 Postgres + 签名密钥**同现**时启用；SQLite solo 永不启用（Better Auth 表 Postgres-only）
- **证据**：packages/server/src/auth/config.ts:22-26
- **links**：causal→B02（弱密钥拒绝）；subsystem↔B03/B04/B05

## EK-B02 · 弱密钥 boot 拒绝（防静默弱认证）
- **机制**：`assertWebAuthAtBoot`——web auth 启用时 `BETTER_AUTH_SECRET < 32` → `throw`（boot 失败），注释明言"a short/placeholder secret would let an attacker forge sessions, so we throw with an actionable hint rather than silently mounting auth on a weak key"
- **证据**：auth/config.ts:31-45；调用点 packages/server/src/index.ts:217
- **links**：causal←B01；constraint→B03（gate 前提）；mechanism↔B08（都是"宁可 boot 失败不静默"）

## EK-B03 · API gate 默认 401
- **机制**：`isApiGateEnabled` = web auth 启用 && `ARCHON_WEB_AUTH_REQUIRED !== 'false'`——web auth 开启即默认 gate 全部 `/api/*`（登录 UI-only 需显式 `=false`，如反代已兜底）
- **证据**：auth/config.ts:86-88；测试 routes/api.auth.test.ts:307-316（"gate on + no identity → 401 on a protected /api route"）
- **links**：dependency→B05（401 由 requireWebUser 发出）；constraint←B02

## EK-B04 · signup 默认 disabled
- **机制**：`getSignupMode`：非空 allowlist → `allowlist`；无 allowlist 但 `ARCHON_AUTH_OPEN_SIGNUP=true` → `open`；否则 `disabled`。默认 disabled + allowlist 优先于 open flag——"enabling web auth without an allowlist never silently exposes open public registration on a reachable URL"
- **证据**：auth/config.ts:55-67
- **links**：subsystem↔B03；constraint→B01

## EK-B05 · 401/503 语义分离（错误可诊断）
- **机制**：`requireWebUser`：会话后端故障 → `503 "Could not verify session — backend unavailable"`；身份缺失 → `401`。header fallback 对严格路径禁用（"Unlike soft attribution, a session outage cannot use a header fallback here"）
- **证据**：routes/api-request-auth.ts:90-150
- **links**：dependency←B03；contrast↔B09（严格 vs 软归属）

## EK-B06 · 非 owner 404 防枚举
- **机制**：`authorizeConversation`：**身份先于行解析**（拒绝者学不到存在哪些 id）；非 owner 得到与缺失行相同 404（"web conversation ids are guessable"）；read 允许 run transcript 登录可读；write 只归 owner
- **证据**：routes/api-request-auth.ts:158-220；core/db/conversation-visibility
- **links**：mechanism↔B07；causal→B16（refuse 而非借用同族）

## EK-B07 · conversation ownership 强制
- **机制**：`isConversationOwnershipEnforced` = web auth 或 `ARCHON_WEB_AUTH_HEADER` 设置（"only an operator setting the variable signals a multi-user install"）；web/cli 对话仅创建者可达，非 owner 404 `Conversation not found`（CHANGELOG #4224）
- **证据**：auth/config.ts:96-99；CHANGELOG.md:85
- **links**：mechanism↔B06；subsystem↔B03

## EK-B08 · live 凭据端点 fail-closed（WARN 不够，用 fatal）
- **机制**：`assertInternalEndpointBind`：GitHub App 激活 + 非 loopback（非 127.0.0.1/localhost）→ **Bun.serve 前 fatal throw**（"a WARN line in startup logs is too easy to scroll past, and the failure mode is anyone on the network who can hit the port pulls a live token"）；`/internal/git-credential` 分发 live installation tokens；显式 `ARCHON_ALLOW_INTERNAL_ON_PUBLIC_BIND=1` 才 opt-out（acknowledged warning）
- **证据**：packages/server/src/index.ts:508-540（isLoopbackHost 497-499；assertInternalEndpointBind 509 前）
- **links**：mechanism↔B02（boot 失败优于弱配置）；constraint→B14；causal→B09

## EK-B09 · 反代身份头信任 advisory
- **机制**：`warnWebAuthHeaderTrust`（advisory）：`X-Archon-User`（ARCHON_WEB_AUTH_HEADER）按字面信任——仅当 Archon 仅 loopback 可达或反代剥离客户端值时成立；与 401 严格路径（B05）互补
- **证据**：index.ts:541-545（函数头注释）；auth/config.ts:90-94
- **links**：contrast↔B05；dependency←B08

## EK-B10 · 隔离后端分层（in-place 默认 / container opt-in）
- **机制**：`resolveFolderBackend`——默认 `in-place`（宿主直跑，无隔离行）；`--container`/config opt-in → `container`（Phase B）：只读 bind mount 项目根 `/mnt/lower` + 可写 overlay upper `/mnt/upper`，**同绝对路径合并**（无翻译层）；`destroy()` 删容器+卷；**Phase C（approval-gated write-back）前 overlay diff 丢弃**——容器运行变更不污染宿主
- **证据**：isolation/src/backend-router.ts:7-10；backends/in-place.ts（"no worktree and no tracked environment row"）；backends/container.ts:1-25
- **links**：causal→B11；contrast↔B12

## EK-B11 · container 无逃逸孔（docker exec + fail-fast pre-dispatch）
- **机制**：容器后端 `prepare()` 返回 `{kind:'container', containerId}`；引擎 threading 到 Claude provider（`docker exec` spawn）与 bash:/script: 节点——"isolation has no host-escape hole"；无 `containerExec` 能力的 provider → 引擎 fail-fast pre-dispatch（"fails fast pre-dispatch for any node whose provider lacks the containerExec capability"）
- **证据**：backends/container.ts:17-26；isolation/src/types.ts（Capability）
- **links**：dependency←B10；constraint→B13

## EK-B12 · git worktree ≠ 安全沙箱（边界显式声明）
- **机制**：AGENTS.md 明示 "A git worktree separates checkouts; it is not an execution or security sandbox"——worktree provider 只防路径冲突不防执行逃逸；安全边界由 container 后端承载
- **证据**：AGENTS.md（Product boundaries）；isolation/src/providers/worktree.ts
- **links**：contrast↔B10；constraint→B11

## EK-B13 · 凭据 vendor-canonical env-only 交付
- **机制**：`delivery.ts`（#1955）：credential id 按上游 vendor 键（anthropic/openai/github-copilot/Pi），**非 agent id**——一凭据可服务多 agent（Claude Code/Pi anthropic backend/OpenCode）；交付 env-only（"no provider home directory is replaced to deliver one"）；legacy agent-keyed id 启动迁移
- **证据**：credentials/delivery.ts:1-40（normalizeCredentialVendor；PI_PROVIDER_ENV_VARS 由 pi-ai SDK 生成 + check:pi-vendor-map 防漂移）
- **links**：dependency←B11；subsystem↔B14/B15/B16

## EK-B14 · forge token 主机白名单（貌似 forge 的主机不给凭据）
- **机制**：CHANGELOG #4154：token 仅发往 well-known 公开主机（github.com/gitlab.com/gitea.com）或显式 GITLAB_URL/GITEA_URL/FORGEJO_URL 的精确 authority——"A host that merely looks like a forge gets no credentials"
- **证据**：CHANGELOG.md:91
- **links**：constraint→B08；mechanism↔B15（发送前验证）

## EK-B15 · webhook 签名按原始字节 + body limit 先拒
- **机制**：CHANGELOG #4031/4075：Gitea webhook 签名在 body 解析前按**收到的原始字节**校验；Gitea 路由同 GitHub 32MiB body limit；**超限 webhook 在认证前拒绝**（"Oversized GitHub and source webhook deliveries are refused before authentication and create no receipt"）
- **证据**：CHANGELOG.md:93
- **links**：mechanism↔B14；constraint→B03

## EK-B16 · per-user runs 凭据不足则 refuse（不借 host 登录）
- **机制**：CHANGELOG #3433："Per-user runs without credentials refuse rather than borrowing the host login"；#4282："Runs and chat turns authenticate only as the identity they were given, or fail naming the credential. Tokens are no longer sent in cleartext or kept in run evidence"
- **证据**：CHANGELOG.md:92；core/src/github-auth
- **links**：causal→B06；mechanism↔B13

## EK-B17 · authored gate 只收声明决策（stale gate 拒绝）
- **机制**：CHANGELOG #3831/4048：gate 只接受声明的决策；未声明 decision ID → 报错并列出允许值；**stale gate 动作失败且不改动后续 gate**（防迟到响应改写状态）；每个 surface（console/chat/Slack/CLI）都记录 + 拒 stale
- **证据**：CHANGELOG.md:80；workflows/src/gate-rework.ts
- **links**：mechanism↔B18；subsystem↔workflows

## EK-B18 · attention 跨 surface 授权统一
- **机制**：wait: attention 可经 console/chat/Slack/CLI/API 回答（"Every answer records the surface it came through, and every surface can refuse a stale answer"）；`reject` 永不取消 run（#4038/4063）；`expectedGate` 冲突 → 409 且后续 gate 保持打开（#4162）
- **证据**：CHANGELOG.md:104-105,125
- **links**：mechanism↔B17；subsystem↔workflows

## EK-B19 · fan-out 子 run 独立等待（父暂停不取消子）
- **机制**：fan_out 子节点共享父 checkout，`max_parallel` 并发；子 pause 作为自己的 run 等待，父在 fan_out gate 上暂停并列出未完成子——"instead of the child being cancelled"（#3965/4125）
- **证据**：CHANGELOG.md:109；workflows/src/child-runs.fan-out-pauses.test.ts
- **links**：dependency→B17；mechanism↔B06（子 run 所有权独立）

## EK-B20 · AGENTS.md 治理自约束（governed engine 边界）
- **机制**：Archon = "governed agentic automation engine, not a replacement for the coding agents it orchestrates"；single operator / client isolation at deployment layer（"Multiple users within one install are supported and are not multi-tenancy"）；"Keep policy, transport, persistence, and execution concerns separate"；"Silent fallback in an agent runtime can waste money or broaden capabilities"；machine-checkable rules in code/types/tests/lint/CI
- **证据**：AGENTS.md（Product boundaries / Engineering taste / Fail clearly）
- **links**：constraint→全部；subsystem↔B13

## EK 图质量门
- 20 条 EK；links 覆盖率 100%；平均出边 ~1.9；游离 0%；全部可回溯仓库实盘

## EK-B21 · 工作目录 .env 隔离（strip 机制闭环）[reconciled via R-3]
- **机制**: server 启动 `stripCwdEnv()` 移除 CWD .env 键（index.ts:6-10，"when `bun run dev:server` is run from inside a target repo those keys leak into the server process"）；v0.12.0 曾被 provider 二进制加载目标仓库 `.env` 撤销 strip（项目 `ANTHROPIC_API_KEY` 可到 Claude 覆盖订阅）→ **0.12.1 修复**："Release binaries … no longer load `.env` or `bunfig.toml` from their working directory"（#4426/4427）
- **证据**: CHANGELOG.md:31（[0.12.1] - 2026-10-17 节）+ packages/server/src/index.ts:6-10
- **links**: mechanism↔EK-B13（凭据最小暴露）；causal→EK-B16（凭据不进 run）

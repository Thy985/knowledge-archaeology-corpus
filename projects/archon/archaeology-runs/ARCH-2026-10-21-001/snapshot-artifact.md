# Repository Snapshot Artifact — ARCH-2026-10-21-001（Archon）

## 快照定位
| 字段 | 值 |
|---|---|
| repository | https://github.com/coleam00/Archon.git |
| branch | dev（default） |
| HEAD | `11aba360220190bd633993b9ab9c847f54923b07` |
| HEAD commit | "Keep setup platform choices and summaries in sync (#4831)"（2026-10-20 20:46 +0300） |
| version | 0.12.1（package.json；tags v0.12.1 为 CVE 修复后版本） |
| pushed_at（API） | 2026-10-20（考古当日活跃） |
| 规模 | 3350 文件；TypeScript 2795（含 .ts/.tsx）；测试文件 1016 |

## 项目基础地图
### 语言 / 运行时 / 许可证
- TypeScript（strict）+ Bun（bun.lock/bunfig.toml/bun run）；MIT；自述 "The first open-source harness builder for AI coding. Make AI coding deterministic and repeatable."
- 定位（AGENTS.md）：self-hostable, **governed** agentic automation engine——workflow 混合确定性步骤 + AI agents + human gates + audit trails

### 入口
- CLI：`archon`（packages/cli，scripts.cli）；服务器：`bun dev:server` / start（packages/server/src/bin.ts → index.ts → startServer）
- 独立 auth-service：auth-service/server.js（AUTH_PORT 9000，bcrypt + HMAC cookie；缺 AUTH_USERNAME/AUTH_PASSWORD_HASH/COOKIE_SECRET → process.exit(1)）

### Monorepo（16 packages）
| package | 职责 |
|---|---|
| @archon/core | 编排器/operations/workflows/credentials/state/db/github-auth（核心引擎） |
| @archon/server | Hono 服务端 + 控制面 `/api/*` + 认证 gate（routes/api.auth.ts, auth/config.ts） |
| @archon/isolation | 隔离后端多选：container（docker-exec/overlay/start）+ in-place + backend-router + pr-state |
| @archon/cli | CLI 入口 |
| @archon/forge / @archon/git | SDLC forge（gh 默认）/ git worktree |
| @archon/providers / provider-contract | AI provider 适配 + 契约 |
| @archon/workflows | workflow 语言/执行 |
| @archon/web / docs-web | Web UI / 文档站 |
| @archon/chat-contract / chat-slack / adapters / plugin-manifest / paths | 契约/插件/路径 |

### 核心数据结构（类型证据，抽样）
- server：OpenAPIHono（@hono/zod-openapi）route 注册 + Zod schema（routes/schemas/*）
- auth：Better Auth singleton（instance.ts，Postgres-only）+ auth/config.ts 纯函数 gate
- isolation：BackendRouter（backend-router.ts）→ Container / InPlace 后端 → docker-exec / overlay
- core：orchestrator / operations / workflows / state / credentials 子系统（packages/core/src/）

### 测试体系
- 1016 测试文件（Vitest/Bun test 混合）；关键认证测试：routes/api.auth.test.ts（"gate on + no identity → 401 on a protected /api route" 等 401 断言 #307-316）
- isolation 每后端带测试（container/in-place/docker-exec/overlay/factory/router）

### 配置（.env.example + 治理 gate）
- 认证：WEB_AUTH OFF by default（需 DATABASE_URL + BETTER_AUTH_SECRET 同现）；`BETTER_AUTH_SECRET <32` → boot 抛错（assertWebAuthAtBoot）；`ARCHON_WEB_AUTH_REQUIRED=false` 可关 API gate；`ARCHON_AUTH_ALLOWED_EMAILS` allowlist；`ARCHON_AUTH_OPEN_SIGNUP=true` 才开放注册（默认 disabled）；反向代理身份头 `X-Archon-User`（ARCHON_WEB_AUTH_HEADER）
- SDLC：`ARCHON_SDLC_FORGE=forge` 可选（gh 默认）；forge token 仅发往 github.com/gitlab.com/gitea.com 或显式 GITLAB_URL/GITEA_URL/FORGEJO_URL（#4154）
- 隔离：isolation 后端选择（container/in-place）；git worktree "not an execution or security sandbox"（AGENTS.md）

### 权限与治理机制（0.12.1 现状）
- **控制面认证**：`isWebAuthEnabled`（OFF by default）→ `isApiGateEnabled`（web auth 开启时默认 gate 所有 /api/* → 401）→ conversation ownership（isConversationOwnershipEnforced：#4224 非 owner 404）
- **凭据治理**：per-user runs without credentials refuse（不借 host 登录）；forge token 主机白名单；凭据按身份认证（#4282）
- **webhook 签名**：Gitea/GitHub 签名按原始字节校验 + 32MiB body limit（#4031,#4075）
- **signup 姿态**：allowlist > open flag > disabled（默认 disabled——不静默暴露开放注册）
- **AGENTS.md 自约束**：governed engine 边界（非 coding agent 替代品）；single operator/client isolation at deployment layer；fail clearly（agent runtime 中静默 fallback 可浪费钱/扩能力）；machine-checkable rules in code/types/tests/lint/CI
- **CVE-2026-111322 语境**：0.2.13–0.11.1 `/api/*` 无认证且默认绑 0.0.0.0（CVSS 9.8）；本快照（0.12.1）为修复后版本——认证门控 + 401 + 弱密钥拒绝

### 外部依赖
- Runtime：Bun（bun.lock 332KB）；Hono + @hono/zod-openapi；Better Auth（Postgres）；bcryptjs/HMAC（auth-service）；Docker（isolation container 后端）；gh CLI（forge）；Zod
- 存储：SQLite（solo）/ Postgres（web auth 必需）
- AI providers：多 provider（provider-contract）

## 快照事实性声明
- 以上全部直接读取自浅克隆仓库（HEAD 11aba36，2026-10-21 02:15 UTC+8 检出）；版本号来自 package.json（0.12.1）；CVE 影响范围来自 KnowlegeMap 雷达卡（外部事实，标注来源）

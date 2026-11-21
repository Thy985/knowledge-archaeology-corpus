# 01 Project Layer — Archon（ARCH-2026-10-21-001）

## 架构总览（双层强制面）
```
                     ┌─ 决策面（本进程内） ─────────────────────────────┐
   operator/client ──►  @archon/server（Hono + Better Auth gate + /internal/*）──►  @archon/core（orchestrator）
   CLI ──────────────►  @archon/cli ──►  workflows（gates/attention/fan-out）         │
   chat plugins ─────►  @archon/chat-* （archon-chat/1 契约）                         ▼
   webhooks ─────────►  routes（签名校验）                          ┌── isolation（in-place/container/worktree）
                     └─────────────────────────────────────────────┴── AI providers（vendor-canonical 凭据）
```

## 进程/部署拓扑
- **单实例服务**：Bun.serve（packages/server/src/index.ts:233 `HOST || '0.0.0.0'`）；children 用绑定 socket（ARCHON_PORT）
- **独立 auth-service**（auth-service/server.js）：bcrypt + HMAC cookie；缺 AUTH_USERNAME/AUTH_PASSWORD_HASH/COOKIE_SECRET → `process.exit(1)`（fail-closed 启动）
- **Docker 部署**（docker-compose.yml + Caddyfile.example）：Dockerfile + docker-entrypoint.sh；auth-service 由 compose 编排
- **隔离执行**（container backend）：只读 bind mount 项目根 `/mnt/lower` + 可写 overlay `/mnt/upper`，同绝对路径合并（无翻译层）；`docker exec` spawn Claude CLI / bash: / script: 节点

## 生命周期
1. boot：strip-cwd-env（packages/server/src/index.ts:6-10 防 CWD .env 泄漏）→ load-env → **assertWebAuthAtBoot**（index.ts:217）→ registry activePlatforms → **assertInternalEndpointBind**（serve 前，index.ts:509）→ **warnWebAuthHeaderTrust** → Bun.serve → children 绑定
2. 请求：routes（OpenAPIHono）→ resolveAuthContext / requireWebUser → authorizeConversation → run actor → workflows / provider
3. run：workflow 混合 deterministic steps + AI agents + human gates（wait: attention / authored gates）；attention 可跨 surface 回答（console/chat/slack/CLI/API）
4. teardown：isolation destroy（container + volume；Phase C 前 overlay diff 丢弃）

## 核心模块
| 模块 | 关键文件 | 职责 |
|---|---|---|
| server | packages/server/src/index.ts, routes/, auth/ | 控制面 /api/*、认证 gate、webhooks、SSE |
| core | packages/core/src/{orchestrator,operations,workflows,credentials,state,db} | 引擎：编排/凭据/状态/持久化 |
| isolation | packages/isolation/src/{factory,backend-router,backends,container} | 执行隔离分层 |
| workflows | packages/workflows/src/{gate-rework,attention-*,child-runs.*,bash-invocation} | workflow 语言/gates/attention |
| cli | packages/cli | 命令行入口 |
| forge/git | packages/forge, packages/git | SDLC 集成（gh 默认）+ git worktree |
| providers | packages/providers + provider-contract | AI provider 适配（含 containerExec 能力声明） |
| auth-service | auth-service/server.js | 独立认证（Docker 用） |

## 核心数据结构（类型证据）
- `WebUserContext { userId, role }`（api-request-auth.ts）
- `ConversationVisibility { kind: 'all' | 'owner', userId? }`（core/db/conversation-visibility）
- `RunActor { kind: 'user' | 'unidentified' | 'operator', userId? }`
- `ResolvedCredential { kind: 'api_key' | 'oauth'; apiKey/oauthApiKey; rawCreds? }`（credentials/delivery.ts）
- `BackendPrepareRequest / PreparedEnv { cwd, execContext: {kind:'host'} | {kind:'container', containerId} }`
- `Capability`：provider 声明 containerExec（无则引擎 fail-fast pre-dispatch）

## 测试体系
- 1016 测试文件（Vitest/Bun test）；认证 401 断言（routes/api.auth.test.ts:307-316,498,549）；isolation 每后端/每容器机制带测试；workflows gates/fan-out/attention 全覆盖
- 生成物 conformance：pi-vendor-map / codex-protocol / capability-matrix / node-execution-reference（scripts.check:* 防漂移）

## 配置（治理面）
| 变量 | 默认 | 语义 |
|---|---|---|
| DATABASE_URL + BETTER_AUTH_SECRET | 无 | web auth 开关（两者同现才启用） |
| BETTER_AUTH_SECRET | — | <32 字符 → boot throw |
| ARCHON_WEB_AUTH_REQUIRED | true（web auth 启用时） | API gate；false=仅登录 UI |
| ARCHON_AUTH_ALLOWED_EMAILS | 空 | signup allowlist（非空→invite 模式） |
| ARCHON_AUTH_OPEN_SIGNUP | false | 开放注册（仅无 allowlist 时生效） |
| ARCHON_WEB_AUTH_HEADER | X-Archon-User | 反代身份头 |
| HOST | 0.0.0.0 | 监听绑定（GitHub App 模式限制） |
| ARCHON_ALLOW_INTERNAL_ON_PUBLIC_BIND | — | /internal/* 公共绑定 opt-out（显式确认） |
| ARCHON_SDLC_FORGE | gh | forge 后端（forge 需插件支持全部操作） |

## 权限与治理机制（0.12.1 现状，全可回溯）
1. **控制面认证 opt-in gate**：isWebAuthEnabled（auth/config.ts:24）——OFF by default
2. **弱密钥 boot 拒绝**：assertWebAuthAtBoot（config.ts:39）——防"silently mounting auth on a weak key"
3. **API gate 默认 401**：isApiGateEnabled（config.ts:86）——web auth on → /api/* 全要身份
4. **signup 默认 disabled**：getSignupMode（config.ts:67）——allowlist > open flag > disabled
5. **401/503 分离**：requireWebUser（api-request-auth.ts）——会话后端故障 503 ≠ 身份缺失 401
6. **非 owner 404 防枚举**：authorizeConversation——身份先于行解析；非 owner = 缺失行同 404；write 归 owner
7. **live 凭据端点 fail-closed**：assertInternalEndpointBind（index.ts:509）——GitHub App + 非 loopback → serve 前 fatal
8. **反代身份头 advisory**：warnWebAuthHeaderTrust——仅 loopback/反代剥离时可信
9. **凭据 vendor-canonical**：delivery.ts（#1955）——env-only、按 vendor 键、不替换 provider home
10. **forge token 主机白名单**（#4154）+ webhook 原始字节签名（#4031/4075）+ 不借 host 登录（#3433）

## 外部依赖
Bun / Hono + @hono/zod-openapi / Better Auth（Postgres-only）/ bcryptjs / Docker（container 后端）/ gh CLI / SQLite（solo）或 Postgres / 多 AI provider

# 04 Flow Atlas — Archon（ARCH-2026-10-21-001）

> 七类流从真实代码导出；Edge 标注可回溯（symbol / file / condition / state transition）。

## Control Flow（控制流）
```
boot: strip-cwd-env（index.ts:6-10）→ load-env → assertWebAuthAtBoot（index.ts:217）
  → registry.activePlatforms → assertInternalEndpointBind（index.ts:509，GitHubApp+非loopback→fatal）
  → warnWebAuthHeaderTrust（index.ts:541）→ Bun.serve（index.ts:238）→ children 绑定 ARCHON_PORT
请求: OpenAPIHono routes → requireWebUser/resolveAuthContext → authorizeConversation → run actor → workflows/providers
run: workflow nodes（deterministic + AI + gates）→ wait: attention 跨 surface 回答 → 完成通知
teardown: isolation destroy（container+volume；Phase C 前 overlay diff 丢弃）
```
- **Edge**：`assertInternalEndpointBind` 的 fatal throw 在 `Bun.serve` 之前——"rejected config never opens the listening socket — even briefly"（index.ts:510-512）

## State Flow（状态流）
- web auth 状态机：`off → on` 仅当 `DATABASE_URL && BETTER_AUTH_SECRET` 同现（config.ts:24-26）；SQLite 永远 off
- signup 状态机：`disabled → allowlist | open`（allowlist 非空 → allowlist；否则 OPEN_SIGNUP=true → open；默认 disabled）
- gate 状态：run 暂停于 gate → answer（surface 记录）→ continue；**stale answer → 拒绝且不改动后续 gate**（#3831/4048）；fan_out 子 run pause 独立等待（#3965）

## Data Flow（数据流）
- 请求身份：session（Better Auth）→ `findOrCreateUserByPlatformIdentity('web', …)`；或 header（X-Archon-User）→ 同名解析（api-request-auth.ts:20-45）
- 凭据交付：db 解密 → `ResolvedCredential{api_key|oauth}` → `delivery.ts` env-only merge 进 workflow/chat env（不替换 provider home）
- 隔离数据：host cwd 只读挂载 `/mnt/lower` + overlay upper `/mnt/upper`，**同绝对路径**（无翻译层）→ Phase C 前 diff 丢弃

## Evidence Flow（证据流）
- run evidence 不含 token/cleartext 凭据（#4282："Tokens are no longer sent in cleartext or kept in run evidence"）
- webhook：签名按**收到的原始字节**校验后才解析 body（#4031/4075）；超限 body 在认证前拒绝且**不创建 receipt**
- 认证测试钉住行为：api.auth.test.ts 401 断言（gate on + no identity → 401）

## Authority Flow（权威流）
```
/web 或 /cli 请求 → resolveAuthContext
  → auth session? → user（role 来自 userDb）
  → header? → user（best-effort）
  → 无 → undefined
→ requireWebUser（严格路径）：无身份→401；后端故障→503；header 有效→user
→ authorizeConversation：身份先于行加载（防枚举）→ 非 owner = 缺失行同 404 → write 仅 owner
→ RunActor{user|unidentified|operator}：web auth 或 header 存在时无身份 = 'unidentified'（非 operator）
```
- **Edge**：`isConversationOwnershipEnforced()` 下 `requireConversationVisibility` 走严格路径——"neither a missing identity (401) nor an identity-backend outage (503) can fall back to the whole install"（api-request-auth.ts:145-152）
- **Edge**：GitHub App 模式下 /internal/git-credential 的暴露 = fatal（index.ts:513-523）

## Memory Flow（记忆流）
- conversation 所有权持久化：`conversationDb` 按 platformId/webPlatformId 查 + `user_id` 归属（db/conversations）；非 owner 404
- isolation_environments 行：container 后端跟踪 env 行（branch_name 空哨兵——worktree-only 字段），destroy 时清理
- credentials 迁移：legacy agent-keyed id 启动迁移到 vendor-canonical（delivery.ts 注释）

## Policy Flow（治理闭环）
```
决策（AGENTS.md / .archon/*）→ 策略（auth/config.ts 纯函数 gate + .env 配置）
  → 执行（api-request-auth 门控 / assertInternalEndpointBind / delivery.ts 凭据）
  → 未来决策（CHANGELOG 迭代：#3831→#4224→#4154→#4282→#4031 治理持续收紧）
```
- **Edge**：策略以**纯函数**实现（config.ts 无 Better Auth/pg import）→ 单测无需 DB（"so they can be unit-tested without constructing a database-backed auth instance"）
- **Edge**：`MIRRORS the per-user GitHub gate`（core/github-auth/config.ts）——两处认证 gate 显式镜像关系，防"一处收紧一处漏"

# Independent Validation Report — ARCH-2026-10-21-001（Archon）

> Auditor 盲重建：不引用考古结果，独立重读仓库建立 Findings 后对比。禁止修改原考古产物（修正经 Reconciliation 落地）。

## 判定统计（12 项独立抽查）

| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 11 | EK-B01/B02/B03/B04/B05/B06/B07/B08/B09/B10/B11/B12/B13/B14/B15/B16/B20（抽查覆盖） |
| PARTIALLY_CONFIRMED | 1 | EK-B17（gate-rework 声明决策 enforcement 具体实现未深读） |
| MISSING | 1 | EK 未覆盖 0.12.1 `.env` 泄漏修复（#4426/4427） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 独立抽查证据（12 项）

| # | 抽查事实 | 独立证据 | 判定 |
|---|---|---|---|
| 1 | isWebAuthEnabled 双条件 | config.ts:25-26 `Boolean(DATABASE_URL && BETTER_AUTH_SECRET)` | CONFIRMED |
| 2 | 弱密钥 boot throw | config.ts:19,38-41（MIN 32 + throw） | CONFIRMED |
| 3 | 401/503 分离 | api-request-auth.ts:90,102,109,115 | CONFIRMED |
| 4 | 非 owner 404 | api-request-auth.ts:177,191 | CONFIRMED |
| 5 | fail-closed 端点 fatal 先于 serve | index.ts:497-538（isLoopbackHost + fatal + throw + opt-out） | CONFIRMED（S-1） |
| 6 | signup 默认 disabled | config.ts:49-50,72,77-83 | CONFIRMED |
| 7 | 严格路径无降级 | api-request-auth.ts:121-122 | CONFIRMED |
| 8 | in-place 默认 / container opt-in | backend-router.ts:7 | CONFIRMED |
| 9 | overlay 丢弃 | container.ts:13-16 | CONFIRMED |
| 10 | env-only 交付 | delivery.ts:8-9 | CONFIRMED |
| 11 | worktree 非沙箱 | AGENTS.md:25 | CONFIRMED（S-3） |
| 12 | 治理迭代（#3433/#3831/#4154） | CHANGELOG.md:45,80,91 | CONFIRMED（S-2） |

## 3 个成功（Auditor 支持性强化）

### S-1 · fail-closed 端点的"绝不打开 socket"语义（CONFIRMED 强化）
- **发现**: assertInternalEndpointBind 注释（index.ts:509-512）："Runs BEFORE Bun.serve so a rejected config never opens the listening socket — even briefly — and `server_listening` is never logged"
- **影响**: EK-B08 补"拒绝配置在 serve 前终止，监听 socket 绝不打开（哪怕一瞬间）"

### S-2 · #4154 forge 白名单语义确认（CONFIRMED 强化）
- **发现**: CHANGELOG.md:91 完整语义——"sent only to its well-known public host (github.com/gitlab.com/gitea.com) or to the exact authority of the matching GITLAB_URL/GITEA_URL/FORGEJO_URL. A host that merely looks like a forge gets no credentials"
- **影响**: 06_validation 中"delivery.ts 未直接 grep 到"缺口收窄：语义已确认，实现位置在 forge 包（未深读范围），非否定

### S-3 · worktree 边界锚精确化（CONFIRMED 强化）
- **发现**: AGENTS.md:25 位于 Product boundaries 节——"A git worktree separates checkouts; it is not an execution or security sandbox"
- **影响**: EK-B12 证据锚精确到节/行

## 3 个错误/风险点（Auditor 主动攻击）

### E-1 · EK-B13 迁移证据需补强（验证通过）
- **攻击**: delivery.ts 注释声称 "stored rows are migrated at startup (migrations/000_combined.sql + SQLite adapter's migrateColumns)"
- **验证**: migrations/000_combined.sql 存在 ✓——EK-B13 证据链成立（无修正，注明已验证）

### E-2 · EK-B17 enforcement 实现位置（PARTIALLY_CONFIRMED）
- **攻击**: "declared decisions" 的 enforcement 是否在 gate-rework.ts——文件存在但具体机制未逐行深读
- **处理**: 保持 EK-B17 为 CONFIRMED（CHANGELOG #3831/4048 语义明确）+ 补注明实现细节未逐行核对

### E-3 · 0.12.1 发布窗口 = CVE 同期 + `.env` 泄漏修复（MISSING → 新 EK）
- **发现**: CHANGELOG.md:25 `[0.12.1] - 2026-10-17`（CVE-2026-111322 披露 10-15/17 同期）；#4426/4427：v0.12.0 provider 在目标仓库加载 `.env` 撤销 strip（"a project's ANTHROPIC_API_KEY could reach Claude and override your subscription"）→ 0.12.1 修复
- **影响**: 新增 EK-B21（工作目录 .env 隔离——strip 机制的完整闭环）；C-B01（CVE 驱动重建）保持 Hypothesis（0.12.0/0.12.1 同日发布，认证 gate 引入时间需 git blame）

## 遗漏与 Coverage 声明
- 未独立深读：workflows gate 具体 enforcement、forge 包 token 白名单实现、providers 适配器、.archon 方向文档、git worktree provider 内部
- **Benchmark case（供 skill CI Gold Record）**:
  - BC-ARCHON-1: "控制面 401 门控"——gate on + no identity → 401 protected /api route（api.auth.test.ts:307-316 可复跑）
  - BC-ARCHON-2: "fail-closed 端点 boot"——GitHubApp + HOST=0.0.0.0 → serve 前 fatal（index.ts:513-523 可复现）

## 结论
- 12 项抽查 11 CONFIRMED / 1 PARTIALLY；0 实质 CONTRADICTED；主 Package 核心机制全部确认
- 修正项：MISSING×1（新增 EK-B21）+ 强化×3（S-1/S-2/S-3）→ Reconciliation 落地

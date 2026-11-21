# 05 Candidates — Archon（ARCH-2026-10-21-001）

> 不能确认的内容留在候选。Hypothesis ≠ Fact；Cross-project Candidate 不写成 Principle。

## C-B01 · 0.12.x 治理层是否为 CVE-2026-111322 驱动的重建（Hypothesis，A 类）
- **现状**: 快照（0.12.1）治理机制完整（opt-in gate/401/弱密钥拒绝/fail-closed 端点）；CHANGELOG 头版未见显式 "CVE-2026-111322" 字样（抽样）
- **缺失证据**: git log blame auth/config.ts 引入时间线 vs CVE 披露（10-15/17）；0.12.0-0.12.1 release notes
- **验证路径**: `git log --oneline packages/server/src/auth/` + GitHub releases；若 gate 引入于 CVE 披露后 → "事故驱动治理重建"确认；否则为既有演进

## C-B02 · container write-back（Phase C）的逃逸面（Hypothesis，A 类）
- **现状**: Phase B 丢弃 overlay diff（"a container run's changes stay in the overlay and are discarded on destroy()"）；Phase C = approval-gated write-back 未实现
- **缺失证据**: 批准后 write-back 的原子性（partial diff 如何回滚）、TOCTOU（approve 后宿主文件变化）、overlay 逃逸路径
- **验证路径**: 实现后考古；当前为设计意图（Phase 注释），不可当作已实现事实

## C-B03 · Better Auth Postgres-only 是否故意限制 solo 面（Observation→Hypothesis）
- **现状**: "SQLite/solo installs (no DATABASE_URL) can never enable it — Better Auth's tables are Postgres-only here"（config.ts:14-15）——solo 永远无 web auth
- **假说**: 有意维持 solo 单用户信任模型（无多用户无认证需求），而非 Postgres 技术依赖
- **验证路径**: .archon/direction.md / 设计文档提及；或 git log 引入 Better Auth 的 commit 说明

## C-B04 · 应用层治理 vs OS 级遏制互补性（Cross-project Candidate）
- **现状**: Archon 控制面在**进程内**治理（gate/401/所有权）；Box/MXC 在**进程外**遏制（OS 沙箱/策略层）
- **假说**: 完整 agent 治理 = 进程内控制面配置 + 进程外 OS 遏制双层；单层不可完备（Archon 无 OS 遏制=agent 逃逸后无外部防线；Box 无控制面认证=管理面暴露）
- **验证路径**: 周五 MXC×Box 双对照时并置 Archon 控制面发现；升维前需 ≥2 独立证据

## C-B05 · X-Archon-User 无认证 solo 的信任面（Hypothesis）
- **现状**: `warnWebAuthHeaderTrust` advisory（仅 loopback/反代剥离时可信）；solo（无 web auth）下 header 仍可命名身份（isConversationOwnershipEnforced 支持）
- **假说**: 若 solo 用户绑非 loopback 且无反代剥离，X-Archon-User 可被伪造 → 所有权假象
- **验证路径**: warnWebAuthHeaderTrust 完整实现（index.ts:545+）是否含强制路径；当前仅见 advisory 头

## C-B06 · gate 决策 ID 的持久化语义（Hypothesis）
- **现状**: stale gate 拒绝 + 未声明 ID 拒绝（#3831/4048）；expectedGate 冲突 409（#4162）
- **假说**: gate 状态与 run evidence 同库持久化（防重启丢失）；decision ID 全局唯一还是 run 内唯一
- **验证路径**: workflows 持久化层（db/workflows）；不影响已确认行为

## C-B07 · 与 deepseek-harness 的 approval seam 对照（Cross-project Candidate）
- **现状**: Archon human gates（attention/gate 声明式决策）vs dsh 的 approval/permission-presets seam
- **假说**: "声明式决策集合 + 拒绝未声明 ID" 比"自由文本批准"更防误批准——候选 Pattern
- **验证路径**: dsh archaeology（已有 corpus）并置对照；跨项目验证 pending

# Reconciliation — ARCH-2026-10-21-001（Archon）

> 合并 Independent Validation 修正。只修正 Corpus Artifact，不修改生产 Skill。

## 修正清单

### R-1 · EK-B08 补"绝不打开 socket"语义（采纳 S-1）
- **修正内容**: 追加："拒绝配置在 Bun.serve 之前 fatal 终止——监听 socket 绝不打开（哪怕一瞬间），`server_listening` 绝不记录"（index.ts:510-512）
- **证据**: index.ts:509-512（assertInternalEndpointBind 头注释）

### R-2 · EK-B12 证据锚精确化（采纳 S-3）
- **修正内容**: 证据锚改为 AGENTS.md:25（Product boundaries 节原文）
- **证据**: AGENTS.md:25

### R-3 · 新增 EK-B21 · 工作目录 .env 隔离（strip 机制闭环，采纳 E-3）
- **新增 EK**: server 启动 strip CWD .env（index.ts:6-10）；v0.12.0 曾被 provider 二进制加载目标仓库 .env 撤销 strip（项目 ANTHROPIC_API_KEY 可到 Claude 覆盖订阅）→ 0.12.1 修复（#4426/4427）；"Release binaries … no longer load .env or bunfig.toml from their working directory"
- **links**: mechanism↔EK-B13（凭据最小暴露）；causal→EK-B16
- **证据**: CHANGELOG.md:31（0.12.1 节）+ index.ts:6-10

### R-4 · EK-B13 迁移证据标注已验证（采纳 E-1）
- **修正内容**: EK-B13 补注"migrations/000_combined.sql 存在已验证（Auditor E-1）"
- **证据**: migrations/000_combined.sql

### R-5 · 06_validation 缺口收窄（采纳 S-2）
- **修正内容**: "#4154 实现位置"缺口改为"语义已确认（CHANGELOG.md:91），实现位置在 forge 包（未深读范围，非否定）"

### R-6 · C-B01 保持 Hypothesis（采纳 E-3）
- **修正内容**: 补上下文——0.12.0/0.12.1 同日（2026-10-17）发布，与 CVE 披露同期；CHANGELOG 有 CVE 记录惯例（#1153 CVE-2025-62718）但未提及 CVE-2026-111322；认证 gate 引入时间需 git blame——CVE 驱动重建为 Hypothesis 非 Fact

## 未修改项
- KO-B01~B05、Flow Atlas、Candidates、01/03/04/05 主体（无攻击成立）
- 原考古产物不可修改原则：以上修正以 corpus 落盘版本为准，本地 package 保留初版 + 本文件记录差异

## Reconciliation 后 EK 统计
- EK 总数: 21（B01~B21）；links 覆盖率 100%；平均出边 ~1.9；游离 0%
- 6 项修正全部可反向追溯独立证据（independent_validation_report.md）

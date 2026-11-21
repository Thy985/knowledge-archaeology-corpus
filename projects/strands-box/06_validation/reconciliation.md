# Reconciliation — ARCH-2026-10-23-001（strands-box）

> 合并 Independent Validation 修正。只修正 Corpus Artifact，不修改生产 Skill。

## 修正清单

### R-1 · EK-B05 构造面表述修正（采纳 E-1）
- **修正内容**: "无公开构造器" → "无公开**直接**构造器（interpreter_resolved/under_home 为 pub(crate)）；只能经 **PathResolver.approve_host/approve_host_entry/approve_virtual** 批准获得（path.rs:203/243/278），返回 Result<ApprovedPath, PathRefusal>——拒绝 spelling 与 canonical identity 不一致、非绝对路径、越界路径"
- **证据**: crates/policy/src/path.rs:203-243-278；类型化主张不变（raw spelling 无法直接构造 ApprovedPath）

### R-2 · EK-B12 补 FORBIDDEN_PATHS 精确定位 + floors 五层（采纳 E-2 + S-2）
- **修正内容**: "containment 持有 FORBIDDEN_PATHS" 补精确定位——`crate::os_paths::{forbidden_paths, Forbidden, ForbiddenClass, Match}`（os_paths.rs），floors.rs 消费（:297/:375/:406-410）；require_all 五层 deny-only floors（platform_credential_grants/bounded_grants/no_conflicting_grants/live_path_identities/no_identity_changing_executables），纯检查先于文件系统触碰；**deny 豁免**（"A denial is exempt…deny / is the strongest entry"）
- **证据**: crates/containment/src/floors.rs:1-40,297,375；os_paths.rs

### R-3 · EK-B07 补 per-tool action 生成器（采纳 E-3）
- **修正内容**: "generated per-tool action" → "由 **cedar-policy-mcp-schema-generator =0.6.1** 从 raw MCP tools/list 生成 typed Cedar action schemas（mcp-tools-sdk =0.4.1 解析），精确 pin"
- **证据**: crates/policy/Cargo.toml [dependencies] 注释

### R-4 · 新增 EK-B21 · telemetry DecisionRecord 结构（采纳 MISSING）
- **新增 EK**: DecisionRecord（record.rs:338）+ DecisionCause（:132）+ 一 lane/box；决策记录含 action/resource/decision（DecisionObserver 契约喂入）
- **links**: mechanism↔EK-B17（observer 非阻塞契约）；dependency→EK-B01（三拦截点决策都入记录）
- **证据**: crates/telemetry/src/record.rs:132,338

### R-5 · 06_validation 缺口收窄（采纳 S-1/S-3）
- **修正内容**: fail_closed_guards "测试钉死拒绝行为" → 补"加载必须失败是 Definition of Done（expect_load_error panic 断言）"；EK-B10 补"Bearer 注入有测试验证（credential/mod.rs:255）"

## 未修改项
- KO-B01~B05、Flow Atlas、Candidates、01/03/04/05 主体（无攻击成立）
- 原考古产物不可修改原则：以上修正以 corpus 落盘版本为准，本地 package 保留初版 + 本文件记录差异

## Reconciliation 后 EK 统计
- EK 总数: 21（B01~B21）；links 覆盖率 100%；平均出边 ~1.7；游离 0%
- 5 项修正全部可反向追溯独立证据（independent_validation_report.md）

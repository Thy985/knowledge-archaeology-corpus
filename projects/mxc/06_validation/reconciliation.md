# Reconciliation — ARCH-2026-10-24-001（MXC）

> 合并 Independent Validation 修正。只修正 Corpus Artifact，不修改生产 Skill。

## 修正清单

### R-1 · EK-M04 枚举名修正（采纳 E-1）
- **修正内容**: "按 denied 处理" → "路径身份无法确定（`FilesystemIntent::Unknown`——权限拒/unreachable mount/I/O error）即 fail-closed 触发；`Absent`（cleanly missing）才安全跳过。文档层 'denied' 是语义说法，实现层 trigger 是 Unknown"
- **证据**: core/mxc_common/filesystem_object.rs:88-96

### R-2 · EK-M10 proxy 方向性补强（采纳 E-2）
- **修正内容**: 补"ProcessContainer 代理要求 network.ingress.default: 'allow' 的方向性因果——代理端点是 host-loopback 直连目标，因此代理语境下 ingress 必须放行 loopback 方向（配套而非矛盾）"
- **证据**: docs/containment-policy/1.0.0/policy.md（ProcessContainer 代理段）

### R-3 · EK-M01 补 native-ingress 迁移例外（采纳 E-3）
- **修正内容**: 补"SDK policy types 不构成另一套配置权威"是**目标态**——native-ingress 存在 deprecated binding 例外（迁移暂存期）
- **证据**: docs/development/architecture/versioning.md（native-ingress section）

### R-4 · EK-M14 补契约注册数据结构（采纳 MISSING）
- **修正内容**: 补 ContractStatus（Published/Development 双态）+ ContractDescriptor（version/status/schema_id/schema_path）——registry.rs 注册元数据结构
- **证据**: core/mxc_contract/registry.rs（ContractStatus::as_str + ContractDescriptor）

### R-5 · 06_validation 缺口收窄（采纳 S-1/S-2/S-3）
- **修正内容**: EK-M02 补三路断言（JSON/base64/exact）；EK-M11 补 errorCode+messageContains 机器可校验错误契约；EK-M06 补三处实验门

## 未修改项
- KO-M01~M06、Flow Atlas、Candidates、01/03/04/05 主体（无攻击成立）
- 原考古产物不可修改原则：以上修正以 corpus 落盘版本为准，本地 package 保留初版 + 本文件记录差异

## Reconciliation 后 EK 统计
- EK 总数: 20；links 覆盖率 100%；avg 出边 ~1.9；游离 0%
- 4 项修正全部可反向追溯独立证据（independent_validation_report.md）

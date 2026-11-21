# Independent Validation Report — ARCH-2026-10-24-001（MXC）

> Auditor 盲重建：不引用考古结果，独立重读仓库建立 Findings 后对比。禁止修改原考古产物（修正经 Reconciliation 落地）。

## 判定统计（12 项独立抽查）

| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 11 | A#1/A#2/A#3/A#5/A#6/A#7/A#8/A#9/A#10/A#11/A#12 |
| PARTIALLY_CONFIRMED | 1 | A#4（FilesystemIntent 枚举名细节） |
| MISSING | 1 | Proxy 语义细节（EK-M10 需补 "ProcessContainer 代理 require ingress allow" 的方向性解释） |
| DOWNGRADED / OVER_GENERALIZED / CONTRADICTED / NEEDS_HUMAN_REVIEW | 0 | — |

## 独立抽查证据（12 项）

| # | 抽查事实 | 独立证据 | 判定 |
|---|---|---|---|
| 1 | 三轴分离权威表 | versioning.md:16-19（四行决策权威表） | CONFIRMED |
| 2 | closed JSON 拒绝 unknown field | config_parser.rs:2005-2016（"unknown field `experimental`" 三路断言：JSON/base64/exact） | CONFIRMED |
| 3 | deny wins over allow | network_rules.rs:676-679（"Denies precede allows so an explicit deny overrides a broader allow"） | CONFIRMED |
| 4 | FS fail-closed trigger | filesystem_object.rs:88-96（Absent vs **Unknown**；"This is the fail-closed trigger when deniedPaths are present"） | **PARTIALLY**（E-1） |
| 5 | 后端注册 10 项 + ProbeCoverage 强制声明 | backend_registry.rs:14（"Declaring this is mandatory…"）+ :34（BACKENDS[10]） | CONFIRMED |
| 6 | experimental_enabled 门 | run.rs:54-55,66,156,186（backend_unavailable before host resolution） | CONFIRMED |
| 7 | 双 learning capability 语义分离 | logging-access-denied.md:22-37（"must not be conflated" + deny-and-record/audit + Weakened 标注） | CONFIRMED |
| 8 | IsolationSession 缺 network 必须拒绝 | tests/policy/sdk-v1/invalid/isolation-session-without-network.json（errorCode malformed_request + "requires an explicit network policy"） | CONFIRMED |
| 9 | telemetry 未知值 Blocked | telemetry.md:37（"anything else — **Blocked** (fail closed)"） | CONFIRMED |
| 10 | WSLc 双态 | policy.md:146（"only all-deny isolated or all-allow bridged"） | CONFIRMED |
| 11 | sandboxId 前缀推断后端 | policy.md:159-160（"infer the backend from its prefix"）+ :191 示例 | CONFIRMED |
| 12 | 测试文化代码化 | AGENTS.md:77（"do not invent evidence"）+ 行为覆盖段 | CONFIRMED |

## 3 个成功（Auditor 支持性强化）

### S-1 · closed JSON 拒绝有三路断言（CONFIRMED 强化）
- **发现**: config_parser.rs 对同一 unknown-field 错误同时断言 JSON 直读 / base64 编码 / parse_exact 三条路径（2008-2015）——契约校验深度超单路径
- **影响**: EK-M02 强化——closed 契约在 FFI/base64 路径同样被测试钉死

### S-2 · invalid 金标准带 messageContains（CONFIRMED 强化）
- **发现**: invalid/isolation-session-without-network.json 声明 errorCode（malformed_request）+ messageContains（"requires an explicit network policy"）——策略拒绝有机器可校验错误契约
- **影响**: EK-M11 强化——拒绝非静默，且错误契约可断言

### S-3 · experimental 门在 3 处独立执行（CONFIRMED 强化）
- **发现**: run.rs:66（run 路径）:156（另一路径）:186（backend_unavailable map）三处检查 experimental_enabled
- **影响**: EK-M06 强化——门不是单点，跨执行路径复验

## 3 个错误/风险点（Auditor 主动攻击）

### E-1 · FilesystemIntent 枚举名（PARTIALLY_CONFIRMED）
- **攻击**: EK-M04 说"按 denied 处理"——代码实际是 `FilesystemIntent::Unknown` variant（filesystem_object.rs:88-96）：Absent（cleanly missing 安全跳过）vs **Unknown**（权限拒/unreachable mount/I/O error → fail-closed trigger）
- **核实**: "denied" 是文档层说法；实现层 trigger 是 Unknown（无法判定身份即拒绝）
- **修正**: EK-M04 表述改为"路径身份无法确定（Unknown）即 fail-closed 触发——Absent 才安全跳过"

### E-2 · Proxy 方向性语义补强（MISSING 细节）
- **攻击**: EK-M10 未解释 ProcessContainer 为何需要 ingress allow
- **核实**: policy.md（"For ProcessContainer, a proxy requires network.ingress.default: 'allow'"）——代理端点是 host-loopback 直连，因此代理语境下 ingress 必须放行 loopback 方向
- **修正**: EK-M10 补方向性因果（proxy=host-loopback 目标 → ingress allow 是配套而非矛盾）

### E-3 · "SDK 不构成配置权威"的迁移语境（MISSING 细节）
- **攻击**: EK-M01 未点名 native-ingress 迁移
- **核实**: versioning.md（"The native-ingress section identifies the deprecated binding exceptions that remain while the migration is staged"）——"无另一套权威"是目标态，存在 deprecated 例外
- **修正**: EK-M01 补"native-ingress deprecated 例外（迁移暂存期）"

## 遗漏（MISSING → 补）
- **Registry 契约数量与状态位**: ContractStatus Published/Development 双态 + ContractDescriptor（version/status/schema_id/schema_path）——EK-M14 补契约注册数据结构细节

## Benchmark case（供 skill CI Gold Record）
- **BC-MXC-1**: closed JSON unknown-field 三路断言（config_parser.rs 可复跑）
- **BC-MXC-2**: deny wins over allow（network_rules.rs EgressPlan 单测可复跑）
- **BC-MXC-3**: IsolationSession 缺 network 拒绝契约（tests/policy/sdk-v1/invalid 金标准，errorCode+messageContains 双断言）

## 结论
- 12 项抽查 11 CONFIRMED / 1 PARTIALLY；0 CONTRADICTED；主 Package 核心机制全部确认
- 修正项：PARTIALLY×1（EK-M04）+ MISSING×3（EK-M10/EK-M01/EK-M14）→ Reconciliation 落地

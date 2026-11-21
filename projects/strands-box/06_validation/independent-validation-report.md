# Independent Validation Report — ARCH-2026-10-23-001（strands-box）

> Auditor 盲重建：不引用考古结果，独立重读仓库建立 Findings 后对比。禁止修改原考古产物（修正经 Reconciliation 落地）。

## 判定统计（12 项独立抽查）

| 判定 | 数量 | 涉及 |
|---|---|---|
| CONFIRMED | 11 | EK-B01/B02/B03/B04/B06/B07/B08/B09/B13/B15/B17/B18（抽查覆盖） |
| PARTIALLY_CONFIRMED | 1 | EK-B05（ApprovedPath 构造面表述需修正） |
| MISSING | 1 | FORBIDDEN_PATHS 精确定位（os_paths.rs）+ containment floors 五层 |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 独立抽查证据（12 项）

| # | 抽查事实 | 独立证据 | 判定 |
|---|---|---|---|
| 1 | 三适配器=一个 authority | box/AGENTS.md:292 "Three views of one authority, never three" | CONFIRMED |
| 2 | DenyReason 四分支 | decision.rs:50-56（Forbidden/NoMatch/PolicyPending/InternalFault）+ :167,:176 使用 | CONFIRMED |
| 3 | cedar 安全 pin | Cargo.toml:29-31（"a *security* pin…part of the fail-closed contract" + =4.11.0） | CONFIRMED |
| 4 | SELF_DEFENDED_FILES | policy.rs:20（["box.toml","policy.dw"]） | CONFIRMED |
| 5 | ApprovedPath 构造面 | path.rs:203/243/278 approve_host/approve_host_entry/approve_virtual 为 **pub fn**（PathResolver），返回 Result<ApprovedPath,PathRefusal> | **PARTIALLY**（E-1） |
| 6 | resource() 标签非权威 | request.rs:253 "This is a label, never an authority" | CONFIRMED |
| 7 | MCP 双门 gate2 非默认拒绝 | decisions.md:816-826 原文 | CONFIRMED |
| 8 | egress fold fail-closed | decider.rs:26-28（"nothing egresses"+DenyReason::Credential）+ :41 MutationCollision | CONFIRMED |
| 9 | 凭据路径永不授予 | box/AGENTS.md:280（containment FORBIDDEN_PATHS）→ floors.rs + os_paths 精确定位 | CONFIRMED（E-2 补定位） |
| 10 | reload 调用方认证 | policy.rs:228-242 注释原文 | CONFIRMED |
| 11 | DecisionObserver 非阻塞 | observe.rs:11-12（"must not block, await, or perform input or output"+infallible） | CONFIRMED |
| 12 | fail_closed_guards 契约 | fail_closed_guards.rs:28-30（expect_load_error + panic "load should have aborted"）+ :38（fs:raed 拼写错误策略必须加载失败） | CONFIRMED |

## 3 个成功（Auditor 支持性强化）

### S-1 · fail-closed 加载契约是 Definition of Done（CONFIRMED 强化）
- **发现**: fail_closed_guards.rs 头注释（"Policy load-contract regression tests (Definition of Done)"）+ expect_load_error panic 断言——拼写错误（fs:raed）等 fail-open 类策略**必须加载失败**，否则测试 panic
- **影响**: EK-B19 强化——fail-open 回归被测试钉死（不是只测"拒绝行为"，而是测"加载必须失败"）

### S-2 · containment floors 是 deny-only 五层（CONFIRMED 强化）
- **发现**: floors.rs require_all 五层（platform_credential_grants → bounded_grants → no_conflicting_grants → live_path_identities → no_identity_changing_executables），按成本排序，两个纯检查先于触碰文件系统的检查（"refusing it should not require acting on it first"——grant 路径可能 workload 影响）；**deny 豁免**（"A denial is exempt…deny / is the strongest entry a caller can write"——deny 越宽越安全）
- **影响**: EK-B12 补"floors 五层 + deny 豁免"细节

### S-3 · egress Bearer 注入有测试验证（CONFIRMED 强化）
- **发现**: credential/mod.rs 测试断言 "Authorization: Bearer real-secret"（:255）+ StripQueryParam/AddQueryParam mutation
- **影响**: EK-B10 补"Bearer 注入 + mutation 机制有测试验证"

## 3 个错误/风险点（Auditor 主动攻击）

### E-1 · ApprovedPath 构造面表述（PARTIALLY_CONFIRMED）
- **攻击**: EK-B05 说"无公开构造器"——grep 发现 `pub fn approve_host/approve_host_entry/approve_virtual`（path.rs:203/243/278，PathResolver 上）
- **核实**: ApprovedPath 无**直接**公开构造器（interpreter_resolved/under_home 为 pub(crate)）；但存在公开批准路径 approve_*（返回 Result<ApprovedPath, PathRefusal>，拒绝 spelling 与 canonical identity 不一致/越界/非绝对路径）
- **修正**: EK-B05 表述改为"无公开直接构造器；只能经 PathResolver.approve_* 批准获得（拒绝返回 PathRefusal）"——类型化主张仍成立（raw spelling 无法直接产生 ApprovedPath），但批准入口是公开 seam

### E-2 · FORBIDDEN_PATHS 定位（MISSING→补）
- **攻击**: EK-B12 说"containment 持有 FORBIDDEN_PATHS"但未给精确定位
- **核实**: `crate::os_paths::{forbidden_paths, Forbidden, ForbiddenClass, Match}`（os_paths.rs）；floors.rs 消费（:297,:375,:406-410）
- **修正**: EK-B12 补"定义于 os_paths.rs::forbidden_paths，消费于 floors.rs"

### E-3 · MCP per-tool action 生成器（MISSING 细节）
- **攻击**: EK-B07 说"generated per-tool action"未点名生成器
- **核实**: Cargo.toml `cedar-policy-mcp-schema-generator = "=0.6.1"`（"Generates typed Cedar action schemas from raw MCP tools/list responses"）+ `mcp-tools-sdk = "=0.4.1"`
- **修正**: EK-B07 补"per-tool action 由 cedar-policy-mcp-schema-generator 从 tools/list 生成（精确 pin）"

## 遗漏（MISSING → 新增 EK-B21）
- **telemetry 决策记录结构**: DecisionRecord（record.rs:338）+ DecisionCause（:132）——EK 未覆盖 telemetry 内部结构（此前仅提 records.jsonl）
- 新增 EK-B21（telemetry DecisionRecord/DecisionCause + 一 lane/box 记录语义）

## Benchmark case（供 skill CI Gold Record）
- **BC-BOX-1**: fail-closed 加载契约——fail-open 类策略（如 fs:raed 拼写错误）必须加载失败（fail_closed_guards.rs 可复跑）
- **BC-BOX-2**: 统一事件语义——Request::Fs 的 FsAccess 映射（request.rs:44-130：Read/Write/Delete/Move 分类）可复现

## 结论
- 12 项抽查 11 CONFIRMED / 1 PARTIALLY；0 CONTRADICTED；主 Package 核心机制全部确认
- 修正项：PARTIALLY×1（EK-B05 构造面表述）+ MISSING×2（EK-B12 定位、新增 EK-B21）→ Reconciliation 落地

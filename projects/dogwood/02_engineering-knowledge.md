# 02 · Engineering Knowledge — Dogwood（EK Graph，宽底座）

> v3.1 规范：每条 EK 声明 `links`（六类边）。证据等级：S3=implemented（代码/文档存在）、S3+测试意图=测试文件存在且 docstring 明确行为（未运行，因环境无 Rust 工具链）。

## 图统计
- EK 总数：28；有 links：28（100%）；平均出边 ≥2；游离 0
- 证据等级：S3（代码/文档）= 22，S3+测试意图 = 6

---

## A. 核心语言机制

### EK-01 Cedar 派生语法 + 时间条件扩展
- **内容**：语言以 Cedar `permit`/`forbid` + `when`/`unless` 为基础，新增时间条件（`since`/`formerly`/`once`/窗口聚合）回看事件历史；示例策略 `when formerly within 1h { Action::"Approve"::request{...} }`。
- **证据**：README.md 首段 + 示例。
- **links**：subsystem→EK-02（temporal 引擎）、subsystem→EK-07（compile-to-Cedar）、contrast→EK-06。
- **等级**：S3。

### EK-02 时间条件三构件：since / formerly / once + 窗口聚合
- **内容**：temporal extension 提供 `since`（自某事件以来）、`formerly`（过去窗口内发生过）、`once`（曾发生）与聚合（窗口内计数/求和等）；静态检查含嵌套聚合拒绝（reject_nested_agg）、保留 binder 拒绝（reject_reserved_binder）、exists 安全（check_exists_safe）。
- **证据**：extension/temporal/check.rs（fn reject_nested_agg:124、reject_reserved_binder:154、check_exists_safe:182）；guide/04-temporal-expressions.md。
- **links**：mechanism→EK-03（范围限制）、causal→EK-04（事件历史）、constraint→EK-05（max_window）。
- **等级**：S3。

### EK-03 范围限制分析（range restriction）：时间条件被限制在可检查范围
- **内容**：check.rs 实现范围限制分析（collect_range_restricted + eq_restricted_var + body_has_sigil + check_chain/check_demands）——把时间条件约束到"可判定的检查范围"（bounded var / sigil 检测），不可满足的时间链被拒绝。
- **证据**：extension/temporal/check.rs（collect_range_restricted:465、check_chain:255、body_has_sigil:563）。
- **links**：dependency→EK-02、constraint→EK-05、causal→EK-07（lowering 前的静态保证）。
- **等级**：S3。

### EK-04 事件 schema：decision vs history points
- **内容**：.dwschema 声明事件种类，区分**决策点**（decision point，触发授权判定的事件）与**历史点**（history point，仅作为时间条件回看对象）；`is_decision_kind`/`decision_kinds` 匹配 authorizer 的 gate。
- **证据**：event_schema/derive.rs（2004 行）+ incremental_lowering.rs 测试意图（"decision_kinds / is_decision_kind match the authorizer's gate"）。
- **links**：causal→EK-02（历史点供时间条件）、subsystem→EK-10（pin）。
- **等级**：S3+测试意图。

### EK-05 max_window 回看上限（默认 24h）
- **内容**：service schema 默认事件窗口上限 24h（可配置）；时间条件只能在窗口内回看——既限资源又防"远古事件"进入决策。
- **证据**：AGENTS.md（"a 24h window cap"）；guide/03-event-schema.md。
- **links**：constraint→EK-02、constraint→EK-03、mechanism→EK-11（内存引擎窗口）。
- **等级**：S3。

### EK-06 编译到 Cedar 而非新造运行时
- **内容**：策略 lower 到标准 Cedar；temporal/provider 字段变 `context.*` 槽，运行时填充；policy engine 可本地 Cedar 或远程策略存储。
- **证据**：README "Compile-to-Cedar" + cedarify/to_ast.rs + schema_augment.rs。
- **links**：causal→EK-07（lowering 管线）、causal→EK-08（schema 增广）、subsystem→EK-01。
- **等级**：S3。

### EK-07 两阶段 lowering + 调用方 distincter（增量可组合）
- **内容**：ParsedPolicySet::parse → lower 两阶段；调用方可传 **distincter** 让子策略独立 lowering（不同来源到达时）再组合成一个 Cedar PolicySet，合成 policy ids 与 hoisted `context.<id>` 字段名不碰撞；默认路径 `policy_<index>`；一个 parsed set 可对多个 action schema lowering；feed-forward 增积式（前次增广 schema 进入后续 lowering）。
- **证据**：incremental_lowering.rs 测试意图（8 条行为）+"The two-phase API ... caller-supplied distincter"。
- **links**：mechanism→EK-06、dependency→EK-03、contrast→EK-09（infallible 求值）。
- **等级**：S3+测试意图。

### EK-08 schema 增广（schema_augment）把扩展字段物化为 Cedar context 槽
- **内容**：lowering 时 cedarify/schema_augment.rs 把 temporal/provider 字段作为 `context.<id>` 增广进 Cedar schema——扩展语义由编译期物化，运行时按槽填充。
- **证据**：cedarify/schema_augment.rs（1501 行）。
- **links**：causal→EK-07（feed-forward 的来源）、subsystem→EK-06。
- **等级**：S3。

## B. Information Providers（注入面）

### EK-09 provider 契约：erroring provider = UNDEFINED BEHAVIOR（契约/实现分离）
- **内容**：provider 契约（guide 05）明确：运行出错是**未定义行为**——无语义保证，策略集不得依赖 deny-on-error（防御脚本是唯一防御）。**参考实现**选择：出错 → Deny（错误进 diagnostics），防"permit 的 guard 无法检查却放行"的 partial context 授权；is_authorized 设计为 **infallible**（求值失败折叠进 diagnostics）。契约层与实现层严格分离，策略不得依赖任一侧。
- **证据**：fail_closed.rs docstring 全文。
- **links**：mechanism→EK-16（infallible 设计）、constraint→EK-13（确定性）、subsystem→EK-12（provider eval）。
- **等级**：S3+测试意图（fail_closed.rs）。

### EK-10 确定性沙箱 Rhai 引擎：new_raw + 白名单 + max_operations 后盾
- **内容**：provider 脚本用**共享、锁定、确定性**的 Rhai 引擎：从 `Engine::new_raw()`（不含 StandardPackage 的 clock 等）起步，只注册白名单能力（"bare Rhai engine cannot do I/O at all; exposing a capability is an explicit act"）；OnceLock 共享不可变；`max_operations` 作为 runaway-script 后盾（注释："a runaway-script backstop"）。
- **证据**：extension/provider/eval.rs（4-8 行 docstring + fn engine() 71-72 + MAX_OPS 注释）。
- **links**：contrast→EK-11（README 说"默认无限制"——文档生产指引 vs 代码默认后盾）、dependency→EK-09、subsystem→EK-12。
- **等级**：S3。

### EK-11 生产边界自我声明（README 6 条限制）
- **内容**：README 尾部明列参考解释器生产限制：①URL authority 不得从不可信事件字段构造；②lowered 策略必须过 Validator::validate()（否则退化窗口/未解析引用/类型不匹配在运行时意外行为）；③**不记录审计日志**（生产需在 is_authorized() 外包）；④多租户默认无隔离（pin 是重写 pass，不代表存储分区）；⑤Rhai 默认无 CPU/内存限制（部署需 max_operations/max_call_levels/超时）；⑥错误信息含策略内容（多租户 co-load 会泄露他人策略结构，生产需净化）。
- **证据**：README.md 尾部 "Important limitations"。
- **links**：mechanism→EK-13（诚实审计族）、constraint→EK-10、contrast→EK-09（文档 vs 代码）。
- **等级**：S3（文档自述）。

## C. 求值/授权引擎

### EK-12 Authorizer.infallible：is_authorized → Option<Response>，错误折叠进 diagnostics
- **内容**：`Authorizer::is_authorized(&mut self, event) -> Option<Response>`（infallible by design——求值失败折叠进 Response.diagnostics）；Response 含 decision + rule refs（reason）+ errors + allowed()。
- **证据**：authorize/mod.rs（Response:57-82、is_authorized:214）。
- **links**：dependency→EK-09（provider 错误进 diagnostics）、mechanism→EK-16。
- **等级**：S3。

### EK-13 可插拔引擎：PolicyEngine / TemporalEngine trait + builder
- **内容**：AuthorizerBuilder 可注入 policy_engine（本地 Cedar 或远程策略存储）与 temporal_engine（内存或数据库）——两轴独立可换；engine.rs 提供 CedarPolicyEngine + InMemoryTemporalEngine（Mode 枚举）。
- **证据**：authorize/mod.rs（builder:192、policy_engine:239、temporal_engine:248）；engine.rs（CedarPolicyEngine:96、InMemoryTemporalEngine:287）。
- **links**：mechanism→EK-07（组合）、subsystem→EK-05、subsystem→EK-12。
- **等级**：S3。

### EK-14 pin 分区：重写 pass 使策略"如同按 pin 键分区"
- **内容**：默认一个 authorizer 监视一条事件历史，principal 间无隔离；`pin` 特性在解释器实现**重写 pass**——策略被解释为按 pin 键字段分区；但**不意味着**存储层分区（README 明示生产需自行实现隔离）。
- **证据**：engine.rs PartitionKey（192）；README 多租户限制。
- **links**：contrast→EK-04、constraint→EK-11。
- **等级**：S3。

### EK-15 反例/边界测试文化：cedarify_adversarial + boundary_* + expected_failures
- **内容**：测试套件含 cedarify_adversarial.rs（"The goal is to break things: 畸形 provider 参数/深嵌套/空 schema/unicode/语义退化但语法有效"）、boundary_parser_rejects / boundary_validator_catches / boundary_macro_rejects（拒绝路径逐类断言）、expected_failures/（期望失败目录）。
- **证据**：dogwood-language/tests/ 目录 + cedarify_adversarial.rs docstring。
- **links**：mechanism→EK-18（docs-as-tests）、subsystem→EK-09（fail_closed 在其中）。
- **等级**：S3（测试意图阅读）。

### EK-16 决策确定性与等值语义测试
- **内容**：测试覆盖 decimal_equality_matches_cedar（小数等值须匹配 Cedar 语义）、entity_equality_verdicts（实体等值裁决）、error_message_differential（错误信息差异）、feed_forward_robustness（增广稳健）——确定性/兼容性是显式测试目标。
- **证据**：dogwood-language/tests/ 文件名。
- **links**：mechanism→EK-13、contrast→EK-09。
- **等级**：S3（测试意图阅读）。

## D. 治理/工作流

### EK-17 agent 工作流治理：.claude/skills + 强制 CLI 验证门
- **内容**：AGENTS.md 定义 authorization 生命周期（action schema → service schema → policies → check/replay），每阶段一个 `.claude/skills/` 单源真相 skill，且强制 `dogwood` CLI 验证门（"enforces a mandatory dogwood CLI validation gate"）；skill 引用 guide 而非复制。
- **证据**：AGENTS.md 全文 + .claude/skills/ 4 个 skill。
- **links**：mechanism→EK-19（MCP schema 生成）、subsystem→EK-01。
- **等级**：S3。

### EK-18 docs-as-tests：文档示例 = CLI 检查的 bundle，doc 编译失败 = build failure
- **内容**：dogwood-docs 测试 harness 把每个示例 bundle（policy + schema [+ trace/providers]）跑过 `dogwood` 二进制；"a doc example that stops compiling is a build failure"——文档与实现强耦合防漂移。
- **证据**：Cargo.toml workspace 注释 + dogwood-docs/tests/examples.rs + docs_as_tests/。
- **links**：mechanism→EK-15、constraint→EK-06。
- **等级**：S3。

### EK-19 MCP schema 生成：tools/list manifest → action schema
- **内容**：action schema 可手写或从 MCP `tools/list` manifest 生成（guide/11-mcp-schema-generation；docs_as_tests/mcp_schema_gen.rs）——把"每个工具/操作一个 action + context 布局"自动化。
- **证据**：AGENTS.md（"hand-written or generated from an MCP tools/list manifest"）+ guide/11。
- **links**：causal→EK-17（生命周期第 1 步）、subsystem→EK-01。
- **等级**：S3。

### EK-20 单 crate 封装（impl/api 拆分跨 crate 无法强制封装）
- **内容**：Cargo.toml 明确设计决策：dogwood-language 是单 crate——"A split impl/api pair cannot enforce encapsulation across a crate boundary — a published api forces its impl dependency to be published too, and the impl's pub items are then reachable directly. One crate with crate-private modules gives compiler-enforced encapsulation."；dogwood-cli 是薄壳（序列化报告/错误类型留在 crate，因 JSON 输出形状是 CLI 关注点非库关注点）。
- **证据**：Cargo.toml workspace 成员注释。
- **links**：mechanism→EK-13、subsystem→EK-17。
- **等级**：S3。

### EK-21 策略验证门：lowered 策略必须过 Validator（README 明示）
- **内容**：Validator::validate() 是 lowered 策略授权前强制步骤——跳过会允许退化窗口/未解析引用/类型不匹配在运行时意外行为。
- **证据**：README 限制 #2 + validator.rs/validate.rs。
- **links**：constraint→EK-07、dependency→EK-12。
- **等级**：S3。

## E. 证据/API 面

### EK-22 trace/replay：trace_api + corpus + CLI replay
- **内容**：提供 trace_api.rs / corpus.rs 与 CLI `replay`（policy + trace 文件 → 逐事件裁决）——可回放的历史验证。
- **证据**：dogwood-language/src/trace_api.rs、corpus.rs；README quick start（replay）。
- **links**：mechanism→EK-18（docs 示例含 trace）、subsystem→EK-04。
- **等级**：S3。

### EK-23 宏库降低策略样板（macros 2335 行）
- **内容**：dogwood-language/src/macros/ 提供策略宏（docs/06-macros + 09-calling-macros；boundary_macro_rejects 测试）——声明式样板复用。
- **证据**：macros/mod.rs 2335 行 + guide/06/09。
- **links**：mechanism→EK-01、subsystem→EK-17。
- **等级**：S3。

### EK-24 RichType 枚举取代 stringly-typed 富类型
- **内容**：CHANGELOG 2026-08-12：Replace stringly-typed rich types with a RichType enum；ProviderField/TemporalField 标 non_exhaustive（API 演进许可）。
- **证据**：CHANGELOG.md。
- **links**：constraint→EK-09、subsystem→EK-10。
- **等级**：S3。

### EK-25 错误诊断可定位：Diagnostics 含 rule refs + errors
- **内容**：authorize 的 Diagnostics 提供 reason()（DogwoodRuleRef 迭代器）+ errors()（str 迭代器）——裁决可解释到具体规则。
- **证据**：authorize/mod.rs（Diagnostics:39-57）。
- **links**：mechanism→EK-12、subsystem→EK-21。
- **等级**：S3。

### EK-26 日志/审计缺失是明示缺口（非静默缺陷）
- **内容**：README 限制 #3："Dogwood returns decisions but does not log them. A production implementation should use some form of logging around is_authorized() for compliance and forensics."——把缺失写进文档成为已知边界。
- **证据**：README 限制清单。
- **links**：mechanism→EK-11、contrast→EK-15。
- **等级**：S3。

### EK-27 事件字段不可信：URL authority 构造禁令
- **内容**：README 限制 #1：Never construct the URL authority from untrusted event fields（provider 安全模式）——事件历史数据被视为不可信输入。
- **证据**：README 限制清单 + provider guide。
- **links**：constraint→EK-09、constraint→EK-10。
- **等级**：S3。

### EK-28 语义/实现分层写入测试（策略不得依赖任一侧）
- **内容**：fail_closed.rs 的元层面：不仅测行为，还**写清语义边界**——"Under the provider contract an erroring provider is UNDEFINED BEHAVIOR ... What this test pins is narrower ... the reference implementation's chosen handling ... If this implementation choice changes deliberately, update this test; policies must not depend on it either way."——测试同时是契约文档。
- **证据**：fail_closed.rs docstring。
- **links**：mechanism→EK-09、mechanism→EK-18、contrast→EK-11。
- **等级**：S3+测试意图。

---

## EK Graph 边密度

| 指标 | 值 |
|------|-----|
| 平均出边 | ~2.1 |
| 游离 EK | 0 |
| mechanism 边 | EK-02/03/07/09/12/15/16/17/18/19/23/28 间多条 |
| causal 边 | EK-02→04→07→08→17；EK-06→07 |
| constraint 边 | EK-05→02/03；EK-11→10；EK-21→07/12；EK-27→09/10 |
| contrast 边 | EK-06↔01；EK-09↔11；EK-10↔11；EK-14↔04；EK-16↔09 |
| subsystem 边 | temporal 族（02/03/05）；provider 族（09/10/11/27）；authorize 族（12/13/21/25）；测试族（15/16/18/28） |

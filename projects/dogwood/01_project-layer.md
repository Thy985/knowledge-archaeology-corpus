# 01 · Project Layer — Dogwood（项目地图）

> 本层为 L0 工程事实，全部可追溯到仓库实际内容（commit c6237c88）。

## 1.1 项目定位

Agent 与其工具的**治理语言**（governance language）：支持 Cedar 策略并新增时间条件（`since`/`formerly`/`once`/聚合），回看 agent 近期事件历史后判定。本仓库是**参考解释器**（reference interpreter），明确 NOT for production（README 尾部）。

## 1.2 架构（crate 级）

```
dogwood-language（核心库，单 crate 封装）
├── parser/            # .dw 策略解析（2455 行，grammar.pest）
├── interpreter/       # eval.rs（2107）/ value.rs / log_parse.rs
├── extension/temporal/  # 时间条件：parse/check/validate（1840+1597+1534 行）
├── extension/provider/  # Information providers：declarations/validate/eval（Rhai）
├── extension/dialect.rs
├── cedarify/          # compile-to-Cedar：to_ast.rs + schema_augment.rs（增广 schema）
├── event_schema/      # .dwschema：derive/relativize/parse（event 事件 schema）
├── authorize/         # Authorizer（is_authorized → Option<Response>）
├── engine.rs          # PolicyEngine / TemporalEngine trait + InMemory 实现 + PartitionKey
├── api.rs（1721）/ validator.rs / validate.rs / policy_schema.rs / service_schema.rs
├── ast.rs / policy_set.rs / policy_view.rs / trace_api.rs / corpus.rs
└── macros/            # 宏库（2335 行）

dogwood-cli            # dogwood 二进制：validate / lower / replay / schema
dogwood-docs           # mdBook guide（12 章）+ 示例 bundle（docs 即测试：examples.rs 跑 CLI）
```

## 1.3 核心模块职责

| 模块 | 职责 | 证据 |
|------|------|------|
| parser/ | .dw 策略文本 → AST（pest grammar） | parser/mod.rs 2455 行 |
| extension/temporal/ | 时间条件语法/静态检查/验证（范围限制、嵌套聚合拒绝、保留 binder 拒绝） | check.rs 43/124/154/182/248 等函数 |
| extension/provider/ | 信息提供者：声明、校验、求值（沙箱 Rhai 确定性引擎） | eval.rs（Engine::new_raw + 白名单 + max_operations） |
| cedarify/ | 把 temporal/provider 字段 lower 成 Cedar `context.*` 槽 + schema 增广 | cedarify/to_ast.rs + schema_augment.rs |
| event_schema/ | 事件 schema：决策点 vs 历史点、pin、max_window 上限 | event_schema/derive.rs 2004 行 |
| authorize/ | 授权入口：Authorizer.is_authorized（infallible，错误折叠进 diagnostics） | authorize/mod.rs:214 |
| engine.rs | PolicyEngine/TemporalEngine trait + CedarPolicyEngine + InMemoryTemporalEngine（Mode） | engine.rs:96/287 |

## 1.4 生命周期

- **策略生命周期**（AGENTS.md）：action schema（.cedarschema）→ service schema（.dwschema + providers）→ policies（.dw）→ check/replay；每步有 `.claude/skills/` 流程 + **强制 `dogwood` CLI 验证门**
- **求值生命周期**：validate → lower（compile-to-Cedar）→ replay/authorize（policy engine + temporal engine 可插拔）
- **MCP 接入**：guide/11-mcp-schema-generation（从 MCP tools/list manifest 生成 action schema）

## 1.5 主要状态

- 事件历史（event history）：InMemoryTemporalEngine（Mode 枚举，内存）；DB-backed 可插拔
- 策略集：ParsedPolicySet → LoweredPolicySet（两阶段 API，incremental lowering + distincter）
- 决策：Decision（permit/deny）+ Diagnostics（rule refs + errors）

## 1.6 配置与入口

| 项 | 值 | 证据 |
|----|-----|------|
| CLI | `dogwood validate/lower/replay/schema` | README + guide/12-cli.md |
| schema | .cedarschema（action）、.dwschema（event）+ providers.json（Rhai） | guide/03/10 |
| 默认窗口上限 | 24h（AGENTS.md：service schema 默认 max_window 24h） | AGENTS.md |
| 发布 | 内部 registry "brazil"（非 crates.io） | Cargo.toml workspace.package.publish |

## 1.7 权限 / policy / governance 机制

- 语言本体：permit/forbid + when/unless（Cedar 派生）+ temporal 条件
- 参考实现安全边界（README 尾部 6 条）：URL authority 不从不可信字段构造 / lowered 策略必须过 Validator / 审计日志缺失（生产需自包）/ 多租户默认无隔离（pin 是重写 pass 非存储分区）/ Rhai 沙箱默认无 CPU 内存限制 / 错误信息可能泄露跨租户策略结构
- 仓库治理：AGENTS.md + CLAUDE.md + .claude/skills/（4 个 skill，单源真相）+ CONTRIBUTING.md + CODE_OF_CONDUCT + SECURITY.md
- **语言契约 vs 实现选择分离**：fail_closed.rs 明确"provider 出错 = UNDEFINED BEHAVIOR（契约层无语义保证），参考实现选 Deny（实现层），策略不得依赖任一侧"——契约/实现严格分层

## 1.8 外部依赖

- Cedar（cedarpolicy，策略引擎本地或远程 policy store 可插拔）
- Rhai（嵌入式脚本，provider 求值）
- pest（parser 文法）
- mdBook（文档）；serde 等常规

## 1.9 设计证据入口

- Cargo.toml workspace 注释（单 crate 封装理由：impl/api 拆分跨 crate 无法强制封装）
- docs/guide/08-formal-specification.md（形式化规格）
- README 尾部限制清单（生产边界）
- .claude/skills/（agent 工作流治理）
- CHANGELOG.md（2026-08-12 起）

# Repository Snapshot — Dogwood（ARCH-2026-09-08-001）

```yaml
repository: https://github.com/dogwood-policy/dogwood
commit_sha: c6237c88099b3f492ecc5fcee42df06a19224b97
branch: main
tag: (无 tag，浅克隆 --depth 1)
repository_version: 1.0.0（Cargo.toml workspace.package.version）
analysis_timestamp: 2026-09-08T02:00:00+08:00
knowledge_archaeology_skill_version: v3.2
```

## 项目基础地图

```
dogwood/
├── README.md                  # 定位 + key features + 限制清单（NOT production）
├── AGENTS.md / AGENTS-README.md / CLAUDE.md  # agent 工作流治理
├── CHANGELOG.md               # 2026-08-12 起
├── SECURITY.md / CONTRIBUTING.md / CODE_OF_CONDUCT.md / LICENSE(Apache-2.0) / NOTICE
├── Cargo.toml                 # workspace：3 crates，单 crate 封装设计注释
├── dogwood-language/          # 核心库（Rust 2024）
│   ├── src/
│   │   ├── parser/            # .dw 策略解析（2455 行）
│   │   ├── interpreter/       # eval.rs（2107）/ value.rs / log_parse.rs
│   │   ├── extension/temporal/  # 时间条件 parse/check/validate（1840/1597/1534 行）
│   │   ├── extension/provider/  # Rhai 确定性引擎 + 声明/校验/求值
│   │   ├── cedarify/          # to_ast + schema_augment（compile-to-Cedar）
│   │   ├── event_schema/      # derive/relativize/parse（.dwschema）
│   │   ├── authorize/         # Authorizer（is_authorized → Option<Response>）
│   │   ├── engine.rs          # PolicyEngine/TemporalEngine + InMemory + PartitionKey
│   │   ├── api.rs（1721）/ validator.rs / validate.rs
│   │   ├── policy_schema.rs / service_schema.rs / ast.rs / policy_set.rs / policy_view.rs
│   │   ├── trace_api.rs / corpus.rs / macros/（2335 行）
│   ├── configuration/         # 起始 action/event schema
│   └── tests/                 # 30+ 测试文件（boundary_*/cedarify_adversarial/fail_closed/...）
├── dogwood-cli/src/           # dogwood 二进制：validate/lower/replay/schema（薄壳）
├── dogwood-docs/              # mdBook guide 12 章（含 08 formal spec）+ examples + tests
└── .claude/skills/            # 4 个 agent 工作流 skill（authoring-action-schema 等）
```

## 识别项

| 项 | 值 | 证据 |
|----|-----|------|
| 主要语言 | Rust（2024 edition） | Cargo.toml |
| 主要运行入口 | `dogwood` CLI（validate/lower/replay/schema）；库入口 Authorizer | README + authorize/mod.rs |
| 核心模块 | temporal extension / provider / cedarify / authorize / engine / event_schema | src/ 结构 |
| 核心数据结构 | Condition/Term/AggExpr（temporal AST）、ParsedPolicySet→LoweredPolicySet、Event、Response{decision,diagnostics}、PartitionKey | ast.rs/check.rs/authorize/mod.rs/engine.rs |
| 核心状态 | 事件历史（InMemoryTemporalEngine，Mode）；策略集（parsed/lowered） | engine.rs |
| 主要测试体系 | 边界拒绝（boundary_*）、对抗（cedarify_adversarial）、失败闭合（fail_closed）、增量组合（incremental_lowering）、期望失败（expected_failures）、docs-as-tests（docs_as_tests + examples.rs） | tests/ 目录 |
| 主要配置 | .cedarschema（action）/ .dwschema（event：decision/history points、pins、max_window 默认 24h）/ providers.json + Rhai / macros | AGENTS.md + guide |
| 权限/policy/governance 机制 | permit/forbid+when/unless（Cedar 派生）+ 时间条件 + provider 注入；AGENTS.md 生命周期 + 强制 CLI 门；生产限制清单（README 6 条） | README + AGENTS.md |
| 主要外部依赖 | Cedar（cedarpolicy）、Rhai、pest、mdBook、serde 等 | Cargo.toml（依赖声明） |

## 快照说明

- 浅克隆（--depth 1）HEAD c6237c88，main 分支。
- 规模：84 .rs 文件 / 46,208 行（find + wc 实测）。
- 本快照事实全部可回溯到仓库实际内容（文件路径/行号见各层引用）。

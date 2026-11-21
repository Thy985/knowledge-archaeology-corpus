# Restate — Project Archaeology Overview（ARCH-2026-11-01-001）

## 一句话认知

**Restate 是"步骤持久化派"durable execution 的独立运行时侧代表**：把每次 handler 执行切成 journal 条目（journal_v2），每步写持久化存储（Bifrost 日志 + RocksDB 表），崩溃/重启后从最后完成的 entry 恢复——与上轮考古的 agent-substrate（挂起恢复派）构成同一问题的两条技术路线对照。

## 项目定位（README 事实）
- "Restate is the simplest way to build resilient applications."
- 定位 "the Durable Runtime for your Agents and Backends"（KnowlegeMap candidate 记录，cross-check README：Durable AI Agents、Workflows-as-Code、Microservice Orchestration、Event Processing、Async Tasks、Stateful Actors/state machines）
- SDK：TypeScript / Java+Kotlin / Python / Go / Rust
- 采用证据：Replit 全量采用（"Restate now handles all durable orchestration at Replit"，来自 KnowlegeMap candidate，非本 repo 内断言——标 External/Observation）

## 考古范围与方式
| 项 | 值 |
|---|---|
| run_id | ARCH-2026-11-01-001 |
| repository | https://github.com/restatedev/restate.git |
| commit | `c30b4bced95c1934d12628f6438872a1bffb1c26`（main，2026-10-31 00:27:58 +0100） |
| 版本 | 1.8.2-dev（Cargo.toml） |
| 规模 | 1271 .rs / 367,514 行 / 20 .proto / 43 crates / 25MB |
| 模式 | initial（首次考古） |
| 方法 | 本地生产版 knowledge-archaeology v3.2（多角色只读考古）；无 Docker → 静态考古（不编译不跑集成测试） |
| 参照 | 上轮 agent-substrate（ARCH-2026-10-31-001）——挂起恢复派 vs 步骤持久化派对照 |

## 核心发现速览（详见各层）
1. **Journal 持久化是主机制**：每步执行写 journal_v2 条目（20 种 Command），崩溃后从最后完成 entry 恢复（`crates/worker/src/partition/state_machine/mod.rs:4968 append_journal_entry`）。
2. **幂等是显式状态机**：`handle_duplicated_requests` 按 InvocationStatus 分派——Free→新调用、in-flight→append response sink、Completed→回放响应；workflow 忽略幂等 key 返回 `WORKFLOW_ALREADY_INVOKED_INVOCATION_ERROR`（mod.rs:1076）。
3. **挂起/恢复**：OnSuspendCommand 先尝试用已有 notifications resolve `awaiting_on`（UnresolvedFuture 树），能解析直接 Resume；否则写 SuspendedEvent + 转 Suspended 状态 + 挂起 vqueue entry（suspend.rs）。
4. **Bifrost 分布式日志**："Restate's durable interconnect system"——loglet 抽象保证 append 返回 offset 即 durable committed；LogChainWriter seal/extend 日志链（bifrost.rs:50 ErrorRecoveryStrategy）。
5. **VQueue（keyed 调度）**：Sha256(partition_key||ty||scope||limit_key||exclusive?1:0||service||key) 生成 VQueueId，exclusive handler 独立队列天然无锁串行；DRR 调度器（util.rs:102）。
6. **元数据版本化**：MetadataStore put/delete 带 Precondition（None/DoesNotExist/MatchesVersion）防并发覆盖（metadata.rs:55）。
7. **治理证据驱动**：AGENTS.md/CLAUDE.md 强制 evidence-based + cite file:line + `cargo nextest run`；CI 18 workflows；存储迁移机制（migrations/ + features/ 双轨）。
8. **License 修正**：实际为 **BSL 1.1**（4 年后转 Apache 2.0），非 snapshot 初记的纯 Apache-2.0。

## 三层知识架构配比
| 层 | 数量 | 文件 |
|---|---|---|
| Project Layer | 1 地图 | 01_project-layer/ |
| Engineering Knowledge（EK Graph） | 44 条（全部带 links） | 02_engineering-knowledge/ |
| Generalized KO | 9 个（全部带 aggregation_rule） | 03_knowledge-layer/ |
| Flow Atlas | 7 类流 | 04_flow-atlas/ |
| Candidates | 6 条（跨项目假说/暂定模式） | 05_candidates/ |
| Validation & Evidence | 6 审计 + 反例 + 质量指标 | 06_validation/ |

## 认知状态声明
- 本 Package 区分 **Fact（S3 已实现 / S4 测试验证）** 与 **Observation（从代码推断）** 与 **Hypothesis（Candidates）**；跨项目结论一律标 `Cross-project validation pending`。
- 未编译/未运行集成测试（无 Docker），凡"运行时行为"类断言全部标 Observation/S4- 以下，不冒充 S5 运行时验证。

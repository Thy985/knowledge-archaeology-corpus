# Repository Snapshot Artifact — ARCH-2026-11-01-001

> 目标仓库：restatedev/restate（Restate：Building resilient applications made easy）
> 克隆方式：git clone --depth 1（浅克隆）

## 快照事实（全部可追溯）
| 项 | 值 |
|---|---|
| repository | https://github.com/restatedev/restate.git |
| HEAD SHA | `c30b4bced95c1934d12628f6438872a1bffb1c26` |
| branch | main |
| commit time | 2026-10-31 00:27:58 +0100 |
| commit msg | `[CI] Use $/ for is-latest-release in homebrew workflow` |
| 版本 | 1.8.2-dev（Cargo.toml） |
| license | **BSL 1.1**（LICENSE 头部：Business Source License 1.1，Licensor: Restate Software, Inc., Restate GmbH；Change Date: 4 years after release → Change License: Apache License, Version 2.0） |
| rust toolchain | 1.98.1（rust-toolchain.toml，edition 2024） |
| 仓库体积 | 25M |

## 代码规模
| 指标 | 值 |
|---|---|
| .rs 文件 | 1271（非 target） |
| .rs 行数 | 367,514 |
| .proto 文件 | 20 |
| 测试目录 | 5（crates/codederror、encoding、partition-store/src/tests 等） |
| 顶层 crates | 43（admin/bifrost/core/worker/partition-store/types/service-protocol/…） |

## 项目定位（README 事实）
- "Restate is the simplest way to build resilient applications."
- 分布式 durable 版通用构建块：**Durable AI Agents**、Workflows-as-Code、Microservice Orchestration、Event Processing、Async Tasks、Agents/Stateful Actors/state machines
- SDK：Typescript / Java+Kotlin / Python / Go / Rust
- 安装：Homebrew / npx @restatedev/restate-server

## 顶层结构
- `crates/`：43 个内部 crate（核心运行时）
- `server/`：restate-server 二进制（src/main.rs, lib.rs, build_info.rs, signal.rs, telemetry.rs）
- `cli/`：restate CLI（main.rs）
- `tools/`：restatectl / restate-doctor / xtask / bench 工具
- `charts/`：Helm charts；`docker/`：容器；`npm/`：npx 分发；`monitoring/`：监控
- `service-protocol/`：服务协议（子 crate service-protocol-v4 在 crates/）
- `docs/dev/`：开发文档；`AGENTS.md` + `CLAUDE.md`：AI 治理规则
- `.github/workflows/`：CI（ci.yml, docker, helm, npm, homebrew, release 等 18 个）

## 核心模块（crates/ 43 个）
| crate | 职责（由代码推断） |
|---|---|
| core | 节点核心：task_center（任务调度）/ network（message_router, connection_manager, tls）/ metadata / io（executor, reactor） |
| worker | 分区状态机：partition/state_machine（InvocationStatusProjection @mod.rs:5598）、invoker、ingress |
| partition-store | 持久化表：invocation_status_table / journal_table / journal_table_v2 / inbox_table / state_table / promise_table / locks_table / outbox_table / service_status_table / deduplication_table / fsm_table / journal_events / snapshots |
| types | 核心类型：journal（entries.rs:176 `EntryType` 21 种）/ invocation / identifiers / cluster_state / config / deployment / endpoint_manifest |
| bifrost | 日志系统（Loglet）：appender / background_appender / log_chain_writer / loglet（Bifrost 分布式日志） |
| storage-api | 存储抽象：invocation_status_table/mod.rs:144 `InvocationStatus` 8 态 |
| service-protocol / service-protocol-v4 | 服务协议 v4（bilrost 迁移见 docs/dev/bilrost-migration-guidelines.md） |
| ingress-http / ingress-kafka | 入口：HTTP / Kafka |
| metadata-store / metadata-server / metadata-providers | 元数据服务（RocksDB 持久化） |
| timer / timer-queue | 定时器 |
| sharding | 分片 |
| wal-protocol | WAL 协议 |
| rocksdb | RocksDB 封装 |
| log-server / log-server-grpc | 日志服务器 |

## 核心数据结构
### EntryType（journal 条目，crates/types/src/journal/entries.rs:176）
Input / Output / GetState / SetState / ClearState / GetStateKeys / ClearAllState / GetPromise / PeekPromise / CompletePromise / Sleep / Call / OneWayCall / Awakeable / CompleteAwakeable / Run / CancelInvocation / GetCallInvocationId / AttachInvocation / GetInvocationOutput / Custom（21 种）

### InvocationStatus（storage-api/src/invocation_status_table/mod.rs:144，8 态）
Scheduled / Inboxed / Invoked / Suspended{metadata, awaiting_on} / Paused / Completed / Free（default）

### 关键机制（由目录与代码推断，待阶段④验证）
- **Journal 持久化**：每步执行写 journal entry（Input→…→Output），崩溃后从最后完成 entry 恢复（步骤持久化派核心）
- **Bifrost 分布式日志**：loglet 抽象，多副本日志链（log_chain_writer）
- **Keyed 调度**：actor 模型（virtual objects）——按 key 分区，同 key 串行执行
- **状态表**：state_table（get/set/clear）、promise_table（awakeable）、locks_table（keyed 锁）、outbox_table（消息出站）
- **定时器**：timer-queue（Sleep entry）

## 权限与治理机制
- AGENTS.md：证据驱动（evidence-based）、禁止臆测、必须 cite 文件:行号、AGENTS.md applyTo '**' 全仓库生效；测试用 `cargo nextest run`（非 cargo test）
- CLAUDE.md：AI 助手规则（待阶段④读）
- .github/workflows/ci.yml：CI 门禁；cla.yml（CLA 检查）；release.yml（发布门禁）

## 测试体系
- `cargo nextest run`（AGENTS.md 指定）
- 核心测试目录：crates/partition-store/src/tests/（inbox/invocation_status/journal_events/journal/journal_v2/locks/state 等表级测试）
- crates/codederror/tests、crates/encoding/tests
- server/tests、tools/（bench 工具）

## 外部依赖
- RocksDB（crates/rocksdb）
- DataFusion（crates/storage-query-datafusion——查询引擎）
- Kafka（ingress-kafka）
- gRPC/bilrost（service-protocol，bilrost 迁移中）
- TLS（network/tls.rs）
- 其他：见 Cargo.lock（未逐条展开）

## 口径注记
- .rs 统计排除 target/ 与 vendor 类目录；proto 20 个为仓库全部 .proto
- 本快照为浅克隆（--depth 1），无历史 commit 数据；"修复/失败史"类证据需在阶段④从代码注释/docs 提取

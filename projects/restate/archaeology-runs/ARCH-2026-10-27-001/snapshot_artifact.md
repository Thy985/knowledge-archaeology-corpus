# Repository Snapshot Artifact — ARCH-2026-10-27-001

## 快照元数据
| 项 | 值 |
|---|---|
| repository | https://github.com/restatedev/restate.git |
| branch | main |
| commit | `db4b02e26171e3b35367144d27741dc7b56c04dd` |
| commit date | 2026-10-26 15:15:25 +0000（"HEAD@selection"） |
| workspace version | 1.8.2-dev（Cargo.toml workspace.package.version） |
| 最近 release | v1.8.1（10-21）/ v1.8.0（10-14）/ v1.7.14（10-22） |
| GitHub 元数据 | 4609★/228 forks · language=Rust · created 2023-01-04 · pushed 2026-10-26 |
| license | **BUSL-1.1**（LICENSE 文件：Restate Software, Inc./GmbH；禁止"Public Restate Platform Service"（第三方注册自身服务经管理面调用）；内部部署/GUI/DSL 抽象层豁免；Change Date = release 后 4 年 → Apache-2.0）——**非 OSI 开源许可，考古须标注** |

## 规模
- 文件数 1525（git ls-files）
- .rs 文件 1242 个，约 35.3 万行（`find . -name "*.rs"` wc -l）
- 含 `#[test]` 的 .rs 文件 196 个（cargo nextest 为测试入口，AGENTS.md 明示）
- workspace 成员：cli / crates/*（44 crates）/ server / lite / benchmarks / tools/*（restatectl、restate-doctor 等）/ workspace-hack

## 语言与技术栈
- Rust（edition 2024，rust-version 1.98.1，resolver 2）+ protobuf（prost 0.14，service-protocol v4）+ Arrow/DataFusion（query_engine 55.1）+ OpenTelemetry 0.33 + RocksDB（restate_rocksdb 封装）+ AWS SDK（dynamodb/msk/kms 等后端）+ tonic gRPC

## 入口
- 服务端二进制：`server/src/main.rs`（lib.rs/main.rs/build_info.rs/signal.rs/telemetry.rs——薄入口，核心在 crates/）
- 管理 CLI：`cli/src/main.rs` + commands/（restatectl 同类）

## 核心模块（crate 级，行数=全 crate .rs 行数）
| crate | 行数 | 职责（据代码目录与符号） |
|---|---|---|
| crates/worker | 33944 | 分区处理器：partition/processor（fsm.rs 状态机、dedup.rs 去重、outbox.rs、leadership.rs、status.rs）、state_machine/entries、subscription_controller |
| crates/partition-store | 22068 | RocksDB 状态面 13 表族目录：invocation_status_table / journal_table(v2) / state_table / inbox_table / outbox_table / promise_table / timer_table / vqueue_table / locks_table / deduplication_table / fsm_table / durable_lsn_tracking / snapshots / service_status_table |
| crates/bifrost | 19342 | 日志段（journal/log）：bifrost.rs（Bifrost/BifrostInner/MaybeLoglet/PreferenceToken）、appender、loglet/{provider}、providers/{local_loglet（RocksDB）、replicated_loglet（sequencer/log_server/network）、memory_loglet（测试）}、read_stream、watchdog |
| crates/core | 16091 | task_center（builder/extensions/handle/runtime/task_kind）、partitions、network、metadata |
| crates/vqueues | 11195 | 虚拟队列：scheduler/cache/migrations/context（virtual object 并发核心） |
| crates/node | 5343 | 节点装配 |
| crates/metadata-store / metadata-server | ~ | 集群元数据（分布式） |
| crates/timer / timer-queue | 1295+ | 定时器（durable timers） |
| crates/types | ~ | config（aws/bifrost/dynamodb_store/gcp/ingress/invocation/kafka/rocksdb/worker 等）、identifiers、sharding |

## 核心数据结构/状态（据代码符号）
- InvocationStatusV2Lazy（partition-store invocation_status_table，RocksDB value，分区键 + InvocationId/InvocationUuid）
- journal_table(v2)：durable journal（每步 entry 持久化）
- Bifrost Commit（bifrost/types.rs:77）——日志提交确认
- Loglet（bifrost/loglet.rs）——日志段抽象；Provider（provider.rs）——local（RocksDB）/ replicated（sequencer+log server）/ memory
- PartitionProcessor fsm（worker/partition/processor/fsm.rs）——分区状态机

## 配置与治理
- 配置：crates/types/src/config/（admin/aws/bifrost/dynamodb_store/gcp/ingress/invocation/kafka/log_server/metadata_server/networking/object_store/query_engine/rocksdb/throttling/worker）
- 治理：AGENTS.md（applyTo '**'——证据引用、迭代证据搜索、cargo nextest 规范、不臆造）；docs/dev/（development-guidelines/rust-guidelines/build-and-ci/release 等）；clippy.toml + deny.toml（依赖治理）
- CI：.github workflows（未深读）

## 外部依赖锚（Cargo.toml 根）
arrow 59.3 / datafusion 55.1 / aws-sdk-dynamodb 1.123 / aws-msk-iam-sasl-signer / moka 0.12 / opentelemetry(+otlp) 0.33 / prost 0.14 / prost-build / tonic（grpc）——托管服务形态（分布式日志+元数据+对象存储后端可换）

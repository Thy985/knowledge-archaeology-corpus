# Repository Snapshot Artifact — ARCH-2026-10-29-001

> 快照对象：hydro-project/infinity（浅克隆）；本文件所有事实可回溯到仓库实际内容（`file:line`）。

## 版本快照
| 字段 | 值 |
|---|---|
| repository | https://github.com/hydro-project/infinity.git |
| commit | `1abc6fcff1f186124f924b56aa92c3a3684c90c5`（HEAD） |
| branch | main |
| commit 信息 | "fix: enable aws-config sso feature so SSO profiles work (#176)" |
| commit 时间 | 2026-10-27 13:54:48 -0700 |
| license | Apache-2.0（LICENSE.txt；NOTICE 存在） |
| 版本 | 无版本号文件（workspace 多 crate，releases 为 sandbox-*/rap-* v0.2.0 等子 crate 独立 tag，10-22） |

## 项目基础地图
### 定位与规模
- **定位**（README.md）：Rust framework for massively concurrent agentic systems——"Infinity does for agents what async did for threads"
- **规模**：139 个非测试 .rs 文件 / 48,484 行；测试 .rs 33 个 / 11,782 行（测试/代码 ≈ 0.24x）
- **Cargo workspace**（Cargo.toml）：21 crates；resolver=2；rig-core 0.31（模型抽象）；AWS SDK（dynamodb/sqs/scheduler/dsql）+ sqlx（postgres）——本地双模型（rig/bedrock）+ AWS 原生存储

### 架构（crate 分层）
| 层 | crate | 行数 | 职责 |
|---|---|---|---|
| 核心 runtime | infinity-agent-core | 15,050 | Thread/Step/Slice/HistoryManager/事件处理器/内置工具（core/src 29 文件） |
| 本地驻留 | infinity-daemon | 7,490 | resident 模式（ws_handler/rap_callback/mcp_proxy/session_store/remote） |
| CLI | infinity-agent-cli | 7,195 | 交互 CLI |
| 沙箱 | sandbox-core/local/remote | 3,543/1,066/534 | stacked sandboxes（git.rs/jj.rs 集成 jujutsu） |
| 协议 | rap-protocol（392 行 lib.rs）+ rap-client/rap-bridge/rap-steering-server/rap-github-event-poller | 392/515/704/348/715 | Reactive Agent Protocol：网络工具调用不持连接 |
| 云部署 | infinity-agent-lambda | 1,894 | serverless step 模式（SQS 驱动单 slice） |
| 模型提供方 | infinity-provider-protocol/bedrock/rig | 1,280/1,350/504 | 模型抽象（rig-core 本地 + Bedrock） |
| 消息 | infinity-slack-bot/dataflow | 2,328/1,342 | Slack 集成 |
| 其他 | infinity-mcp-bridge（797）/infinity-protocol（613）/infinity-mdtests（100） | — | MCP 兼容层 / 内部协议 / doctest 注入 |

### 核心抽象（infinity-agent-core）
- **Thread**（system/thread.rs:96）：一个会话线程，拥有 HistoryManager；"one step prepares a batch of inputs, runs at most one completion round, commits results durably, dispatches at most one async tool call"（:113-118 注释）
- **StepOutcome**（thread.rs:34）：Skipped / Completed{usage, context_window}
- **Step 语义**（thread.rs:106-113 注释）：step 之间线程无 in-flight work——serverless 可整个丢弃，resident 可 park 在 channel；配置懒解析（不 boot 工具服务器）
- **HistoryManager**（event_processor.rs:141）：RefCell<Vec<InfinityMessage>> + processed_message_ids + turn_buffer；handle_content/handle_completion/flush_turn
- **DeferQueue trait**（system/defer.rs）：synthetic events 在 tool call in-flight 时推迟（防中断）；InMemoryDeferQueue（resident）/ NoDeferral（serverless SQS 直接中断语义）
- **SyntheticKind**（message.rs:41-135）：Tagged（SubscriptionEvent/ThreadReport/ParentMessage/Compaction/CompactionComplete）+ SubscriptionEvent；is_deferrable_synthetic_event（thread.rs:63）
- **stores**（stores.rs）：ConversationStore trait（load_history_up_to/append_messages/spawn_thread/close_thread/compaction 系列 14 方法）+ StateStore trait（get_processed_ids/add_processed_message_ids/metadata/subscriptions/user_choices）；InMemory 实现

### 生命周期（driver.rs:90-）
- drive_thread：一 driver 一 thread；批 channel 输入 → Thread::step → 用户文本到达中断 in-flight completion → synthetic events 在 tool pending 时 defer → 自动 compaction → 无工作可等时 Idle 退出（router 按需 respawn）
- InFlightStep（driver.rs:44）= future + cancel oneshot 一体（"impossible for them to desynchronize"）；cancel 路径 flush 已流式输出到 store（无部分 turn 丢失）
- ThreadLifecycleState：Live/Idle（driver.rs:73-80）

### serverless step 模式（infinity-agent-lambda）
- event_handler.rs:37 `function_handler(LambdaEvent<SqsEvent>)`；:120-122 "one SQS delivery = one slice per thread...step partitions by group and runs per-thread steps concurrently"；:174-178 单 batch=单 slice，`system.step(inputs, &collector, &mut NoDeferral)`
- **存储**：DynamoDbStateStore（state_store.rs：processed_message_ids String Set，get/add via DynamoDB，幂等去重）；Postgres conversation_history（conversation_history.rs:86 message_id VARCHAR(255) NOT NULL + INSERT ... message_data）
- **durable timers**（tools/sleep.rs）：WakeupScheduler——EventBridge Scheduler 一次性 schedule 指向 input queue，唤醒以工具结果消息回到 agent
- **工具**：rap_http / sleep / sqs_sender / toolset_cache

### RAP 协议（rap-protocol/src/lib.rs）
- RapInvocation/RapToolResult/RapSubscriptionEvent/RapUserChoice/RapViewUpdate/RapOAuth/RapCallback（:45-199）；ToolsetManifest/ToolDef（:213-224）；CallbackClient trait（:243）
- 定位（README）：serve tools over the network without holding a connection open——结果/订阅事件就绪即投递，agent 可无进程

### 治理与配置
- **AGENTS.md**：Conventional Commits 强制；`./check.bash` 通过才 finalize；"Never ignore errors (Result::Err)"；`.expect("...")` 带描述、逻辑 bug 前缀 "bug:"
- **check.bash**（CI 门）：format（cargo fmt --check）→ docs/ui prettier → clippy --all-targets -D warnings → cargo check --all-targets → cargo test --all-targets → doctests（infinity-mdtests include_mdtests）→ web e2e（playwright，可选）→ THIRD-PARTY 校验；前置：git + jj（jujutsu）+ bwrap/sandbox-exec
- **CI**：.github/workflows/{ci,conventional_commits,docs,release}.yml
- **配置**：rap-servers.example.json；daemon config.rs

### 外部依赖（workspace.dependencies 核心）
rig-core 0.31 / serde / serde_json / schemars / tokio / futures-util / async-trait / chrono / uuid / url / tracing / reqwest 0.13 / rhai（脚本）/ strkind / aws-sdk-{dynamodb,sqs,scheduler,dsql} / aws-config（sso feature——HEAD commit 刚启用）/ sqlx 0.8（postgres+json+chrono+uuid）

### 测试体系
- 单元测试：各 crate #[cfg(test)]（agent-core system/tests.rs + test_helpers.rs）
- 集成：agent-cli/tests、daemon/tests、provider-bedrock/tests、slack-dataflow/tests、sandbox-local/tests
- doctest：infinity-mdtests（include_mdtests 注入 docs 代码示例）
- 测试/代码比 0.24x（11,782/48,484）

# 01 · Project Layer — AWS Infinity

## 1. 项目定位

| 项 | 值 |
|---|---|
| 名称 | Infinity（Hydro 项目，AWS × UC Berkeley 合作） |
| 仓库 | https://github.com/hydro-project/infinity |
| 语言 | Rust（workspace，21 crates） |
| License | Apache-2.0 |
| 定位 | agent runtime + tool protocol，目标"大规模并发 agentic 系统" |
| 规模 | 57,187 行 Rust（168 .rs）+ 6,507 行 docs + 5,247 行 .snap 测试快照 |
| 默认分支 | main（HEAD `1abc6fc`，2026-10-27 "fix: enable aws-config sso feature so SSO profiles work (#176)"） |
| 生态位 | 与 Temporal/Restate/DBOS 同属 durable execution 品类，但执行模型不同（挂起-恢复 vs 步骤重放） |

## 2. 架构总览

```
┌────────────────────────────────────────────────────────────────────┐
│  Cargo workspace（21 crates，resolver="2"）                         │
│                                                                    │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────────────┐    │
│  │  core       │   │  protocol    │   │  providers           │    │
│  │ agent-core  │──▶│  RAP wire    │   │  provider-protocol   │    │
│  │ (15.4k)     │   │  rap-protocol│   │  provider-bedrock    │    │
│  │             │   │  rap-client  │   │  provider-rig        │    │
│  └─────┬───────┘   └──────────────┘   └──────────────────────┘    │
│        │                                                          │
│  ┌─────▼───────────┐   ┌───────────────────┐   ┌───────────────┐  │
│  │  Applications   │   │  Sandbox (编码)   │   │  RAP tools    │  │
│  │  agent-cli      │   │  sandbox-core     │   │  mcp-bridge   │  │
│  │  agent-lambda   │   │  sandbox-local    │   │  rap-bridge   │  │
│  │  daemon         │   │  sandbox-remote   │   │  rap-*        │  │
│  │  slack-bot      │   └───────────────────┘   └───────────────┘  │
│  │  slack-dataflow │                                             │
│  └─────────────────┘                                             │
└────────────────────────────────────────────────────────────────────┘
```

### 核心分层（low-level/overview.md 一手）

- **`infinity-agent-core`**：agent 循环（HistoryManager / prepare_input / run_completion / execute_action / built-in tools / 平台 traits / agent system API）
- **`infinity-provider-protocol`**：`ModelProvider` trait + 进程外 provider 传输（轻量，供 provider crates 单独依赖）
- **`rap-protocol`**：RAP wire types（RapInvocation / toolset manifests / display segments）
- **`rap-client`**：客户端 RAP 管道（HttpClient trait / ToolsetLoader / RapNotifier / 本地 callback server）
- **`infinity-agent-lambda`**：AWS 绑定（SQS handler / DSQL conversation store / DynamoDB state store）——生产 step-mode 参考实现
- **`infinity-daemon`**：常驻进程绑定（Infinity Code 用；in-memory + JSON-file stores + 按需启动 per-session RAP servers）——生产 local 参考实现

## 3. 核心模块（agent-core）

| 模块 | 职责 | 关键符号 |
|---|---|---|
| `event_processor.rs` | slice 三阶段 + HistoryManager | `prepare_input` / `run_completion` / `execute_action` / `HistoryManager`（1,933+ 行） |
| `message.rs` | 一切皆消息 | `InputMessage`（synthetic 标签 + group_id） |
| `stores.rs` | 平台 store traits | `ConversationStore` / `StateStore` / `InMemoryConversationStore` / `InMemoryStateStore` |
| `traits.rs` | 平台抽象 | `ConversationStore`（27）/ `StateStore`（223）/ `InputSender`（311） |
| `system/builder.rs` | 系统构建 | `AgentSystemBuilder<C,S,M,H=NoRapHttp>` / `new` / `new_local` / `build` / `start` / `start_with_observer` |
| `system/thread.rs` | 线程句柄 | `Thread<C,S,M,H>` / `replay_snapshot` / `pending_tool_call` / `step_no_defer` |
| `system/local/driver.rs` | 本地 driver | 自动 compaction（>75% context）/ 空闲回收 / 中断 |
| `system/defer.rs` | 事件暂存 | `DeferQueue` trait / `InMemoryDeferQueue` / `NoDeferral` |
| `system/observer.rs` | 事件观察 | `ThreadObserver`（on_event / on_subscribe） |
| `system/config.rs` | 静态线程配置 | `StaticThreadConfig` |
| `tools/thread.rs` | 线程工具 | `SpawnThreadTool` / `ReportToParentTool` / `CloseThreadTool` / `SendMessageToChildTool` |
| `tools/rap_tool.rs` | RAP 工具 | `RapTool` / `invoke_rap_tool` |
| `tools/sleep.rs` | 睡眠工具 | `SleepUntilEventOrInputTool` / `TokioSleepTool` / `TokioSleepUntilTool` |
| `tools/cancel_subscription.rs` | 订阅取消 | `CancelSubscriptionTool` |

## 4. 生命周期

### 一个 thread 的生命周期

```
launch ──▶ [slice: prepare → complete → dispatch] ──▶ yield（等 tool result）
              ▲                                             │
              └────────── 消息到达（新 slice）◀──────────────┘
              │
              ├──▶ idle（无消息，本地 driver 回收 task）
              ├──▶ compaction（历史 > 75% context）
              ├──▶ spawn_thread（产生 child thread）
              └──▶ close_thread（终止 + 可选 report）
```

### 一个 slice 的生命周期（architecture.md 一手）

```
Load:     HistoryManager::new_with_history（恢复历史 + 祖先链 + compaction 摘要 + processed IDs）
Prepare:  prepare_input（去重 / 丢 closed / 路由订阅事件 / 追加可行动内容）
Complete: run_completion（流式补全；sync tool loop-back；ContextOverflow 恢复）
Dispatch: execute_action（RAP = POST invocation + callback_url，fire-and-forget；sync tool 内联）
Persist:  turn buffer flush（turn 完成才提交；tool call 前先 sync）
Yield:    slice 结束，进程退出（Lambda）/ task 回到 channel（嵌入式）
```

## 5. 数据流与存储

| 数据 | 存储 | Lambda 实现 | daemon 实现 |
|---|---|---|---|
| 会话历史 + 线程树 + compaction 摘要 | `ConversationStore` | Aurora DSQL | in-memory + JSON 文件（~/.infinity/） |
| processed IDs + metadata + 活跃订阅 + pending choices | `StateStore` | DynamoDB（InfinityAgentsState） | in-memory + JSON |
| 输入消息 | `InputSender` | SQS FIFO（MessageGroupId=ThreadId） | mpsc channel |

## 6. 配置

- 顶层 `Cargo.toml`：workspace 21 members + resolver="2" + workspace.package（repository）
- `rap-servers.example.json`：RAP 服务器清单
- daemon：`config.rs` / `rap_servers.rs`；默认数据 `~/.infinity/sessions.json`（7 天清理）
- Lambda：env vars（`INPUT_QUEUE_URL` / `OUTPUT_QUEUE_URL` / `DSQL_CLUSTER_ENDPOINT` / `RAP_CALLBACK_URL` / `DELAY_QUEUE_URL` / `SCHEDULER_ROLE_ARN` / `TOOLS_CONFIG_*`）
- `clippy.toml`：lint 配置

## 7. 权限与治理

- **AGENTS.md**：Conventional Commits；`./check.bash` 必跑；"Never ignore errors"；`.expect("描述")`（逻辑 bug 前缀 "bug:"）
- **check.bash**：CI 测试入口
- **THIRD-PARTY**：license 白名单自动化生成（Rust: Apache-2.0/MIT/ISC/BSD-3/CC0/CDLA/MPL/Unicode/Zlib；npm: 同族）——CHANGELOG 记录
- **沙箱**：sandbox-core/local/remote 提供编码 agent 隔离（jj/git ModeProvider）
- **Lambda IAM**：serverless/architecture.mdx 提及（receiver/queue/scheduler 权限）
- **RELEASING.md**：发布流程（cargo-smart-release 镜像 hydro）

## 8. 入口

- **本地**：`infinity-agent-cli`（TUI）+ `infinity-daemon`（常驻进程，WS + RAP servers + MCP proxy）
- **Serverless**：`infinity-agent-lambda`（SQS 触发 handler `function_handler`）
- **编码 agent**：`infinity-agent-cli` + sandbox + slack-bot
- **示例**：`infinity-agent-core/examples/agent_scale.rs`（内存 benchmark）

## 9. 外部依赖

AWS SDK（sqs/dynamodb/dsql/scheduler/bedrock）、tokio 生态、serde/async-trait/thiserror、axum（HTTP）、MCP 兼容层（infinity-mcp-bridge）、chrono（timers）、insta（快照测试）、playwright（e2e）、vt100/alacritty（TUI 快照）

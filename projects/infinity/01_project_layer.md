# Project Layer — Infinity 项目地图

## 1. 定位
Rust framework for massively concurrent agentic systems（README.md）——"Infinity does for agents what async did for threads"：agent 等待工具时不阻塞，并发运行、让出、零成本等待下一事件。

## 2. 架构总览（crate 分层）
```
                      ┌─────────────────────────────────────────────┐
                      │              Cargo workspace（21 crates）   │
   Agent Runtime      │  infinity-agent-core（15,050 行）           │
   ──────────────     │   ├─ Thread/Step/Slice 执行模型              │
   Resident           │   │  ├─ event_processor（HistoryManager）    │
   ──────────────     │   │  ├─ stores（Conversation/State trait）   │
   infinity-daemon    │   │  └─ tools（thread/rap/sleep/cancel）     │
   infinity-agent-cli │  ├─ infinity-daemon（resident ws/mcp/rap）   │
                      │  ├─ infinity-agent-cli（交互）               │
   Serverless         │  └─ infinity-agent-lambda（SQS step 模式）   │
   ──────────────     │                                              │
   Model Providers    │  ├─ infinity-provider-protocol/rig/bedrock   │
   ──────────────     │                                              │
   Sandbox            │  ├─ sandbox-core/local/remote                │
   ──────────────     │                                              │
   Protocol           │  ├─ rap-protocol/client/bridge/steering      │
                      │  ├─ infinity-mcp-bridge                      │
                      │  └─ infinity-slack-bot/dataflow              │
                      └─────────────────────────────────────────────┘
```

## 3. 核心模块（infinity-agent-core/src）
| 模块 | 文件 | 职责 |
|---|---|---|
| Thread | system/thread.rs（724） | 一个会话线程；StepOutcome；step 执行（prepare→completion→commit→dispatch）；pending_tool_call/awaiting_tool_result/replay_snapshot；filter_deferrable |
| HistoryManager | event_processor.rs（5209） | 历史管理：turn_buffer 流式合并、unvalidated/known_safe 两级持久化、processed_message_ids 幂等、中断合成、completion 循环（run_completion） |
| 事件分派 | system/local/{driver,router,sender,handle,launch}.rs | 每线程 driver 循环；router FuturesUnordered 池；Live/Idle 生命周期 |
| 存储 trait | stores.rs（768） | ConversationStore（load_history_up_to/append/spawn/close/compaction 14 方法）+ StateStore（processed_ids/metadata/subscriptions/choices） |
| Defer | system/defer.rs | DeferQueue trait：InMemoryDeferQueue（resident）/ NoDeferral（serverless） |
| 工具 | tools/{thread,rap_tool,sleep,cancel_subscription,config}.rs | spawn_thread/report_to_parent/close_thread/send_message_to_child；RAP 工具；sleep |
| 消息模型 | message.rs | InputMessage/SyntheticKind（Tagged 5 类 + SubscriptionEvent） |

## 4. 核心数据结构
- `Thread<C,S,M,H>`（thread.rs:96）：history: HistoryManager + pending_choices + in_flight + current_thinking
- `HistoryManager`（event_processor.rs:141）：thread_id/root_thread_id/history RefCell/processed_message_ids/unvalidated_items/pending_items/turn_buffer/interrupted_tool_calls
- `StepOutcome`（thread.rs:34）：Skipped | Completed{usage, context_window}
- `InputMessage`（message.rs）：content + synthetic: Option<SyntheticKind> + message_id
- `SyntheticKind`（message.rs:41-135）：Tagged{SubscriptionEvent,ThreadReport,ParentMessage,Compaction,CompactionComplete} | SubscriptionEvent；is_associative/is_final
- `ThreadLifecycleState`（driver.rs:73）：Live/Idle
- `DeferQueue`（defer.rs）：push/drain

## 5. 生命周期
- **resident**：router 按需 spawn driver → driver 批处理输入 → step → 无工作可等 → Idle（driver 退出，内存释放）→ 新输入到达 → respawn（router.rs:177-178）
- **serverless**：SQS 消息 → Lambda handler → system.step（单 slice）→ 输出入队 → 进程退出（event_handler.rs:37,174-178）
- **订阅**：active_subscriptions 不保持 driver 驻留——事件 respawn（driver.rs:358-363 注释）

## 6. 状态机
```
（线程状态，非显式 enum，由 HistoryManager 字段组合表达）
  unvalidated_items  →  mark_inputs_model_validated →  pending_items（committed）
  turn_buffer        →  flush_turn →  history（append_known_safe）→  sync 持久化
  pending tool call  →  [用户文本] →  interrupted（合成结果）|  [结果] →  AlreadyAnswered 判重
  driver: Live ⇄ Idle（respawn on demand）
```

## 7. 测试体系
- 单元：各 crate #[cfg(test)]（agent-core system/tests.rs 覆盖 driver/compaction/订阅 respawn）
- 集成：cli/daemon/bedrock/slack-dataflow/sandbox-local tests
- doctest：infinity-mdtests（include_mdtests 把 docs 代码示例转 doctest）
- CI 门：check.bash（fmt→clippy -D warnings→check→test→doctest→web e2e→THIRD-PARTY）

## 8. 配置
- Cargo.toml workspace deps（rig-core/aws-sdk/sqlx）
- rap-servers.example.json（RAP 工具服务器）
- daemon config.rs（resident 配置）
- Lambda 环境变量：INPUT_QUEUE_URL/ARN、DELAY_QUEUE_URL、RAP_CALLBACK_URL、MODEL_ID（event_handler.rs:104-152）

## 9. 权限与治理
- AGENTS.md：Conventional Commits；check.bash 通过才 finalize；"Never ignore errors (Result::Err)"；`.expect()` 带描述，逻辑 bug 前缀 "bug:"
- check.bash：格式/clippy -D warnings/全 target 编译/全测试/doctest/可选用例 web e2e + THIRD-PARTY 校验；前置 jj（jujutsu）+ bwrap/sandbox-exec
- CI：ci.yml / conventional_commits.yml / docs.yml / release.yml

## 10. 外部依赖
rig-core 0.31 / serde / tokio / reqwest 0.13 / rhai / aws-sdk-{dynamodb,sqs,scheduler,dsql} / aws-config（sso feature，HEAD 刚启用）/ sqlx 0.8（postgres）/ chrono / uuid / async-trait

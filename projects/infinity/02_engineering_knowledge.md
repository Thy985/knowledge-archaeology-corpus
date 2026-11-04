# Engineering Knowledge Graph — Infinity（24 条，links 六类边）

> 每条 EK 保留证据来源（file:line）；边类型：mechanism 同机制 / subsystem 同子系统 / causal 因果 / dependency 依赖 / constraint 约束 / contrast 对照。

## EK-D01 Thread::step 单一 completion + 至多一个异步工具
step 准备一批输入、运行至多一次模型补全、持久化结果、分派至多一个异步工具调用；step 之间线程无 in-flight work。
- [code] crates/infinity-agent-core/src/system/thread.rs:106-118, 500-620
- [code] crates/infinity-agent-core/src/system/thread.rs:34（StepOutcome）
- links: subsystem→D05,D09,D10,D14；causal→D02

## EK-D02 Commit barrier：turn 持久化先于外部反应
`history.sync()` 在分派前完成——"the call is durable in history before its result can possibly arrive"。对同步工具同样成立：执行前先 sync（"we must sync the history so that thread spawning uses the correct state"，event_processor.rs:1838）。
- [code] crates/infinity-agent-core/src/system/thread.rs:579-590
- [code] crates/infinity-agent-core/src/event_processor.rs:1838
- links: causal→D03；constraint→D01；mechanism→D14

## EK-D03 Dispatch barrier + 失败入队 error result
工具分派失败 → send_tool_error 入队 error ToolResult（带原始 tool/call IDs）→ agent 恢复而非永久等待。未知工具 → `{id}-unknown-tool` 合成错误结果 + loop back（event_processor.rs:1824-1835）。
- [code] crates/infinity-agent-core/src/event_processor.rs:1960-1990
- [code] crates/infinity-agent-core/src/event_processor.rs:1824-1835
- [code] crates/infinity-agent-core/src/system/thread.rs:593-606
- links: causal→D02；dependency→D01

## EK-D04 同步工具 loop-back（should_loop_back）
同步工具在同一 completion 内立即返回结果并 `continue 'outer` 再跑一轮；异步工具才走 ExecuteToolCall action。
- [code] crates/infinity-agent-core/src/event_processor.rs:1855-1900
- links: mechanism→D01；contrast→D03

## EK-D05 消息 ID 幂等去重（processed_message_ids）
handle_content 先查 processed_message_ids，重复即 skip；新消息 append 后立即 insert。内存 + DynamoDB String Set 双实现。
- [code] crates/infinity-agent-core/src/event_processor.rs:244-289
- [code] crates/infinity-agent-lambda/src/state_store.rs:48-80
- links: subsystem→D06,D09；mechanism→D16；constraint→D01

## EK-D06 工具结果反向走查（Unanswered/AlreadyAnswered/NoPendingCall）
match_tool_result 从 history 尾反向走查，只跨 tool call/result；用户文本/assistant 内容是 turn boundary——之前的调用已结算，结果必 stale。
- [code] crates/infinity-agent-core/src/event_processor.rs:300-330
- links: mechanism→D05；causal→D07；constraint→D03

## EK-D07 中断 = 合成 interrupted 结果
用户文本到达 → interrupt_pending_tool_call 注入合成 ToolResult（TOOL_CALL_INTERRUPTED_TEXT），消息 ID `{tool_call_id}-interrupted`；take_interrupted_tool_calls 供 driver 通知 RAP 服务器 + 清除 pending user choice（持久化状态转换先完成）。
- [code] crates/infinity-agent-core/src/event_processor.rs:322-355, 754-757
- [code] crates/infinity-agent-core/src/system/thread.rs:437-460
- links: causal→D06；dependency→D10；subsystem→D08

## EK-D08 未验证输入不持久化（unvalidated/known_safe 两级）
append_unvalidated 先于模型输出；mark_inputs_model_validated（模型输出=验证证明）转 known_safe；上下文溢出时 drop_unvalidated_user_inputs（只保留 tool result/subscription，用户输入删除并遗忘 dedup ID 防 wedge）。
- [code] crates/infinity-agent-core/src/event_processor.rs:468-560
- links: mechanism→D07；constraint→D02；subsystem→D05

## EK-D09 turn_buffer 流式合并 + flush_turn
多 chunk 文本合并为单 buffer 条目（防一行一行的磁盘爆炸）；flush 在完成 turn/turn-ending tool call；flush_turn_trimming_reasoning 去尾部 reasoning。
- [code] crates/infinity-agent-core/src/event_processor.rs:358-440
- links: subsystem→D05；causal→D01

## EK-D10 DeferQueue 平台契约
synthetic events（订阅/子线程报告/父消息）在 tool call pending 时推迟防中断；InMemoryDeferQueue（resident，进程死丢）vs NoDeferral（serverless，接受中断语义）。
- [code] crates/infinity-agent-core/src/system/defer.rs:1-76
- links: constraint→D07；contrast→D11；subsystem→D15

## EK-D11 Driver 生命周期 Live/Idle 同点报告
active_threads 集合与 lifecycle 事件在同一位置变更——"the channel and the set can never disagree"。
- [code] crates/infinity-agent-core/src/system/local/driver.rs:90-130
- links: mechanism→D12,D13；causal→D11→D12

## EK-D12 check_before_idle 三态决策
Continue（rx 有输入）/ AwaitInput（等工具结果）/ Exit（无工作——订阅不保持驻留，事件 respawn）；嵌入式可据订阅状态保活关联资源。
- [code] crates/infinity-agent-core/src/system/local/driver.rs:331-363
- links: causal→D11；constraint→D13；subsystem→D15

## EK-D13 Router FuturesUnordered 池 + on-demand respawn
router 直接拥有 driver futures（非每线程一 task），driver idle 内存即时释放；线程收第一消息 spawn、idle 后收新消息 respawn。
- [code] crates/infinity-agent-core/src/system/local/router.rs:1-8, 177-178
- links: mechanism→D12；subsystem→D11

## EK-D14 InFlightStep future+cancel 一体 + cancel flush
InFlightStep 把 future 与 cancel oneshot 放一个值——"impossible for them to desynchronize"；cancel 路径 flush 已流式内容到 store（无部分 turn 丢失），无输出输入留内存下次重发。
- [code] crates/infinity-agent-core/src/system/local/driver.rs:44-61
- links: mechanism→D02；causal→D09

## EK-D15 Lambda SQS 单 delivery = 单 slice
batchSize:1 默认单消息；FIFO 大 batch 时 step 按 group partition、per-thread 并发；"One SQS delivery = one slice per thread in the batch"；NoDeferral 无处存 defer。
- [code] crates/infinity-agent-lambda/src/event_handler.rs:120-128, 174-178
- links: subsystem→D16,D17；constraint→D10；mechanism→D01

## EK-D16 持久化双写（DynamoDB 幂等 + Postgres history）
DynamoDbStateStore：processed_message_ids String Set（get/ADD）；conversation_history：Postgres 表 message_id VARCHAR(255) NOT NULL + INSERT message_data。
- [code] crates/infinity-agent-lambda/src/state_store.rs:1-80
- [code] crates/infinity-agent-lambda/src/conversation_history.rs:86, 249-275
- links: subsystem→D15；mechanism→D05

## EK-D17 EventBridge durable timers（sleep 工具）
WakeupScheduler：EventBridge Scheduler 一次性 schedule 指向 input queue，唤醒以工具结果消息回到 agent；sleep_until_event_or_input 在输出转换中特判不打印。
- [code] crates/infinity-agent-lambda/src/tools/sleep.rs:42-122
- [code] crates/infinity-agent-lambda/src/event_handler.rs:210-220（特判）
- links: subsystem→D15；contrast→D18

## EK-D18 RAP：网络工具调用不持连接
Reactive Agent Protocol：工具服务器回调（RapCallback）就绪即投递结果/订阅事件，agent 可无进程；MCP 服务器经兼容层运行。
- [code] crates/rap-protocol/src/lib.rs:45-243
- [code] crates/infinity-agent-core/src/tools/rap_tool.rs:136-180
- links: contrast→D17；dependency→D15；subsystem→D19

## EK-D19 spawn_thread fresh context + report_to_parent/close_thread
spawn_thread 仅同步执行（safe_spawn_point 守卫），子线程 FRESH context 不继承父 history；report_to_parent 送结果；close_thread 前必须 cancel 订阅。
- [code] crates/infinity-agent-core/src/tools/thread.rs:112-176, 345-430
- links: subsystem→D20；constraint→D10；mechanism→D07

## EK-D20 订阅事件 associative 内联 vs 临时子线程
is_associative：关联事件内联注入订阅线程 history（如 thread reports）；否则走临时子线程不污染父 context；is_final 移除订阅跟踪。
- [code] crates/infinity-agent-core/src/message.rs:116-135
- links: subsystem→D19；constraint→D10；causal→D20→D19

## EK-D21 自动 compaction（历史压缩）
driver 检测 is_compaction_complete 合成消息；stores.rs ConversationStore 提供 mark_thread_as_compaction/is_compaction_thread/save_compaction_summary/load_latest_compaction_summary；context_window 供阈值。
- [code] crates/infinity-agent-core/src/system/local/driver.rs:81-84, 506-536
- [code] crates/infinity-agent-core/src/stores.rs:353-406
- links: constraint→D01；causal→D08；subsystem→D05

## EK-D22 模型解析 fallback
StaticModel::new 失败 → warn + StaticModel::from_entry 最小 entry（context_window 0 / 无 image）继续——不因模型元数据解析失败中断 step。
- [code] crates/infinity-agent-lambda/src/event_handler.rs:152-168
- links: dependency→D01；subsystem→D15

## EK-D23 配置懒解析（不 boot 工具服务器）
Thread 配置（tools/prompt/notifier）在首个 step 才 resolve——"loading a thread for replay or subscription does not touch (or boot) its tool servers"。
- [code] crates/infinity-agent-core/src/system/thread.rs:111-118
- links: constraint→D01；mechanism→D12

## EK-D24 AGENTS.md 治理 + check.bash 质量门
Conventional Commits 强制；"Never ignore errors (Result::Err)"；expect 带描述 + "bug:" 前缀；check.bash 五层（fmt/clippy -D warnings/check/test/doctest）+ web e2e + THIRD-PARTY。
- [code] AGENTS.md:1-34
- [code] check.bash:1-110
- links: constraint→全部（治理约束基底）；contrast→D13

## EK Graph 统计
- EK 24 条；总边 66；平均出边 2.75；游离 0；聚合规则覆盖率 100%（见 03_ko.md）

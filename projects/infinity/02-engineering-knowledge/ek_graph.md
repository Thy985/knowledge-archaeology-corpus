# 02 · Engineering Knowledge Graph — AWS Infinity

> **EK Graph 说明**：每条 EK 是图的节点，声明 `links`（6 类边）；KO 从图上按 R1-R4 聚合（见 03）。
> 证据符号：`docs/…` = docs/docs/infinity-runtime/ 下文件；`src/…` = crates/ 下文件；`CHANGELOG` = agent-core/CHANGELOG.md；`commit:sha` = git 提交。

## EK 清单（45 条）

### A. Slice 生命周期（核心执行单元）

**EK-01 · Slice 三阶段管线**
slice = `prepare_input` → `run_completion` → `execute_action` 三函数组合；高层的 `AgentSystem::step` 与 `LocalAgentSystem::start` 都是同一管线的封装。
- links: `mechanism→EK-02`（都是 slice 结构）；`subsystem→EK-03/04/05`；`causal→EK-02`
- 证据: `src/infinity-agent-core/src/event_processor.rs:889,1480,1933`；`docs/architecture.md`；`docs/low-level/completion-loop.md`

**EK-02 · "execution slice never blocks on anything external" 不变量**
核心不变式：slice 内不等待任何外部操作；模型补全 + HTTP + 存储调用后 slice 结束。
- links: `constraint→EK-01`（定义 slice 边界）；`causal→EK-09`（使 serverless 可行）；`mechanism→EK-03`（同步工具 loop-back 是例外机制）
- 证据: `docs/architecture.md` 首句

**EK-03 · Synchronous tools loop-back（同步工具回环）**
`Tool::execute_synchronous` 内联执行 + 结果直接注入历史 + 同 slice 内再跑一次 completion；避免 spawn_thread 类微秒调用的 store 往返与竞态（并发事件可能让调用显得被取消）。
- links: `contrast→EK-04`（同步 vs 异步派发）；`causal→EK-01`（不结束 slice）；`mechanism→EK-10`（spawn_thread 用此机制）
- 证据: `docs/architecture.md#synchronous-tools-loop-back`；`src/infinity-agent-core/src/tools/thread.rs:65`

**EK-04 · Async RAP tool dispatch（异步派发）**
RAP 工具 = 单次 HTTP POST（含 callback_url）+ 200 确认即返回；结果经 input queue 重新进入（新 slice）。未知工具名走 loop-back 注入错误结果（幻觉工具调用只多花一次补全）。
- links: `contrast→EK-03`；`subsystem→EK-28`（RAP 协议）；`causal→EK-05`（turn flush 后才 dispatch）
- 证据: `docs/architecture.md`；`src/infinity-agent-core/src/tools/rap_tool.rs:72`

**EK-05 · Turn buffer 持久化时序（先 flush 后 dispatch）**
流式输出缓冲在 turn buffer，turn 完成才提交；工具调用结束 turn 时先 flush buffer 再 dispatch（保证调用在结果可能到达前已 durable）。stream 中途错误丢弃缓冲重试（half-streamed 永不落库）。
- links: `constraint→EK-04`（dispatch 前置条件）；`causal→EK-06`（落库后幂等才有意义）；`mechanism→EK-12`（sync 断言 turn buffer 空）
- 证据: `docs/architecture.md#turns-and-durability`；`docs/low-level/history-manager.md`；`src/event_processor.rs:407-445`；`commit:a84b99e`（"don't commit turn data until the turn is completed"）

**EK-06 · 幂等去重（StateStore processed IDs + history tail）**
用户文本/订阅事件（非天然幂等）按消息 ID 记入 StateStore；工具结果（天然幂等）按 history tail 去重（回看尾部 tool call/result 对）。持久化与消息同一 sync 完成 → crash 重处理而非重复。
- links: `mechanism→EK-07`（FIFO 是配套排序保证）；`causal→EK-05`；`constraint→EK-09`（serverless at-least-once 安全）
- 证据: `docs/low-level/history-manager.md#sync-and-deduplication`；`src/traits.rs:223`（StateStore 注释）

**EK-07 · FIFO per thread（组内有序 + 跨线程并发）**
排序要求直接编码进 queue：消息按 group_id 分组，传输层保证组内 FIFO（AWS=SQS FIFO MessageGroupId；嵌入式=mpsc + per-thread worker）。跨线程无共享状态 → 并行。运行时无分布式锁。
- links: `constraint→EK-06`（幂等 + FIFO 共同保证 slice 串行）；`causal→EK-10`（spawn 新线程 = 新 message group）
- 证据: `docs/architecture.md#ordering`；`docs/low-level/overview.md#inputsender`；`src/infinity-agent-lambda/src/event_handler.rs`（MessageGroupId 注释）

**EK-08 · 消息全同质（Everything is a message）**
用户文本/工具结果/订阅事件/子报告/OAuth 挑战/timer 唤醒都是 `InputMessage`（synthetic 标签 + group_id 区分）。slice 间无进程内状态机追踪"等待什么"——等待的东西最终以消息到达。
- links: `mechanism→EK-01`（slice 只消费一种输入）；`causal→EK-02`（是"可完全关闭"的前提）
- 证据: `docs/architecture.md#everything-is-a-message`；`src/infinity-agent-core/src/message.rs`

### B. 线程与并发

**EK-09 · 挂起=纯数据（serverless 可行的根因）**
等待中的 agent 无 task/stack/连接，只有历史 → 可完全退出进程，scale-to-zero 是执行模型的推论。架构满足 serverless 四约束：有界执行/无进程亲和/at-least-once 安全/无锁并发。
- links: `causal→EK-02`（slice 不阻塞 → 可退出）；`constraint→EK-06/07`（幂等+FIFO 是 serverless 正确性前提）；`mechanism→EK-20`（sleep 中断）
- 证据: `docs/architecture.md#why-this-runs-on-serverless`；`CHANGELOG`（50k agents 测量）

**EK-10 · spawn_thread = 同步工具 + 新 message group**
spawn 在微秒级完成 → 用 `execute_synchronous` 内联执行；子线程 = 新消息组（独立并行 slice）。子继承父历史至 spawn 点 + 指令。
- links: `mechanism→EK-03`（同步回环实例）；`causal→EK-07`（新 group）；`constraint→EK-11`（child_of 校验）
- 证据: `src/infinity-agent-core/src/tools/thread.rs:65-140`；`docs/built-in/threading.md`

**EK-11 · child_of 栈校验（防子线程误执行父计划）**
`child_of` 参数必须等于实际 thread stack；不匹配 → 返回错误（"Do NOT spawn threads — focus on your assigned task"）。因为子继承父上下文（含 spawn 计划），可能混淆。
- links: `constraint→EK-10`；`mechanism→EK-13`（fresh_context 是另一种防污染手段）
- 证据: `src/infinity-agent-core/src/tools/thread.rs:68-95`

**EK-12 · fresh_context 空历史**
`fresh_context: true` 产生空历史子线程（只看 spawn call + 指令）；freshness 是线程属性（子孙也看不见 fresh 边界前内容）。load_history_with_ancestors 在 fresh 边界截断。
- links: `contrast→EK-11`（两种防污染策略）；`mechanism→EK-17`（compaction 摘要也按此重建）
- 证据: `src/traits.rs:27`（load_history_with_ancestors 注释）；`docs/built-in/threading.md#fresh-context-threads`；测试 `fresh_context_spawn_has_no_parent_history`（tests.rs:1516）

**EK-13 · 子报告 = synthetic tool result（无独立消息系统）**
`report_to_parent`/`close_thread` 经 input queue 以事件到达父 → synthetic tool result 引用原 spawn call。运行时不需要独立 agent 消息系统。
- links: `mechanism→EK-18`（synthetic 机制复用）；`causal→EK-07`
- 证据: `docs/built-in/threading.md#reporting-back`；`src/infinity-agent-core/src/tools/thread.rs:222-345`

### C. Synthetic events 与订阅

**EK-14 · Synthetic tool calls（receive_event__injected）**
事件到达 → runtime 添加模型未发起的 assistant tool call `receive_event__injected`（含 original_tool_name/call_id/args）+ tool result 含事件。保留模型 post-training 的工具结果解读能力；`__injected` 后缀标识 runtime 创建；模型不可主动调用（runtime 拒绝）。
- links: `mechanism→EK-13`（子报告复用同一机制）；`contrast→EK-15`（inline vs threaded）；`causal→EK-16`
- 证据: `docs/rap/about/subscription-events.md#synthetic-tool-calls`；`src/event_processor.rs:1309`（input_event 中 synthetic 处理）；AWS blog

**EK-15 · 订阅事件路由：默认临时子线程 / associative 内联**
非 associative 事件：prepare_input 不追加到订阅线程历史，而是 spawn 临时 child（继承父上下文 + 事件）处理，父只收 report 或什么都不收 → 父上下文干净。associative 事件：直接注入订阅线程历史（如长命令的流式日志）。
- links: `contrast→EK-14`（两种注入位置）；`causal→EK-10`（临时 child = spawn）；`mechanism→EK-13`
- 证据: `docs/architecture.md#subscription-events`；`docs/rap/about/subscription-events.md#inline-vs-threaded`；测试 `subscription_event_queued_during_completion_processed_after`（tests.rs:271）

**EK-16 · Deferral（等待期间事件暂存）**
等待非被动工具调用时，订阅事件/子报告被 hold back（避免中断 pending call → "interrupted" 注入 + 真结果被丢）。`DeferQueue` trait：`NoDeferral`（无处可放，Lambda）/ `InMemoryDeferQueue`（内存，本地 driver）/ 自定义 durable（DB/delay queue）。
- links: `constraint→EK-14`（synthetic 事件何时可注入）；`contrast→EK-02`（中断 vs 不中断语义）；`mechanism→EK-19`
- 证据: `src/infinity-agent-core/src/system/defer.rs` 全文；`docs/agent-systems/step-mode.md#deferral`

**EK-17 · Compaction（~3/4 context window 触发）**
历史接近 3/4 context window → 本地 driver spawn compaction 线程 → 摘要存 ConversationStore（按 history index 标记）→ 后续 slice 加载摘要 + index 后消息。子线程按位置重建继承历史。compaction 完成消息回环后重置触发标志（防重复触发）。
- links: `causal→EK-12`（重建机制）；`constraint→EK-21`（ContextOverflow 恢复）；`mechanism→EK-17b`
- 证据: `docs/architecture.md#compaction`；`src/system/local/driver.rs:384-385`（"> 75% full"）；测试 `compaction_during_tool_call_corrupts_history`（tests.rs:790）、`compaction_does_not_retrigger_after_applied`（tests.rs:875）

**EK-17b · compaction 与 child 的相互作用**
compaction 中 child 可能已 spawn（按位置重建）；compaction 后 child 报告/父消息仍可达（synthetic 引用 resolve 到 store）。修复史：`compaction_inside_child_thread_does_not_panic`（tests.rs:1030）；commit:cee7fa7（"seed spawned children with their own spawn call; resolve synthetic events past compaction"）。
- links: `mechanism→EK-17`；`causal→EK-14`
- 证据: `src/event_processor.rs`（find_tool_call 注释：可折叠进 summary 仍从 store resolve）；`CHANGELOG`；测试 `child_close_report_wakes_sleeping_parent_after_compaction`（tests.rs:1978）

### D. 错误处理与恢复

**EK-18 · Provider 错误分类（ErrorClass）**
provider 自声明错误类别：`Transient`（短退避重试）/ `Throttled`（长退避）/ `ContextOverflow`（收缩输入，重试无效）/ `Fatal`（不重试）。运行时绝不解析错误字符串。
- links: `mechanism→EK-19`（分类驱动恢复策略）；`constraint→EK-20`（ContextOverflow 特殊恢复）
- 证据: `src/infinity-provider-protocol/src/completion.rs:62-90`；`docs/low-level/completion-loop.md`

**EK-19 · ContextOverflow 恢复（永不 wedge）**
overflow 时：若唯一新输入是单个 tool result/订阅事件 → 用占位符替换其 body 并重试（调用必须被回答）；否则丢新用户输入（pending tool result 标记 interrupted）并结束流。`TOOL_RESULT_TOO_LARGE_PLACEHOLDER` / `TOOL_CALL_INTERRUPTED_TEXT` 常量。
- links: `causal→EK-18`；`constraint→EK-17`（compaction 是长期解）；测试锚 `oversized_tool_result_never_wedges_the_thread`（tests.rs:1238）
- 证据: `src/event_processor.rs:572`（replace_unvalidated_tool_results）、`src/event_processor.rs` 占位符常量；`docs/low-level/completion-loop.md`

**EK-20 · 中断语义（interrupt vs 不中断）**
用户文本中断进行中 completion（非用户消息不中断，等待当前 turn）；sleep 工具 passive → 用户消息/订阅事件立即处理，pending sleep 结果稍后追加。`<interrupt>` 前缀剥离。
- links: `mechanism→EK-16`（deferral 是相反策略）；`causal→EK-19`
- 证据: `src/system/thread.rs:52`（is_user_text_input）；`docs/built-in/built-in-tools.md#sleep-tools`；测试 `user_text_interrupts_active_completion`（tests.rs:188）

**EK-21 · 重启/崩溃恢复（rebooted session settles persisted unanswered tool call）**
持久化的未回答 tool call 在重启后 settle（结果到达时 attach）；interrupt 前无输出不落库。测试 `rebooted_session_settles_persisted_unanswered_tool_call`（tests.rs:1335）、`shutdown_persists_model_validated_tool_result`（tests.rs:453）、`shutdown_does_not_persist_unvalidated_tool_result`（tests.rs:517）。
- links: `mechanism→EK-05`；`causal→EK-06`；`constraint→EK-09`
- 证据: `tests.rs`；`commit:c2aab9e`（"persist in-flight tool results when a session is stopped"）

### E. 平台可移植

**EK-22 · 四平台 trait（可移植执行模型）**
`ConversationStore` / `StateStore` / `InputSender` / `HttpClient` 四 trait 抽象全部平台差异；core 其余部分（turn 管理/yielding/threading/subscriptions/compaction/dedup）平台无关。`InputSender` 是最小但最关键的 trait（定义 yield 边界）。
- links: `mechanism→EK-23/24`（两个生产实现）；`constraint→EK-09`
- 证据: `docs/low-level/overview.md#the-platform-traits`；`src/traits.rs`

**EK-23 · Lambda = 生产 step-mode 参考实现**
`infinity-agent-lambda/src/event_handler.rs`：每次 invocation 构建 stores（DSQL+DynamoDB）、Bedrock model、tools、sender、system → `system.step(inputs, &collector, &mut NoDeferral)` → 事件经 EventCollector 转 output queue。无 runtime task 存活。
- links: `mechanism→EK-22`；`contrast→EK-24`；`causal→EK-25`（SQS sender loopback）
- 证据: `src/infinity-agent-lambda/src/event_handler.rs`；`docs/agent-systems/step-mode.md#the-lambda-embedding`

**EK-24 · daemon = 生产 local 参考实现**
常驻进程：in-memory + JSON stores、per-session RAP servers 按需启动、WS 客户端、MCP proxy。`keeps_session_alive` flag 防非交互客户端阻塞 idle shutdown（commit:1b20fda）。
- links: `mechanism→EK-22`；`contrast→EK-23`；`subsystem→EK-26`（session 持久化）
- 证据: `src/infinity-daemon/src/`；`CHANGELOG`；commit:1b20fda

**EK-25 · InputSender 即回环（serverless loopback）**
step-mode 中所有"稍后发生"的事（child seed / report / timer）经 `send_to_input_queue` 而不是直接调用；Lambda = SQS 客户端指向同一 FIFO 队列 → 未来投递。这是 slice 可自由结束的原因。
- links: `causal→EK-23`；`constraint→EK-02`；`mechanism→EK-27`（durable timers 走此路径）
- 证据: `docs/agent-systems/step-mode.md#the-sender-is-the-loopback-path`；`src/infinity-agent-lambda/src/tools/sqs_sender.rs`

**EK-26 · Session 持久化与清理（daemon）**
daemon 默认数据 `~/.infinity/sessions.json`；session 持久化 7 天清理（CHANGELOG + memory_store.rs）。`session_id` → `root_thread_id` 重命名（commit:d915f23）。
- links: `mechanism→EK-24`；`contrast→EK-23`（本地 vs 云持久化）
- 证据: `src/infinity-daemon/src/session/mod.rs:52,156`；`src/infinity-daemon/src/memory_store.rs`；commit:d915f23

**EK-27 · Durable timers（serverless 双机制）**
sleep ≤900s → SQS DelaySeconds（relay queue）；>900s → 一次性 EventBridge schedule → 直接入 input queue。均可中断（用户消息/订阅事件优先，pending sleep 结果稍后追加）。
- links: `causal→EK-25`；`contrast→EK-28`（嵌入式 tokio timer）
- 证据: `src/infinity-agent-lambda/src/tools/sleep.rs`（WakeupScheduler）；`docs/infinity-runtime/serverless/architecture.mdx#durable-timers`

### F. RAP 协议

**EK-28 · RAP = 异步工具协议（open spec）**
Reactive Agent Protocol：runtime↔tool 异步 HTTP。`POST /invoke` → 200 确认（响应体不读）→ tool 完成后 POST callback_url（tool_result/oauth/subscription_event/user_choice）。借鉴 MCP，扩展异步执行/长时操作/事件订阅。
- links: `mechanism→EK-04`；`contrast→EK-29`（MCP 兼容）；`subsystem→EK-30`（协议类型）
- 证据: `docs/rap/spec/overview.md`；`docs/rap/spec/basic/tool-invocation.md`；`docs/rap/about/architecture.md`

**EK-29 · MCP 适配层（bridge，机制经盲重建修正）**
`infinity-mcp-bridge`（797 行）：`McpToolSet` 把 MCP 服务器（stdio 或 Streamable HTTP，lazily connected）**暴露为本地 Infinity tools**；`McpClient` 是共享传输层（供通过另一协议暴露同一 MCP session 的适配器使用）。即 MCP 服务器以**工具身份**接入 agent-core，而非以 RAP 协议服务器身份接入（修正：非"协议兼容直跑"）。
- links: `contrast→EK-28`（MCP=本地工具 vs RAP=异步协议）；`subsystem→EK-28`
- 证据: `src/infinity-mcp-bridge/src/lib.rs:1-40`（McpToolSet/McpClient 文档）；README

**EK-30 · RAP wire 类型（strkind）**
`ThreadId`（=RAP group_id；本地 UUID；Lambda=调用者选定的 conversation key 兼 SQS MessageGroupId）/ `ToolCallId`（provider stream 或 minted UUID）/ `ProviderCallId`（echo 原样）；`ChoiceId` 目前是 ToolCallId 别名（issue #109 跟踪）。
- links: `mechanism→EK-07`（ThreadId=group_id）；`subsystem→EK-28`
- 证据: `src/rap-protocol/src/lib.rs`；commit:28026f5（"type tool-call IDs"）

**EK-31 · 订阅生命周期（active subscription 追踪）**
tool result 带 `"subscription": true` → runtime 记入 StateStore active subscriptions；`cancel_subscription` 通知 server + 移除追踪；close_thread 前应先取消。
- links: `causal→EK-15`；`mechanism→EK-14`；`subsystem→EK-28`
- 证据: `docs/rap/about/subscription-events.md#how-subscriptions-work`；`src/traits.rs:223`（get_active_subscriptions）；`src/tools/cancel_subscription.rs`

### G. 治理与工程纪律

**EK-32 · 错误处理铁律（AGENTS.md）**
"Never ignore errors"；`Result::Err` 必须传递或至少记录；`.expect("描述")` 替代 `.unwrap()`，逻辑 bug 前缀 "bug:"。代码中可见 `.expect("bug: ...")`（如 spawn_thread safe spawn point）。
- links: `constraint→EK-33`（工程质量）；`subsystem→EK-34`
- 证据: `AGENTS.md`；`src/tools/thread.rs:105`（expect "bug: ..."）

**EK-33 · Conventional Commits + check.bash 门禁**
提交规范（feat/fix/refactor/docs/test/chore/ci/perf + `!` breaking）；`./check.bash` 必跑；CI 多矩阵（Playwright e2e、clippy、THIRD-PARTY 校验）。
- links: `constraint→EK-32`；`mechanism→EK-34`
- 证据: `AGENTS.md`；commit:de1ec5a（ci ubuntu-24.04 Playwright）

**EK-34 · License 治理（THIRD-PARTY 白名单）**
自动化生成 THIRD-PARTY（cargo-about + license-checker）；Rust/npm 双白名单；allowlist 外 license 使 CI 失败。
- links: `constraint→EK-33`；`subsystem→EK-32`
- 证据: `CHANGELOG`（agent-core v0.1.0）；`about.toml`；`scripts/generate-third-party.sh`

**EK-35 · 测试纪律（可执行知识）**
系统级测试 2,199 行覆盖：中断/去重/compaction/overflow/reboot/fresh-context/父子消息；TUI 快照测试（vt100+alacritty，34 个 bug 文档化，commit:cdcf093）；Playwright e2e（TUI↔daemon↔web，commit:646c8f3）。
- links: `mechanism→EK-21`（reboot 测试）；`subsystem→EK-36`（测试架构）
- 证据: `src/system/tests.rs`；`CHANGELOG`；commit:16c123c/646c8f3

### H. 性能与内存

**EK-36 · 空闲 agent 内存经济（≈154 KB/agent）**
benchmark（agent_scale.rs）：50,000 agents × 20 turns = 2,000,000 completions / 255 s / 7.9 GB RSS ≈ 154 KB/agent；空闲 agent = 只有历史在 stores。优化史：commit:9c07368（"cut idle-agent memory ~30% — seventy thousand agents in 8 GB"）。
- links: `causal→EK-09`（挂起=纯数据的实证）；`mechanism→EK-37`
- 证据: `CHANGELOG`；`src/infinity-agent-core/examples/agent_scale.rs`；commit:9c07368/8b271baa

**EK-37 · 流式文本合并（coalesce chunks）**
连续文本 chunk 合并为一条消息（减少历史条目与 API 调用）；turn buffer 内 coalescing。
- links: `mechanism→EK-05`；`subsystem→EK-36`
- 证据: `commit:8345b7d`（"coalesce consecutive streamed text chunks"）；`src/event_processor.rs:358`（handle_completion）

### I. 边界与例外

**EK-38 · 无运行时超时（stall 检测归 provider）**
runtime 自身不加超时；provider 负责 stall 检测（fine-grained tool streaming 防 mid-tool-call 超时，commit:deb1a9b）。
- links: `contrast→EK-19`；`mechanism→EK-18`
- 证据: `docs/low-level/completion-loop.md`（"The runtime applies no timeouts of its own"）；commit:deb1a9b

**EK-39 · 大消息边界（256 MiB frame）**
LengthDelimitedCodec max frame 256 MiB（commit:8ad86d8）；multimodal 图像结果 end-to-end（commit:1935c38）；图像占位符（模型不支持时 `[image omitted...]`）。
- links: `contrast→EK-19`（overflow 是 context 而非 frame 边界）；`subsystem→EK-37`
- 证据: commit:8ad86d8/1935c38；`src/event_processor.rs` IMAGE_OMITTED_PLACEHOLDER

**EK-40 · OAuth / UserChoice（人类参与点）**
OAuth 挑战与用户选择以 `ChoiceId` 标识，forward 给用户，不触发 completion；pending choices 线程本地（tests.rs:19）。
- links: `mechanism→EK-08`（消息同质）；`subsystem→EK-30`（ChoiceId）
- 证据: `src/event_processor.rs:43-63`（OAuth/UserChoice 消息）；`docs/rap/spec/server/oauth.md`

**EK-41 · 线程停止语义（is_thread_stopped）**
已停止线程不响应 synthetic 输入（订阅事件/子报告/工具结果/OAuth/choices）；用户文本可恢复（"sending new text is how a user resumes a stopped thread"）。
- links: `constraint→EK-21`；`mechanism→EK-08`
- 证据: `src/traits.rs:223`（is_thread_stopped 注释）

**EK-42 · Observer 双钩（on_event 同步 / on_subscribe 追播）**
`ThreadObserver::on_event` 在事件发射点同步调用（须快）；`on_subscribe` 让中途连接的客户端 catch up（ReplaySnapshot）。daemon replay 修复史：commit:7a6e971（"replay in-progress thinking and response state"）。
- links: `mechanism→EK-05`（ReplaySnapshot=current_turn_view）；`subsystem→EK-24`
- 证据: `docs/agent-systems/observers.md`；`src/system/observer.rs`

**EK-43 · 模型动态选择（ModelSource / mid-session switching）**
`ModelSource` 每轮补全前选模型（可 per-conversation / 运行中切换）；daemon 支持 mid-session 切换（commit:1c4f71a）。
- links: `contrast→EK-22`（静态 vs 动态配置）；`mechanism→EK-44`
- 证据: `docs/agent-systems/dynamic-configuration.md`；commit:1c4f71a

**EK-44 · ThreadConfigSource（按线程解析工具/提示词）**
按 thread_id 解析 tools/extra_system_prompt/rap_notifier；进程重启后可重建（store/tenant registry 解析）；Lambda 用 DynamoDB manifest cache。
- links: `mechanism→EK-22`；`causal→EK-23`
- 证据: `docs/agent-systems/dynamic-configuration.md`；`src/infinity-agent-lambda/src/tools/toolset_cache.rs`

**EK-45 · Sandbox 可扩展模式（ModeProvider）**
sandbox 用 `ModeProvider` trait 抽象执行模式：内置 jj/git；`with_provider` 注册自定义。隔离 = 临时目录 + 模式特定清理。
- links: `contrast→EK-24`（daemon 内嵌）；`subsystem→EK-38`（大消息）；`mechanism→EK-22`（可扩展 trait 模式）
- 证据: `src/sandbox-local/src/backend.rs:42-137`；`src/sandbox-core/src/sandbox.rs:57`；commit:c431616

### J. 盲重建补充（Independent Validation 修正，2026-11-03）

**EK-46 · close_thread 边界（禁止关闭 root）**
`close_thread` 校验 thread_id 必须等于当前线程；根线程（无 parent）禁止关闭（"cannot close the root thread. Only child threads can be closed."）；可选 `rap_notifier` best-effort 通知工具服务器（失败不阻止关闭）。
- links: `constraint→EK-10`（子线程生命周期）；`mechanism→EK-13`（report 走 synthetic）
- 证据: `src/infinity-agent-core/src/tools/thread.rs:345-420`；盲重建 B-4

**EK-47 · 订阅事件线程报告标注**
`report_to_parent` 检测 `is_subscription_event_thread` → 报告加前缀 "Report from temporary child thread created to process a subscription event"（区分订阅临时子与真实子线程）。
- links: `mechanism→EK-15`（临时 child）；`causal→EK-13`
- 证据: `src/infinity-agent-core/src/tools/thread.rs:249-300`（is_subscription 分支）；盲重建 B-5

**EK-48 · Tool trait 完整契约（6 方法 + Rhai display）**
`Tool` trait：`name`/`description`/`parameters`/`execute`（异步 RAP 派发）+ `supports_sync`/`is_passive`/`display_script`（可选 Rhai 脚本美化）/ `execute_synchronous`（可选）；`ToolContext.safe_spawn_point: Option<usize>` 仅同步执行期间 set（spawn_thread 用它把自身调用从子继承历史中排除，子 store 里另存副本以扛父侧 compaction）。
- links: `constraint→EK-03`（sync loop-back 契约）；`mechanism→EK-17b`（safe spawn point 与 compaction 相关）
- 证据: `src/infinity-agent-core/src/tools/mod.rs:60-120`；盲重建 B-8

**EK-49 · provider 进程外传输（Unix socket JSON-lines）**
进程外 provider 用 **Unix domain socket + JSON-lines**（`serve_provider` 暴露 / `UnixListener` / `LinesCodec`；socket 路径短（macOS ~104 字节限制））；与 RAP 工具 HTTP 传输形成对比。
- links: `contrast→EK-04`（RAP=HTTP POST vs provider=Unix socket）；`mechanism→EK-22`（平台 trait 的另一实例）
- 证据: `src/infinity-provider-protocol/src/remote.rs:1-50`；盲重建 B-9

**EK-50 · 幂等 ID 覆盖类型精确化**
`sync()` 只把 `InfinityMessage::User` 与 `SubscriptionEvent` 的 message ID 写入 processed-ID 集（代码 filter 显式 `matches!`）；assistant/tool-call 项只存在于已去重输入的下游，工具结果靠 history tail 去重。
- links: `constraint→EK-06`（精确化）；`mechanism→EK-02`
- 证据: `src/infinity-agent-core/src/event_processor.rs:627-673`（sync filter）；盲重建 B-7

## EK Graph 质量指标

| 指标 | 值 | 标准 |
|---|---|---|
| EK 总数 | 50（45 原始 + 5 盲重建补充 EK-46~50） | 40~60 |
| 平均出边数 | 50 EK 全部有 links（≥2 条/条） | ≥1 |
| 游离 EK | 0（全部连入 ≥1 簇） | <20% |
| KO 聚合规则覆盖率 | 9/9 = 100% | 100% |
| KO 平均簇规模 | 见 03（3~12 健康） | 3~12 |

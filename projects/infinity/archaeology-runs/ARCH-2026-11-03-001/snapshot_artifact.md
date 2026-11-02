# Repository Snapshot Artifact — AWS Infinity（hydro-project/infinity）

| 字段 | 值 |
|---|---|
| **run_id** | ARCH-2026-11-03-001 |
| **project** | AWS Infinity（Hydro 项目，AWS × UC Berkeley） |
| **repository** | https://github.com/hydro-project/infinity.git |
| **branch** | main |
| **commit SHA** | `1abc6fcff1f186124f924b56aa92c3a3684c90c5` |
| **commit 说明** | fix: enable aws-config sso feature so SSO profiles work (#176) |
| **snapshot 时间** | 2026-11-03（本地 UTC+8） |
| **克隆方式** | 浅克隆（--depth 1 --branch main） |
| **license** | Apache-2.0（仓库 LICENSE.txt；SPDX `Apache-2.0`，GitHub API 确认） |
| **语言** | Rust（workspace，21 crates） |
| **创建时间** | 2026-04-13（GitHub API） |
| **最近推送** | 2026-10-29T22:57:26Z（GitHub API） |
| **规模信号** | 468★ / 48 forks / 54 open issues / size 5003 KB（GitHub API） |

## 一、项目定位（README / docs 一手）

Infinity 是 **Rust 编写的"大规模并发 agentic 系统"运行时**：`Infinity does for agents what async did for threads`——agent 不再阻塞等待慢工具，而是并发运行工具、等待时让出（yield）、在下一事件到达前零成本。定位陈述："light enough to fit 75k agents in the memory of a Raspberry Pi"（README）。

- 官网：https://infinity.hydro.run
- 三大组成（README）：
  1. **Infinity Runtime**：核心运行时（快速上手 / yielding / hibernation / AWS Lambda 部署）
  2. **Reactive Agent Protocol (RAP)**：开放协议，网络化异步工具服务（结果/订阅事件按需投递，即使 agent 当前无进程）；MCP 兼容层
  3. **Infinity Code**：基于该运行时的编码 agent（后台跑构建/测试、并行编辑线程 + 叠加沙箱、diff 交付）

## 二、仓库结构（顶层）

```
AGENTS.md  Cargo.lock  Cargo.toml  LICENSE.txt  NOTICE  README.md  RELEASING.md  THIRD-PARTY
agent/  check.bash  clippy.toml  crates/  docs/  infinity-ui/  infinity-web/  rap-servers.example.json  scripts/
```

- **crates/**：21 个 Rust crate（见下）
- **docs/**：Docusaurus 站点（6507 行 .md/.mdx 一手文档，含 infinity-runtime / infinity-code / rap 协议规范）
- **agent/**、**infinity-ui/**、**infinity-web/**：编码 agent 与 Web 资产
- **check.bash**：CI 检查脚本
- **AGENTS.md**：提交规范（Conventional Commits）+ 开发要求（`./check.bash` 测试、禁止忽略 Err、用 `.expect("...")` 且 bug 前缀 "bug:"）

## 三、Workspace 与 crate 布局（21 crates，Cargo.toml `[workspace] members`）

| crate | Rust 行数 | 职责（依据 src 结构与文档） |
|---|---|---|
| **infinity-agent-core** | 15,372 | 核心运行时：slice/step 管线、事件处理器、thread 模型、stores traits、tools、builder |
| **infinity-agent-cli** | 10,294 | CLI 驱动（本地运行/daemon 客户端，含 examples/tests） |
| **infinity-daemon** | 9,215 | 嵌入式守护进程：session 管理、WebSocket、RAP servers、MCP proxy、memory store |
| **sandbox-local** | 4,488 | 本地沙箱（编码 agent 隔离，build.rs/scripts） |
| **sandbox-core** | 3,543 | 沙箱抽象：callback/git/jj/metadata/server/types |
| **infinity-slack-bot** | 2,371 | Slack 集成（build.rs） |
| **infinity-agent-lambda** | 1,894 | **serverless step-mode 参考实现**（event_handler/state_store/conversation_history/tools） |
| **infinity-slack-dataflow** | 1,817 | Slack dataflow（build.rs） |
| **infinity-provider-bedrock** | 1,445 | AWS Bedrock 模型 provider |
| **infinity-provider-protocol** | 1,280 | provider 抽象协议 |
| **infinity-mcp-bridge** | 797 | MCP 兼容层（RAP 与 MCP 桥接） |
| **rap-github-event-poller** | 715 | GitHub 事件轮询（RAP 示例工具） |
| **infinity-rap-bridge** | 704 | RAP 桥接 |
| **infinity-protocol** | 613 | 协议类型（ThreadId/ToolCallId/ProviderCallId 等） |
| **sandbox-remote** | 534 | 远程沙箱 |
| **rap-client** | 515 | RAP 客户端（http/notifier） |
| **infinity-provider-rig** | 504 | Rig provider |
| **rap-protocol** | 392 | RAP 线上协议类型（RapInvocation 等） |
| **rap-steering-server** | 348 | steering server |
| **rap-test-servers** | 246 | 测试工具服务器 |
| **infinity-mdtests** | 100 | mock provider / 测试辅助 |

**合计**：168 个 .rs 文件 / 57,187 行 Rust（python3 逐文件统计）。

## 四、核心模块（infinity-agent-core/src）

```
lib.rs              顶层：pub mod event_processor / message / stores / system / tools / traits
event_processor.rs   slice 三阶段：prepare_input / run_completion / execute_action（文档 completion-loop.md）
message.rs          InputMessage（一切皆消息，含 synthetic 标签 + group_id）
stores.rs           ConversationStore / StateStore traits
system/
  builder.rs        AgentSystemBuilder（new_local / start / step，712 行）
  thread.rs         Thread<C,S,M,H>：replay_snapshot / pending_tool_call / step_no_defer（724 行）
  tests.rs          系统级测试（2199 行）
  test_support.rs   测试辅助（453 行）
  events.rs / observer.rs / defer.rs / model.rs / config.rs
tools/
  thread.rs         spawn_thread / report_to_parent / close_thread（671 行）
  rap_tool.rs       RAP 工具实现（393 行）
  sleep.rs          sleep / sleep_until（228 行）
  cancel_subscription.rs / config.rs
```

**核心不变式（docs/infinity-runtime/architecture.md）**：`an execution slice never blocks on anything external`。

**Slice 解剖**（一手文档）：
1. **Load**：`HistoryManager::new_with_history` 从 ConversationStore 恢复会话（含祖先链/compaction 摘要）
2. **Prepare + complete**：`prepare_input` 去重/丢 closed/路由订阅事件；`run_completion` 流式补全
3. **Dispatch + yield**：`execute_action` 派发工具调用（RAP = 单次 HTTP POST + callback_url，fire-and-forget）；slice 结束

**Synchronous tools loop-back**：`Tool::execute_synchronous` 内联执行并 loop 回补全（不 yield）——避免 spawn_thread 类微秒级调用的浪费与竞态；未知工具名同样 loop-back 注入错误结果（幻觉工具调用只多花一次补全而非卡死会话）。

**Turn 持久化**：流式输出缓冲于 turn buffer，turn 完成才提交；stream 中途错误则丢弃缓冲重试；工具调用结束 turn 时**先 flush 再 dispatch**（保证调用在结果可能到达前已 durable）。

**幂等**：每条输入消息/补全带稳定 ID，StateStore 追踪已处理 ID；redelivery 在 prepare_input 识别跳过 → 在 at-least-once 投递上实现 slice 幂等。

**FIFO 排序**：按 group_id 分组，传输层保证组内 FIFO（AWS = SQS FIFO MessageGroupId；嵌入式 = mpsc channel + worker task per thread）；无分布式锁。

## 五、关键机制（一手文档证据）

### 1. Step Mode（serverless）
`AgentSystem::step`：无驻留任务，每次消息投递 = 一次调用。`inputs: Vec<(InputMessage, String)>`（消息 + 稳定 dedup ID）。批内可跨多线程（SQS FIFO batch >1 可交织多个 group）。`step` 按线程分区、并发跑 per-thread slice。`StepOutcome`: `Skipped`（输入被吸收：重复/路由/closed）vs `Completed`（token 用量 + context window → 调度器触发 compaction）。
- 串行约束：per-thread 跨调用串行（`&mut self` + SQS FIFO group 契约）；跨进程无锁。
- **Deferral**：`DeferQueue`（NoDeferral / InMemoryDeferQueue / 自定义 durable）决定等待非被动工具调用期间订阅事件/子报告停在哪。

### 2. Lambda 嵌入（infinity-agent-lambda/src/event_handler.rs = 生产 step-mode 参考实现）
单 SQS FIFO 输入队列（keyed by ThreadId）→ 每次 invocation 构建 stores（DSQL conversations + DynamoDB state）、Bedrock model、tools、sender、system → `step` → 事件经 EventCollector 转 output queue。循环：工具完成后 POST 到 RAP Receiver Lambda → 重新入队输入队列。

### 3. RAP（Reactive Agent Protocol）
开放协议：runtime↔tool 异步 HTTP。`POST /invoke` → 200 确认 → runtime 持久化退出 → tool 完成 POST callback_url（tool_result / oauth / subscription_event / user_choice）。RAP 借鉴 MCP，扩展异步执行/长时操作/事件订阅（MCP 兼容层 infinity-mcp-bridge）。

### 4. Threading（spawn_thread / report_to_parent / close_thread）
- `child_of`（调用者完整线程栈）防子线程执行父线程的 spawn 计划
- 子线程继承父历史至 spawn 点 + 指令；`fresh_context: true` → 空历史（自包含指令）
- 报告经 input queue 以事件到达父线程 → synthetic tool result
- 订阅事件默认**临时子线程**处理（不污染父 context）；associative 事件内联注入主线程历史

### 5. Compaction
本地 driver 在历史接近 3/4 context window 时触发 compaction 线程 → 摘要存 ConversationStore（按历史 index 标记）→ 后续 slice 加载摘要 + index 后消息。子线程按位置重建继承历史。

### 6. Durable Timers（serverless）
sleep/sleep_until：≤900s 走 SQS DelaySeconds；更长走一次性 EventBridge schedule。可中断（用户消息/订阅事件到达立即处理，挂起 sleep 结果稍后追加）。

### 7. Synthetic tool calls（博客 + docs）
事件到达 → runtime 添加 assistant tool call `receive_event__injected`（模型不可主动调用，runtime 拒绝）+ tool result 含事件。保留模型 post-training 的工具结果解读能力；`__injected` 后缀标识 runtime 创建。

## 六、测试体系

- **单元/集成测试**：2 个 `*_test.rs` + 29 个 `tests/` 目录 .rs（crates 内）
- **快照测试**：150 个 .snap 文件（insta 风格，agent-core/src/snapshots/ 存在）
- **系统级测试**：`crates/infinity-agent-core/src/system/tests.rs`（2199 行）
- **check.bash**：CI 检查（clippy.toml / clippy 配置存在）
- **infinity-mdtests**：mock provider（测试辅助）

## 七、配置 / 权限与治理 / 外部依赖

- **配置**：Cargo.toml workspace deps（tokio/serde/async-trait/axum 等）；rap-servers.example.json（RAP 服务器清单）；daemon config.rs；agent-cli examples
- **权限与治理**：AGENTS.md（提交规范/开发要求——Conventional Commits、禁忽略 Err、.expect 带描述）；沙箱（sandbox-core/local/remote）提供隔离；Lambda IAM policy（serverless 架构文档提及）
- **外部依赖**：AWS（Bedrock provider、Lambda、SQS FIFO、DSQL、DynamoDB、EventBridge）、tokio 生态、MCP（兼容层）、GitHub（event poller）、Slack（bot/dataflow）

## 八、快照质量标注

- **证据等级**：全部事实来自仓库实际内容（README/docs/crates/Cargo.toml/AGENTS.md）+ GitHub API（规模/时间戳），无第三方转述
- **本环境限制**：无 Docker/容器运行时 → 无法实跑 demo/benchmark；静态考古证据上限 **S4**（测试验证级），不升 S5
- **测试运行**：未在本环境执行 `cargo test`（浅克隆 + 无编译环境依赖保证）；测试体系以文件证据为准

# 04 · Flow Atlas — AWS Infinity

> 七类流全部从真实代码导出，关键 Edge 可回溯 symbol/file/condition/state transition。

## 1. Control Flow（控制流）

### 1.1 slice 主循环（Lambda step-mode 实例）

```
SQS record ──▶ function_handler (lambda/src/event_handler.rs:37)
  │ parse InputMessage batch ──▶ system.step(inputs, &collector, &mut NoDeferral)
  │
  ▼
AgentSystem::step (system/mod.rs)
  │ 按 group_id 分区 ──▶ per-thread: Thread::step (system/thread.rs)
  │
  ▼
prepare_input (event_processor.rs:889)
  │  [投递顺序：去重 → closed/stopped 丢弃 → 订阅路由 → 追加]
  │  返回 PrepareResult { Ready | Handled | OAuthRequired | UserChoiceRequired | CompactionApplied }
  ▼
run_completion (event_processor.rs:1480)
  │  流式 chunk ──▶ handle_completion (缓冲/合并)
  │  sync tool? ──▶ execute_synchronous loop-back（不结束 slice）
  │  ContextOverflow ──▶ 恢复路径（EK-19）
  │  返回 CompletionAction { Done | ExecuteToolCall }
  ▼
execute_action (event_processor.rs:1933)
  │  RAP: POST /invoke (fire-and-forget)
  │  时序: turn buffer flush 先于 dispatch（EK-05）
  ▼
slice 结束（Lambda：无 task 存活；local：driver 回到 channel）
```

### 1.2 同步工具 loop-back（spawn_thread 路径）

```
run_completion 遇 spawn_thread 工具调用
  │
  ▼
execute_synchronous (tools/thread.rs:65)
  │  child_of 校验（不匹配 → 错误结果返回）
  │  conversation_store.spawn_thread（新 group）
  │  seed 消息 → input queue（EK-25 loopback）
  │
  ▼
结果注入历史 + 同 slice 内再跑一次 run_completion（不 yield）
```

### 1.3 订阅事件路由（非 associative）

```
subscription_event 到达 input queue（group_id=订阅线程）
  │
  ▼
prepare_input 检测非 associative synthetic 事件
  │
  ├─▶ 默认：spawn 临时 child（继承父上下文 + 事件）→ child 处理 → report 回父
  └─▶ associative: inline 追加到订阅线程历史
```

## 2. State Flow（状态流）

### 2.1 HistoryManager 三层缓冲状态

```
                     ┌─────────────────────────────────────────────┐
                     │ HistoryManager（每线程一个实例）              │
  已提交历史 (committed)│                                           │
  ┌──────────────┐    │   ┌──────────┐  ┌──────────┐  ┌─────────┐  │
  │ Conversation │    │   │ 已知安全  │  │ 未验证   │  │ turn    │  │
  │ Store（durable│───▶│   │ pending_ │  │ inputs   │  │ buffer  │  │
  │ 历史+祖先+压缩 │    │   │ items    │  │ (unvalid)│  │ (stream)│  │
  └──────────────┘    │   └──────────┘  └──────────┘  └─────────┘  │
                      └─────────────────────────────────────────────┘
  状态转移：
  - handle_content: 用户输入 → unvalidated inputs（sync 不落库）
  - mark_inputs_model_validated: 模型输出证明 context 适配 → pending_items
  - handle_completion: 流式 chunk → turn buffer（不 commit）
  - flush_turn / flush_turn_trimming_reasoning: buffer → committed（可落库）
  - discard_turn: 中途失败 → 丢弃 buffer（重试从干净历史重建）
  - sync: pending_items + processed IDs → ConversationStore/StateStore（幂等点）
```

### 2.2 线程状态机

```
LAUNCH ──▶ ACTIVE(slice 执行中) ──▶ YIELDED(等消息；无 task)
   ▲            │                        │
   │            ▼                        ▼
   │      TOOL_WAIT(pending call)   IDLE(本地 driver 回收)
   │            │                        │
   │            ▼                        │
   └─────── RESUME(结果到达,新 slice)────┘
   │
   ▼
COMPACTING（>75% context，spawn compaction child）
   │
   ▼
STOPPED（is_thread_stopped=true；仅用户文本可恢复）
```

## 3. Data Flow（数据流）

### 3.1 输入消息流

```
InputMessage {
  group_id: ThreadId     → 目标线程（FIFO 组）
  content: User{Text|ToolResult} | Synthetic{...}
  synthetic: Option<SyntheticKind>   → 事件类型标记
  display_as / metadata / subscription
}
```

### 3.2 数据持久化分界

| 数据 | 何时 durable | 存哪 | 幂等机制 |
|---|---|---|---|
| 用户文本 | 模型输出后（sync） | ConversationStore + processed ID | message_id |
| 订阅事件 | 模型输出后 | 同上 | message_id |
| 工具结果 | sync（tail 去重） | ConversationStore | history tail（无需 ID） |
| tool call | turn flush 时 | ConversationStore | — |
| 会话 metadata | set_metadata | StateStore | — |
| 活跃订阅 | subscription:true result | StateStore | tool_call_id |

## 4. Evidence Flow（证据流）

### 4.1 调试/观测链

```
slice 事件 ──▶ AgentEvent ──▶ ThreadObserver::on_event（同步，发射点）
                               ├──▶ client broadcast（UI/WS）
                               ├──▶ token 计数器（usage 统计）
                               └──▶ ReplaySnapshot（current_turn_view，中途连接 catch up）
```

### 4.2 测试证据链

```
tests.rs 28 测试（system 级）──▶ 行为契约（可执行知识）
  ├── compaction×tool call 竞态 4 个
  ├── fresh context 隔离 2 个
  ├── fatal error 后父消息到达 2 个
  ├── shutdown 持久化语义 2 个
  ├── overflow/reboot/stale 各 1 个
  └── TUI 快照 150 个（vt100+alacritty）
```

## 5. Authority Flow（权威流）

### 5.1 权限边界

```
用户文本 ──▶ 唯一无条件入口（可恢复 stopped 线程）
其他输入 ──▶ 需 is_thread_stopped=false（订阅事件/子报告/工具结果/OAuth/choices）
   │
   ▼
模型 ──▶ 只能通过工具调用触碰外部世界
   │      ├── RAP 工具（HTTP POST，工具自身授权）
   │      ├── 内置工具（spawn/close/sleep/cancel；sandbox 提供隔离）
   │      └── receive_event__injected ──▶ REJECTED（模型不可主动调用）
   │
   ▼
child_of 校验（spawn_thread：防止子线程误执行父计划）
   │
   ▼
sandbox（jj/git ModeProvider）──▶ 编码 agent 的文件系统隔离
```

### 5.2 治理闭环（Policy）

```
AGENTS.md（Conventional Commits + check.bash + 错误铁律）
  │
  ▼
CI 矩阵（ubuntu-24.04 Playwright + clippy + THIRD-PARTY license）
  │
  ▼
RELEASING.md / cargo-smart-release（v0.1.0 2026-09-03 全线 / v0.2.0 safety bump 14 crates）
  │
  ▼
THIRD-PARTY 白名单（非 allowlist license → CI 失败）
```

## 6. Memory Flow（记忆流）

### 6.1 历史继承链

```
load_history_with_ancestors (traits.rs:27 默认实现)
  │
  ├─▶ 1. 检查自身 compaction summary → 摘要 + index 后消息
  ├─▶ 2. 祖先链 get_ancestor_chain
  │       ├─▶ fresh-context 边界截断（fresh 线程：空；祖先 fresh：清上面）
  │       └─▶ 反向找最近 compaction summary（按 cutoff 定位）
  └─▶ 3. 拼接：祖先消息（至 spawn 点） + 本线程消息
```

### 6.2 记忆的遗忘策略

| 策略 | 触发 | 机制 |
|---|---|---|
| Compaction | >75% context | 摘要替换历史（按 index 标记） |
| fresh_context | spawn 时 | 子孙不见父历史 |
| discard_turn | 中途失败 | 未完成 turn 不落库 |
| session 清理 | 7 天 | daemon sessions.json |
| oversized drop | ContextOverflow | 丢新用户输入/占位符替换 |

## 7. Policy Flow（策略流）— 治理闭环

```
Decision（AGENTS.md 规则 / check.bash 门禁）
  │
  ▼
Approval（CI 矩阵：lint + test + license + e2e）
  │
  ▼
Policy（THIRD-PARTY allowlist / 错误分类 ErrorClass / 幂等契约）
  │
  ▼
Enforcement（check.bash 失败→构建失败；overflow→恢复路径；dedup→StateStore）
  │
  ▼
Future Decision（CHANGELOG 记录 + RELEASING.md + Conventional Commits 反馈）
```

## Flow→KO 交叉校验

| KO | 主要 Flow Edge | 校验 |
|---|---|---|
| KO-01 | Control 1.1（slice 三阶段） | ✓ 从代码导出 |
| KO-02 | State 2.1 + Control 1.1（幂等/FIFO） | ✓ |
| KO-03 | Control 1.3 + Data 3.1（synthetic 路由） | ✓ |
| KO-04 | Memory 6.2（compaction/overflow） | ✓ |
| KO-05 | State 2.1 + Data 3.2（4-trait 分界） | ✓ |
| KO-06 | Control 1.2 + Authority 5.1（sleep passive） | ✓ |
| KO-07 | Data 3.2 + Memory 6.2（错误分类→恢复） | ✓ |
| KO-08 | Evidence 4.2（测试契约） | ✓ |
| KO-09 | Control 1.2 + State 2.2（工具=并发边界） | ✓ |

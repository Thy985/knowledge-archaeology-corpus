# Flow Atlas — Infinity（七类流 34 Edge）

> 全部从真实代码导出，关键 Edge 可回溯 symbol/file/condition。

## Control Flow（5）
| # | Edge | 来源 |
|---|---|---|
| CF-1 | 输入到达 → router 分派（spawn/respawn driver） | router.rs:177-178 |
| CF-2 | driver 批处理 channel 输入 → Thread::step | driver.rs:100-118 |
| CF-3 | step：prepare → completion → commit barrier → dispatch barrier | thread.rs:500-606 |
| CF-4 | 同步工具 loop-back（should_loop_back → 'outer） | event_processor.rs:1873-1900 |
| CF-5 | driver 三态 idle 决策（Continue/AwaitInput/Exit）→ 退出或等待 | driver.rs:331-363 |

## State Flow（5）
| # | Edge | 来源 |
|---|---|---|
| SF-1 | unvalidated_items →（mark_inputs_model_validated）→ pending_items | event_processor.rs:510-515 |
| SF-2 | turn_buffer →（flush_turn）→ history（known_safe） | event_processor.rs:358-410 |
| SF-3 | in_flight true→false（completion 生命周期） | thread.rs:566,585 |
| SF-4 | driver Live ⇄ Idle（同点报告 + active_threads 原子变更） | driver.rs:90-130 |
| SF-5 | pending tool call →（用户文本）→ interrupted 合成 | event_processor.rs:322-355 |

## Data Flow（5）
| # | Edge | 来源 |
|---|---|---|
| DF-1 | InputMessage → HistoryManager（history/unvalidated/turn_buffer） | event_processor.rs:244-289 |
| DF-2 | 工具结果 → match_tool_result 反向走查 → append | event_processor.rs:300-330 |
| DF-3 | 模型输出 chunk → handle_completion → turn_buffer → flush | event_processor.rs:358-440 |
| DF-4 | processed_message_ids 内存 ↔ DynamoDB String Set | state_store.rs:48-80 |
| DF-5 | 会话历史 ↔ Postgres conversation_history（message_id+data） | conversation_history.rs:86,249-275 |

## Evidence Flow（4）
| # | Edge | 来源 |
|---|---|---|
| EF-1 | 模型输出 = 未验证输入的验证证明（mark_inputs_model_validated） | event_processor.rs:161-165, 510 |
| EF-2 | interrupted 合成结果（`{id}-interrupted`）入 history | event_processor.rs:322-355 |
| EF-3 | drop_unvalidated_user_inputs（溢出时遗忘 dedup ID 防 wedge） | event_processor.rs:525-560 |
| EF-4 | compaction summary 持久化（save/load） | stores.rs:353-406 |

## Authority Flow（5）
| # | Edge | 来源 |
|---|---|---|
| AF-1 | 工具注册表（config.tools）→ 名/描述/参数 → 模型 tool_defs | thread.rs:498-520 |
| AF-2 | sync vs async 分派权（SyncToolCall 直接执行 / ExecuteToolCall action） | event_processor.rs:1855-1900 |
| AF-3 | spawn_thread 仅同步执行（safe_spawn_point 守卫，bug 断言） | tools/thread.rs:112-121 |
| AF-4 | RAP callback_url 注入 ToolContext（工具回调目标） | thread.rs:500-512 |
| AF-5 | 工具集服务器（toolset_server_urls）加载 + 缓存 | lambda event_handler.rs:100-110, toolset_cache.rs |

## Memory Flow（5）
| # | Edge | 来源 |
|---|---|---|
| MF-1 | ConversationStore：load_history_up_to/append_messages/spawn/close | stores.rs:201-406 |
| MF-2 | StateStore：processed_ids/metadata/subscriptions/user_choices | stores.rs:486-595 |
| MF-3 | InMemory 实现（resident 内存态）vs DynamoDB/Postgres（serverless） | stores.rs:68-423 / lambda |
| MF-4 | turn_buffer 与 history 的分离（部分 turn 不落历史） | event_processor.rs:358-440 |
| MF-5 | 订阅状态（active_subscriptions）不维持 driver 驻留 | driver.rs:358-363 |

## Policy Flow（5）
| # | Edge | 来源 |
|---|---|---|
| PF-1 | defer 策略：DeferQueue（可 defer vs NoDeferral） | defer.rs:1-76 |
| PF-2 | 中断策略：用户文本=打断，synthetic 事件=defer 或打断 | thread.rs:52-70 |
| PF-3 | 幂等策略：processed_ids + 反向走查 | event_processor.rs:244-330 |
| PF-4 | 失败恢复策略：execute_action_with_error_result（error result 入队） | event_processor.rs:1960-1990 |
| PF-5 | compaction 策略：compaction_complete 触发 + context_window 阈值 | driver.rs:81-84, stores.rs:353 |

## 统计
34 Edge（Control 5 / State 5 / Data 5 / Evidence 4 / Authority 5 / Memory 5 / Policy 5）；Flow→KO 交叉校验 8/8（每条 L1 事实可回溯 Flow Edge）

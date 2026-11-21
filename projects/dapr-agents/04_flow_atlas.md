# Flow Atlas — Dapr Agents（七类流 28 Edge）

> 全部从真实代码导出，关键 Edge 可回溯 symbol/file/condition。

## Control Flow（4）
| # | Edge | 来源 |
|---|---|---|
| CF-1 | TriggerAction 到达 → agent_workflow 启动（@message_router 路由） | durable.py:580-586 |
| CF-2 | agent_workflow 三分支分派（orchestrator / executor / 标准循环） | durable.py:631-760 |
| CF-3 | executor 审批循环：run_executor → paused → _await_approval → resume | durable_executor.py:141-218 |
| CF-4 | orchestration 5 activity 链（initialize→select→execute→handle→finalize） | durable.py:1879-1947 |

## State Flow（5）
| # | Edge | 来源 |
|---|---|---|
| SF-1 | workflow 状态：每 step activity 输出持久化（event sourcing） | durable_executor.py:36-58（docstring） |
| SF-2 | executor 会话：session_id → DaprSessionStore 恢复 | durable_executor.py:96-118, 394 |
| SF-3 | 审批暂停点：PausedExecutorRun（tool_call_id/arguments） | executor_run.py（PausedExecutorRun） |
| SF-4 | 审批决策：ToolCallDecision → CONTEXT_TOOL_DECISIONS → 恢复输入 | durable_executor.py:59-96 |
| SF-5 | PlanStep 状态机：update_step_statuses/restructure_plan | orchestrators/llm/state.py |

## Data Flow（4）
| # | Edge | 来源 |
|---|---|---|
| DF-1 | TriggerAction（task/metadata）→ agent_workflow 输入 | durable.py:590-600 |
| DF-2 | call_llm 输出（assistant_response + tool_calls）→ 工具执行 | durable.py:700-730 |
| DF-3 | executor 回合输入（task/instance_id/context/session_id）→ activity | durable_executor.py:141-148 |
| DF-4 | AgentWorkflowMessage（消息内容）→ 子 agent/agent 间 | schemas.py:201-211 |

## Evidence Flow（3）
| # | Edge | 来源 |
|---|---|---|
| EF-1 | workflow 重放判定：ctx.is_replaying → WorkflowReplayFilter 抑制日志 | utils/logger.py:24-44, 53-79 |
| EF-2 | 审批证据：ApprovalRequiredEvent（tool_call 描述）→ ApprovalResponseEvent（approved/reason） | schemas.py:30-150 |
| EF-3 | 运行记录：record_initial_entry（activity 持久化首条目） | durable.py:611-640, 2191 |

## Authority Flow（5）
| # | Edge | 来源 |
|---|---|---|
| AF-1 | 工具注册：AgentTool/tool 装饰器 → tool_executor → 模型 tool_defs | tool/__init__.py + durable.py:715-730 |
| AF-2 | hook 决策权：before_tool_call（RequireApproval/Deny/Proceed，按位置键控） | durable.py:730-800 |
| AF-3 | 审批权威：human（ApprovalResponseEvent）vs 上限自动拒绝 | durable_executor.py:177-218 |
| AF-4 | 子 agent 调度权：AgentWorkflowTool ctx.call_child_workflow（确定性 id） | tool/workflow/agent_tool.py:198-228 |
| AF-5 | 跨 app 限制：activity 内跨 app 子 workflow 调度 → ToolError | tool/workflow/activity_bridge.py:298-308 |

## Memory Flow（4）
| # | Edge | 来源 |
|---|---|---|
| MF-1 | agent-memory（会话历史，AgentMemoryConfig）→ LLM 上下文 | configs.py（AgentMemoryConfig） |
| MF-2 | agent-workflow（durable state，ActorStateStore）→ 恢复 | configs.py:246（AgentStateConfig） |
| MF-3 | DaprSessionStore → executor 会话（binding） | executors/dapr_session_store.py |
| MF-4 | schema bundle 注入：entry_factory/message_coercer 决定状态模型 | configs.py:246-312 |

## Policy Flow（3）
| # | Edge | 来源 |
|---|---|---|
| PF-1 | 审批策略：require_approval 工具 → HITL；上限自动拒绝 | durable_executor.py:141-218 |
| PF-2 | 重试策略：retry_policy（activity 级）；METADATA_RETRYABLE=False → fail | durable_executor.py:36-58 |
| PF-3 | 订阅去重策略：TTLDedupeBackend（subscribe 模式） | workflow/runners/agent.py（import TTLDedupeBackend） |

## 统计
28 Edge（Control 4 / State 5 / Data 4 / Evidence 3 / Authority 5 / Memory 4 / Policy 3）；Flow→KO 交叉校验 7/7

# Engineering Knowledge Graph — Dapr Agents（20 EK）

> 每条 EK 带证据锚（file:line）+ links（六类边）。聚合规则见 03。

## EK-D01 DurableAgent 是唯一 agent 路径（Agent deprecated 为文档层事实）
仓库代码无 `class Agent`（仅 `AgentBase` + `DurableAgent(DurableExecutorMixin, AgentBase)`）；顶层导出 `DurableAgent`（dapr_agents/__init__.py:17）。品类卡记录的"Agent 类 deprecated（v1.0.0-rc.1）"为 docs 层事实，代码已完全迁移。
- [code] dapr_agents/agents/durable.py:269
- [code] dapr_agents/agents/__init__.py:14
- links: 子系统 EK-D02（同属 DurableAgent 主类：存在性与入口）

## EK-D02 agent_workflow 三分支：orchestration / executor / 标准循环
`agent_workflow`（@workflow_entry + @message_router(TriggerAction)）按能力分支：有 orchestration_strategy → 子 workflow；有 executor → `_executor_workflow`；否则标准循环（call_llm activity + 工具执行 + before_tool_call hooks）。
- [code] dapr_agents/agents/durable.py:580-760
- links: 分派 → EK-D04（executor 分支）/ EK-D18（orchestration 分支）；同机制 EK-D03

## EK-D03 workflow 步骤 = call_activity 边界（非确定性 I/O 必须 activity 化）
所有非确定性操作经 `ctx.call_activity(..., retry_policy=...)` 执行（record_initial_entry / load_tools / call_llm / run_executor）；标准循环中普通工具批量 activity、WorkflowContextInjectedTool（需 ctx 的 agent/ask_user 工具）内联 workflow body（"cannot be ran within an activity bc activities do not have the workflow context"，durable.py:840-960）；时间取 `ctx.current_utc_datetime` 而非 datetime.now()；日志在 `ctx.is_replaying` 时抑制。重放安全是 workflow 代码的硬纪律。
- [code] dapr_agents/agents/durable.py:611-640, 700, 840-960
- [code] dapr_agents/agents/durable_executor.py:151
- links: 机制 EK-D06/EK-D07；依赖 EK-D10（executor 也遵循）

## EK-D04 executor 回合 = activity（run_executor 至少一次，会话恢复）
executor 分支的每轮 `run_executor` 是 activity（至少一次）；重试恢复前次 executor session（任务重发、已跑工具可能重跑）；executor 标 METADATA_RETRYABLE=False 的失败 fail workflow 而非重试。
- [code] dapr_agents/agents/durable_executor.py:36-58（模块 docstring）, 233-268
- links: 因果 EK-D04 → EK-D05（暂停）/ EK-D19（会话恢复）；依赖 EK-D03

## EK-D05 HITL 审批 = 暂停-恢复闭环（PausedExecutorRun）
executor 暂停于待审批工具调用（PausedExecutorRun）→ `_await_approval`（发布审批事件 + wait external event/timeout）→ `_resume_input` 以 ToolCallDecision（approved/reason/arguments_digest）恢复。拒绝把审批者理由传给 executor。审批请求 id 确定性：uuid5(f"{instance_id}:{tool_call_id}") 重放一致，发布仅首次（重放时 activity 结果缓存）。hook 路径（_request_approval，durable.py:1201）与 executor 路径共用 ApprovalRequiredEvent/ResponseEvent 模型。
- [code] dapr_agents/agents/durable_executor.py:141-218（_executor_workflow 循环）, 59-96（_resume_input）
- [code] dapr_agents/agents/durable.py:1221-1290（_await_approval + uuid5）
- links: 因果 EK-D04 → EK-D05 → EK-D06（上限）；约束 EK-D03（wait 是 workflow 原语）

## EK-D06 审批上限自动拒绝（max_approval_rounds）
超过 max_approval_rounds（默认 max_iterations）后，再暂停的工具调用被自动拒绝（_LIMIT_REASON）给最后机会；仍暂停 → 下轮 run 前被 executor 拒绝。超限最终返回 `_APPROVAL_LIMIT_MESSAGE`。
- [code] dapr_agents/agents/durable_executor.py:177-218
- links: 因果 EK-D05 → EK-D06；约束 EK-D04

## EK-D07 重放日志抑制（WorkflowReplayFilter + contextvars）
`WorkflowReplayFilter`（logging.Filter）在 `workflow_replaying_ctx`（contextvars）为真时丢弃日志；`with_logger_context` 从函数参数提取 DaprWorkflowContext.is_replaying 设置 contextvar——重放时 agent 代码不产生重复日志。
- [code] dapr_agents/utils/logger.py:24-44, 53-79
- links: 机制 EK-D03；依赖 EK-D03

## EK-D08 双 state store：agent-memory（会话历史）vs agent-workflow（durable state）
记忆（会话历史）与执行状态（workflow/actor state）分离，配置为 AgentMemoryConfig / AgentStateConfig；执行状态需 actorStateStore（品类卡 §5 文档核对）；DaprSessionStore 绑定 agent state store 供 executor 用。
- [code] dapr_agents/agents/configs.py:246（AgentStateConfig）
- [code] dapr_agents/agents/executors/dapr_session_store.py
- links: 主题簇 EK-D09；对照 EK-D01

## EK-D09 AgentStateConfig = schema bundle 注入（按 agent 类型选状态模型）
AgentStateConfig 不是简单存储配置：`ensure_bundle` 注入 StateModelBundle（entry_model_cls/message_model_cls/entry_factory/message_coercer），按 agent/orchestrator 类型自动选（AgentWorkflowState / LLMWorkflowState）；重复注入不同 bundle raise RuntimeError。
- [code] dapr_agents/agents/configs.py:246-312
- links: 依赖 EK-D08；主题簇 EK-D18（LLMWorkflowState）

## EK-D10 agent-as-tool：AgentWorkflowTool = child workflow 调用
`AgentWorkflowTool(WorkflowContextInjectedTool)` 在 workflow body 内 `ctx.call_child_workflow` 调度子 agent；`_child_instance_id` 可显式指定（确定性恢复）；ActivityToolContext 桥接 activity 内调用。
- [code] dapr_agents/tool/workflow/agent_tool.py:118-228
- links: 机制 EK-D11/EK-D15；依赖 EK-D03

## EK-D11 bridge_workflow_tools：activity 内调用 workflow-backed 工具
activity 中无法直接用 ctx.call_child_workflow（无 ctx），`bridge_workflow_tools` 包装为通过 workflow client 调度（_ChildIds 确定性 ID 复用前次子 workflow）。需要 workflow body 的工具（ask_user）与 cross-app agents 不 bridge。
- [code] dapr_agents/tool/workflow/activity_bridge.py:277-320
- links: 机制 EK-D10；约束 EK-D12

## EK-D12 跨 app 调度限制（ToolError）
需要把子 workflow 调度到另一 app 的工具（如 make_mcp_gateway_via_child_workflow_tool）在 activity 中调用 raise ToolError——workflow client 只能调度本 app。
- [code] dapr_agents/tool/workflow/activity_bridge.py:298-308
- links: 约束 EK-D11；边界 EK-D10

## EK-D13 AgentRunner 三模式：run / subscribe / serve
run（ad-hoc，wait=False 返回 instance_id）/ subscribe（pub/sub TriggerAction 消费，TTLDedupeBackend 去重）/ serve（FastAPI：HTTP 路由 + HITL 审批路由 + 流式 SSE/NDJSON + pub/sub 路由）。三模式共享 workflow 入口。
- [code] dapr_agents/workflow/runners/agent.py:206, 871, 911
- links: 主题簇 EK-D14；依赖 EK-D02

## EK-D14 TriggerAction 消息模型 + message_router 路由
TriggerAction（task/workflow_instance_id/caller_headers/metadata）触发 agent；@message_router 装饰器注册消息路由（register_message_routes）；广播（BroadcastMessage, broadcast=True）多 agent 群发。
- [code] dapr_agents/agents/schemas.py:171-183
- [code] dapr_agents/agents/durable.py:1962-1986（on_broadcast）
- links: 依赖 EK-D13；主题簇 EK-D17

## EK-D15 ask_user：需要 workflow body 的工具（不进 activity）
ask_user_tool 依赖 workflow 上下文（等待用户输入 = wait external event），因此不参与 activity bridge——从 activity 内不可用（排除清单）。
- [code] dapr_agents/tool/workflow/ask_user_tool.py
- [code] dapr_agents/tool/workflow/activity_bridge.py:300-302
- links: 对照 EK-D10（同为工具但需要 body）；约束 EK-D11

## EK-D16 MCP 工具接入（自动发现 + workflow 网关）
AgentRunner._ensure_mcp_connected（agent 启动时连接 MCPServer 工具）；mcp_workflow_gateway 提供 child-workflow 化的 MCP 网关工具；MCP 工具在 activity 中可调用。
- [code] dapr_agents/workflow/runners/agent.py:184-204
- [code] dapr_agents/tool/workflow/mcp_workflow_gateway.py
- links: 主题簇 EK-D10/EK-D11（工具接入三族：本地/agent/workflow 网关）

## EK-D17 流式输出：StreamEmitter + SSE/NDJSON + 三消费模式
StreamEmitter 事件流（StreamChunk：Thinking/ToolCall/ToolResult/Message/Error）；HTTP 流式 handler 输出 SSE（_format_sse）或 NDJSON（_format_ndjson）；消费端 InProcessQueueConsumer / PubSubStreamConsumer；终端错误构造最小 ERROR chunk。
- [code] dapr_agents/workflow/runners/agent.py:117-186
- [code] dapr_agents/streaming/{emitter,consumers}.py
- links: 依赖 EK-D13（serve 流式）；主题簇 EK-D14

## EK-D18 orchestration_workflow：LLM 计划执行的 5 activity 链
initialize_orchestration → select_next_task →（子 agent 执行）→ handle_response → check_completion → finalize_orchestration；PlanStep 状态机（update_step_statuses/restructure_plan）；状态经 LLMWorkflowState（AgentStateConfig bundle）。
- [code] dapr_agents/agents/durable.py:1879-1947
- [code] dapr_agents/agents/orchestrators/llm/state.py
- links: 因果 EK-D18 → EK-D10（子 agent 执行）；依赖 EK-D09

## EK-D19 executor 会话恢复（session_id + DaprSessionStore）
first_input 携带 session_id 恢复前次 executor session；supports_caller_session_alias（Claude 从非 UUID id 派生 UUID）决定 session_id 来源优先级；DaprSessionStore 持久化 executor 会话。
- [code] dapr_agents/agents/durable_executor.py:96-118, 141-148
- links: 依赖 EK-D04；机制 EK-D08

## EK-D20 before_tool_call hooks：RequireApproval/Deny/Proceed（按位置键控）
标准循环中 hook 决策按 tool_calls 位置索引键控（tool_call_id 可能为空/重复）；RequireApproval → `_request_approval`（HITL）→ 转 Deny/Proceed；Deny 带理由。升级注释警告：旧版按 id 键控的混合决策在重放时会非确定性失败。
- [code] dapr_agents/agents/durable.py:730-800
- links: 对照 EK-D05（同为审批但路径不同：hook 拦截 vs executor 暂停）；依赖 EK-D03

## 图统计
- EK 20 条；links 全部声明；平均出边 2.75；游离 0

# Project Layer — Dapr Agents（项目地图）

## 定位
分布式 Agent 运行时（Python 包）：DurableAgent（唯一 agent 路径）+ AgentRunner（run/subscribe/serve 三模式）+ Dapr Workflow/actor/state store/pub-sub 基础设施绑定。

## 架构
```
  用户/调用方
     │  TriggerAction (pub/sub) / HTTP run / streaming
     ▼
 AgentRunner (workflow/runners/agent.py)
     │  三模式：run / subscribe / serve
     ▼
 DurableAgent (agents/durable.py:269 = DurableExecutorMixin + AgentBase)
     │  agent_workflow 三分支
     ├─ orchestration_workflow（LLM 计划编排：5 activity 链）
     ├─ _executor_workflow（executor 回合 durable 化：HITL 审批循环）
     └─ 标准循环（call_llm activity + 工具执行 + hooks）
     │
     ▼
 Dapr Workflow Runtime（dapr.ext.workflow，外部基础设施）
     ├─ call_activity（至少一次）/ call_child_workflow（子 agent）/ wait_for_external_event（审批）
     ├─ 状态持久化：agent-workflow（actorStateStore）
     └─ pub/sub：TriggerAction 消息 / 审批事件
  工具层：AgentTool / tool 装饰器 / MCP（tool/mcp）/ workflow 工具（agent_tool/ask_user/activity_bridge）
```

## 核心模块
| 模块 | 关键文件 | 职责 |
|---|---|---|
| agents | durable.py / base.py / durable_executor.py / configs.py / schemas.py | DurableAgent 主类 + executor 编排 + 配置/消息模型 |
| agents/executors | base.py / binding.py / observer.py / event.py / dapr_session_store.py | AgentExecutorBase 运行时（工具/推理循环） |
| agents/orchestration | strategy.py / roundrobin_strategy.py / random_strategy.py | 多 agent 编排策略 |
| agents/orchestrators/llm | state.py / schemas.py / prompts.py | LLM 计划执行（PlanStep 状态机） |
| workflow | runners/agent.py（AgentRunner）/ runners/base.py / utils/* | workflow 绑定 + 路由/订阅注册 |
| llm | chat.py + openai/anthropic/mistral/litellm/... | 多 provider 聊天客户端 |
| tool | tool/workflow/{agent_tool,ask_user_tool,activity_bridge,mcp_workflow_gateway} | 工具体系（agent-as-tool/审批/桥） |
| service | fastapi/ | serve 模式 HTTP 层 |
| streaming | consumers.py / listeners.py / emitter.py | SSE/NDJSON 流 + in_process/pubsub 消费 |
| storage | daprstores/ graphstores/ vectorstores/ | 存储抽象 |
| memory | — | agent 记忆 |
| utils | logger.py（WorkflowReplayFilter） | 重放日志抑制 |

## 核心数据结构
- `TriggerAction`（schemas.py:171）：触发消息（task + workflow_instance_id + caller_headers + metadata）
- `AgentWorkflowMessage(MessageContent)`（schemas.py:201）：agent workflow 消息内容
- `ApprovalRequiredEvent`（:30）/ `ApprovalResponseEvent`（:115）：HITL 审批事件对
- `AgentStateConfig`（configs.py:246）：状态 schema bundle 注入（store/entry_factory/message_coercer）
- `AgentMemoryConfig`：记忆配置
- `PlanStep` / `IterablePlanStep` / `NextStep`（orchestrators/llm）：计划状态
- `PausedExecutorRun`（executor_run.py）：审批暂停点（tool_call_id/arguments/session_id）
- `ExecutorRunInfo`（executors/observer.py）：executor 运行观测

## 生命周期
1. 创建 DurableAgent（绑定 executor/registry/hooks/state/memory 配置）
2. `AgentRunner.run`（ad-hoc）：discover_entry → run_workflow_async（wait=False 返回 instance_id）
3. `AgentRunner.subscribe`：注册 TriggerAction 消息路由（pub/sub）→ 事件到达 → agent_workflow 启动
4. `AgentRunner.serve`：FastAPI 挂载（HTTP 路由 + HITL 路由 + 流式）
5. agent_workflow 内：orchestration / executor / 标准循环三分支；每步 activity 持久化
6. 中断 → 重启 → 按 instance_id 从最后持久化步骤继续（Dapr workflow 语义）

## 状态机（orchestrator 分支）
initialize_orchestration → select_next_task →（子 agent 执行）→ handle_response → check_completion → finalize_orchestration（durable.py:1879-1947，5 activity 链）

## 配置
- pyproject：40+ deps；optional：drasi/claude/vectorstore；requires-python >=3.11,<3.15
- mypy.ini 严格类型；uv.lock 锁定
- 运行时配置：AgentExecutionConfig（max_iterations/max_approval_rounds/retry_policy）、AgentStateConfig（双 store）、AgentMemoryConfig

## 治理
- AGENTS.md：提交前五层门（ruff format + flake8 + mypy + pytest + drasi pytest）
- GOVERNANCE.md / SECURITY.md / CODEOWNERS（CNCF Dapr 治理）
- LICENSE：Apache-2.0

## 测试体系
- tests/ 160 文件（pytest，-m "not integration"）；ext/ 需 DAPR_AGENTS_REQUIRE_DRASI=1
- 关键：tests/workflow/test_workflow_runner.py、tests/agents/、tests/tool/

## 外部依赖
- **dapr[fastapi,workflow]**（Dapr SDK）——durable 引擎在 Dapr runtime/sidecar
- 多 LLM provider SDK、MCP SDK、OpenTelemetry、FastAPI/uvicorn、cloudevents

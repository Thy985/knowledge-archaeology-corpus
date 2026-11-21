# Project Layer — dapr-agents 项目地图

## 一句话
Python 编写的多 Agent 分布式运行时框架：DurableAgent 把"LLM 推理 + 工具调用"包进 Dapr durable workflow，获得崩溃恢复、actor 并发、pub/sub 协作、OTel 观测；AgentRunner 提供 HTTP/pub-sub 入口；hooks/lifecycle 提供策略注入点。

## 架构总览（三层）
```
App / SDK 层：DurableAgent / AgentMetadata（schema 化配置）
    │
执行层：AgentRunner（HTTP /agent/run、/agent/instances）· subscribe()/serve() 入口
    │
运行时层（Dapr sidecar 1.18rc）：workflow（durable execution）· actors（virtual actor）
         · state store（agent-memory 会话 / agent-workflow 执行态）· pub/sub · service invocation
```

## 核心模块表
| 模块 | 职责 | 规模 |
|---|---|---|
| dapr_agents/agents/durable.py | DurableAgent：agent_workflow / orchestration_workflow / call_llm / run_tool / approval / broadcast | 4246 行 |
| dapr_agents/agents/base.py | AgentBase：MCP 资源自动发现 / registry / 工具装载 | 2217 行 |
| dapr_agents/agents/configs.py | AgentMetadata + AgentApprovalConfig + AgentExecutionConfig + 子元数据 | 943 行 |
| dapr_agents/agents/executors/ | AgentExecutorBase + ClaudeExecutor + DaprSessionStore + Echo | — |
| dapr_agents/hooks.py | HookContext 三型 + HookDecision 五态（Proceed/Skip/Mutate/RequireApproval/Deny） | — |
| dapr_agents/lifecycle.py | Lifecycle Protocol 扩展点（BEFORE_AGENT_INVOKE/BEFORE_TOOL_CALL 等，OSS-clean） | 极小 |
| dapr_agents/workflow/ | wf.WorkflowRuntime 封装 + AgentRunner + workflow_entry/message_router/http_router 装饰器 | — |
| dapr_agents/storage/ | daprstores / vectorstores / graphstores | — |
| dapr_agents/observability/ | OTel instrumentor + context_propagation | — |
| dapr_agents/tool/ | 工具注册执行 + workflow/activity_bridge + agent_tool（agent-as-tool） | — |

## 生命周期
```
start（idempotent：注入或自建 runtime → register_workflows → runtime.start() 容错"already running"）
  → _restore_pending_approvals（崩溃前未决审批恢复）
  → subscribe()/serve()/run（AgentRunner 入口）
  → agent_workflow 实例：TriggerAction 消息 → record_initial_entry →（orchestrator | executor | 标准循环）
  → stop（仅 owned runtime shutdown）
```

## 核心数据结构
- AgentMetadata（BaseModel，$schema draft 2020-12，版本化 index/latest/v0.11.1→v1.0.0）
- HookDecision 五态 dataclass；ToolMessage / ApprovalRequiredEvent / ApprovalResponseEvent / AgentStreamChunk
- DaprSessionStore 内部：_Manifest + chunk keys + append/load/delete/list_subkeys
- workflow entry state：entry.messages + entry.tool_history（instance_id 维度）

## 配置与外部依赖
- pyproject：dapr>=1.18.0rc0 / dapr-ext-fastapi / dapr-ext-workflow / openai / mcp>=1.26.0 / opentelemetry 1.35 / fastapi / docker / toolbox-core
- AgentApprovalConfig：pubsub_name（默认 None→HTTP /hitl/approvals）/ topic（agent-approval-requests）/ default_timeout_seconds（默认 300）
- 20+ LLM provider（openai/anthropic/nvidia/huggingface/elevenlabs/dapr…）；MCP client 三协议（stdio/sse/streamablehttp）

## 测试体系
- 129 单测（agents/document/executors/llm/memory/observability/prompt/storage/streaming/tool/types/utils/workflow）
- integration（API keys）+ ext（DAPR_AGENTS_REQUIRE_DRASI 门控）+ CI 8 workflows（build/integration-tests/ci-e2e-ollama/release/schema-compat/dapr-bot）

## 治理
- SECURITY.md（private vuln report）/ GOVERNANCE.md（CNCF）/ CODEOWNERS / DCO 强制 / AGENTS.md 提交门（ruff format + flake8 + mypy + pytest）
- schema-compat CI（agent-metadata 版本化兼容检查）

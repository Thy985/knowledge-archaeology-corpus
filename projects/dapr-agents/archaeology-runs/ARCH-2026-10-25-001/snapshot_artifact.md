# Snapshot Artifact — ARCH-2026-10-25-001（dapr-agents）

| 字段 | 值 |
|---|---|
| repository | https://github.com/dapr/dapr-agents.git |
| commit SHA | `8b4cc8ddbe2845248df5892fc532dd1097298be8` |
| commit 消息 | chore(typing): check types against Python 3.11 (#837) |
| commit 时间 | 2026-10-22 20:50:50 +0000 |
| branch | main |
| 版本 | dapr-agents（PyPI dynamic version）；schemas/agent-metadata 已发布 v1.0.0（latest.json）；requires-python >=3.11,<3.14 |
| license | Apache-2.0 |
| 克隆方式 | git clone --depth 1（浅克隆） |
| 时间戳 | 2026-10-25（本地） |

## 项目基础地图

### 语言与运行时
- Python（>=3.11,<3.14），uv 管理依赖（uv.lock 1.2MB）；mypy 类型检查针对 3.11（commit #837 即此）

### 入口与包结构
- 顶层包 `dapr_agents/`：agents（DurableAgent + AgentBase + executors + orchestrators）/ document / executors（Docker/Local CodeExecutor）/ hooks（HookContext + 5 决策类）/ lifecycle（Protocol 扩展点）/ llm（Anthropic/Dapr/OpenAI 等 ChatClient）/ memory / observability（OTel instrumentor）/ prompt / service / storage（daprstores/graphstores/vectorstores）/ streaming / tool（含 workflow activity_bridge + agent_tool）/ types / utils / workflow（runners + decorators）
- 核心入口：`dapr_agents.agents.durable.DurableAgent`（4246 行主类）；`__init__.py` 导出 DurableAgent/AgentMetadata/AgentExecutorBase/Hooks/决策类/ChatClients 等
- 示例：examples/ 32 个目录（01-llm-call-* → 13-agent-executor-claude + demo-otel-k8s + ext-drasi）；quickstarts/ 15 个文件（02_durable_agent.yaml 配置 + 03_durable_agent_http.py 中断恢复 demo + 04_durable_agent_pubsub.py）

### 核心模块职责
| 模块 | 职责 |
|---|---|
| agents/durable.py | DurableAgent：agent_workflow 主循环 / orchestration_workflow（多 agent）/ call_llm / run_tool / save_tool_results / approval（_await_approval）/ broadcast_to_team / start-stop |
| agents/base.py | AgentBase：MCP 资源自动发现（sidecar metadata mcp_servers）/ registry / tool 装载 |
| agents/executors/ | AgentExecutorBase（状态化 agent runtime）+ ClaudeExecutor + DaprSessionStore（manifest+chunk 持久化）+ EchoExecutor |
| agents/configs.py | AgentMetadata（$schema draft 2020-12 版本化 schema）/ AgentApprovalConfig / AgentExecutionConfig / Memory/Tool/LLM/PubSub/RegistryMetadata |
| agents/schemas.py | ApprovalRequiredEvent / ApprovalResponseEvent（结构化事件契约） |
| hooks.py | HookContext/LLMHookContext/ToolHookContext + HookDecision 五类（Proceed/Skip/Mutate/RequireApproval/Deny） |
| lifecycle.py | Lifecycle Protocol：BEFORE_AGENT_INVOKE/BEFORE_TOOL_CALL 等事件拦截扩展点（auth/OBO/HITL/observability，OSS-clean） |
| workflow/ | wf.WorkflowRuntime 封装（runners/agent.py + decorators） |
| storage/ | daprstores（Dapr state）/ vectorstores（embeddings）/ graphstores |
| observability/ | OTel instrumentor + context_propagation + wrappers |
| tool/ | 工具注册/执行 + workflow/activity_bridge（WorkflowContextInjectedTool）+ agent_tool（AgentWorkflowTool agent-as-tool） |

### 核心数据结构
- AgentMetadata（BaseModel，$schema 链接，版本化 agent-metadata/index.json + latest.json + v0.11.1→v1.0.0）
- HookDecision 五类 dataclass；ToolMessage/ApprovalRequiredEvent/ApprovalResponseEvent/AgentStreamChunk
- DaprSessionStore 内部：_Manifest（分块索引）+ chunk keys + append/load/delete/list_subkeys
- workflow entry state：entry.messages + entry.tool_history（instance_id 维度）

### 状态模型
- agent_workflow 三分支：orchestrator（call_child_workflow）→ executor（AgentExecutorBase 事件流）→ 标准循环（max_iterations 内 call_llm→tool_calls→hooks→run_tool→save_tool_results→next turn）
- 审批状态：ApprovalRequiredEvent 发布 → wait_for_external_event（approval_response_{id}）→ 超时/解析失败 auto-deny
- 生命周期：start（idempotent，注入 runtime 或自建；_restore_pending_approvals 崩溃恢复）→ stop（仅 owned runtime shutdown）

### 测试体系
- tests/ 129 个 test_*.py（agents/document/executors/llm/memory/observability/prompt/storage/streaming/tool/types/utils/workflow + integration + conftest.py + fake_state_store.py + scripts）
- CI：.github/workflows/build.yaml（ruff format/flake8/mypy/pytest not integration + DAPR_AGENTS_REQUIRE_DRASI=1 ext）、integration-tests.yaml（API keys）、ci-e2e-ollama.yaml、release.yml、schema-compat-comment.yaml、dapr-bot*
- AGENTS.md 强制提交门：`uv run ruff format && flake8 && mypy && pytest -m "not integration" && pytest ext`；DCO Signed-off-by 强制

### 配置
- pyproject.toml：dapr>=1.18.0rc0、dapr-ext-fastapi、dapr-ext-workflow、pydantic>=2.11.3、openai>=1.75.0、mcp>=1.26.0、opentelemetry（api/exporter/proto/sdk 1.35）、fastapi/uvicorn、docker、toolbox-core、numpy 等
- mypy.ini；.pre-commit-config.yaml；Makefile；.devcontainer

### 权限与治理机制
- SECURITY.md：漏洞走 private vulnerability report（Dapr security team），禁公开 issue
- GOVERNANCE.md：Dapr 治理（CNCF graduated 项目标准）
- CODEOWNERS + DCO（git commit -s 强制，bot 拦未签名 PR）
- AGENTS.md：代码标准（commit 格式/分支名/license header 逐字复制/语义化版本）
- AgentApprovalConfig：pubsub 或 HTTP 双通道 HITL；default_timeout_seconds=300 auto-deny；None=无限等待

### 外部依赖
- Dapr runtime sidecar（1.18.0rc0）——workflow/actors/state/pubsub/service-invocation 全部走 sidecar
- 20+ LLM provider 适配（openai/anthropic/huggingface/nvidia/elevenlabs/dapr 等，examples 01-llm-call-*）
- MCP（mcp>=1.26.0）——MCPServer auto-discovery + client（stdio/sse/streamablehttp，examples 06-*）
- OpenTelemetry（OTLP export）；Drasi（ext/，环境变量 DAPR_AGENTS_REQUIRE_DRASI 门控）

### 生命周期与版本治理
- schemas/agent-metadata/ 版本化（index/latest/v0.11.1→v1.0.0）——schema-compat CI 检查
- 语义化版本（AGENTS.md "use semantic versioning…backward compatability"）
- 周内密集发布信号：GetStarted 10-14 / launch 10-16 / pushed 10-22

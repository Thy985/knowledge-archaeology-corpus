# Repository Snapshot Artifact — ARCH-2026-10-30-001（dapr-agents）

## 快照事实
| 项 | 值 |
|---|---|
| Repository | https://github.com/dapr/dapr-agents.git |
| Commit SHA | `1fd68b3695f6b2fe89fa8739c30e9628e4633b94` |
| Branch | main（shallow clone depth 1） |
| Commit Message | "fix(deps): bump locked dependencies with open high and critical advisories (#933)" |
| Commit 时间 | 2026-10-27 21:45:29 +0000 |
| 版本 | v1.1.0（PyPI 实测 2026-10-30；pyproject dynamic 版本 setuptools_scm） |
| 语言 | Python >=3.11,<3.15（类型标注 + pydantic v2） |
| License | Apache-2.0（LICENSE 文件） |
| 快照时间 | 2026-10-30 |

## 规模
- 非测试 Python：51,516 行（dapr_agents/）
- 测试：160 文件 41,490 行（tests/；另有 ext/ 测试）→ 测试/代码比 0.81x
- 顶层：AGENTS.md / CODEOWNERS / CONTRIBUTING.md / GOVERNANCE.md / SECURITY.md / Makefile / dapr_agents / docs / examples / ext / quickstarts / schemas / scripts / tests / uv.lock

## 项目基础地图
### 语言与入口
- 纯 Python 包 `dapr-agents`；顶层导出：`dapr_agents/__init__.py` → `DurableAgent`（agents/durable.py:269，`class DurableAgent(DurableExecutorMixin, AgentBase)`）+ AgentTool/tool
- 仓库代码**无 `Agent` 类**（仅 AgentBase + DurableAgent）——"Agent deprecated"为文档层事实，代码已完全迁移（agents/__init__.py:14 仅导出 DurableAgent）

### 核心模块（crate 级 12 组）
| 模块 | 职责 |
|---|---|
| `agents/` | DurableAgent（base.py:134 AgentBase / durable.py:269 DurableAgent / durable_executor.py:124 DurableExecutorMixin / configs.py / schemas.py / executors/ 8 种 executor / orchestration+orchestrators LLM 编排） |
| `workflow/` | Dapr Workflow 绑定（runners/agent.py AgentRunner / runners/base.py / decorators/decorators.py / utils/{activities,core,grpc,names,pubsub,registration,routers,subscription}.py） |
| `llm/` | 多 provider（openai/anthropic/mistral/litellm/huggingface/nvidia/dapr/elevenlabs/iflytek）+ chat.py ChatClientBase |
| `memory/` | agent 记忆（会话历史） |
| `storage/` | 存储抽象（daprstores/graphstores/vectorstores） |
| `tool/` | AgentTool/tool 装饰器 + http/mcp/workflow/storage 工具族 |
| `service/` | fastapi 服务层（AgentRunner serve 模式） |
| `document/` | 文档加载（fetcher/reader/splitter/embedder） |
| `streaming/` | StreamEmitter + 事件流 |
| `observability/` | OpenTelemetry 封装 + wrappers |
| `types/` | 共享类型 |
| `utils/` | logger.py（WorkflowReplayFilter + get_context_aware_logger）、signal 等 |

### 核心数据结构（pydantic v2）
- `AgentWorkflowMessage`（agents/schemas.py）——agent 间/agent 内 workflow 消息
- `AgentStateConfig`（configs.py:246）/ `AgentMemoryConfig`（configs.py）——双 state store 配置
- `PlanStep` / `IterablePlanStep` / `NextStep`（orchestrators/llm/state.py+schemas.py）——LLM 计划执行状态
- `ApprovalRequiredEvent` / `ApprovalResponseEvent`（schemas.py）——HITL 审批事件
- `ExecutorRunInfo`（executor_run.py）——executor 运行记录

### 状态模型（durable execution）
- workflow 状态由 **Dapr Workflow 引擎**管理（dapr.ext.workflow，依赖 `dapr[fastapi,workflow]>=1.19.0rc4`）——event sourcing + 重放（workflow 代码在 await 外部 activity/tool 后重放重建本地状态）
- `_executor_workflow`（durable_executor.py:130）把 executor 回合编排为 workflow；`run_executor` 产生 RUN_WORKFLOW 消息
- `AgentRunner`（workflow/runners/agent.py）三模式：run（ad-hoc）/ subscribe（pub/sub）/ serve（FastAPI HTTP，port 8001）
- 双 state store：agent-memory（会话历史）+ agent-workflow（durable state，actorStateStore 必需）——由 AgentMemoryConfig/AgentStateConfig 配置

### 生命周期
- DurableAgent 创建 → AgentRunner 绑定 → run（创建 workflow 实例，返回 WORKFLOW_ID / instance_id）→ subscribe 订阅输入事件 → 每轮 = workflow 步骤（LLM 推理 + 工具调用 durable 化）→ serve 暴露 HTTP
- 中断恢复：进程崩溃/重启后按 WORKFLOW_ID GET /agent/instances/{id} → 从最后持久化步骤继续，不重放已完成 LLM/工具调用

### 测试体系
- pytest；`tests/` 160 文件（单元+部分集成，`-m "not integration"`）；ext/ 需 DAPR_AGENTS_REQUIRE_DRASI=1
- 核心测试：tests/workflow/test_workflow_runner.py（147 行）、tests/agents/、tests/tool/ 等
- 治理：AGENTS.md 必跑五层门（ruff format + flake8 + mypy + pytest + drasi pytest）；GOVERNANCE.md（Dapr 项目治理）；CODEOWNERS

### 配置
- pyproject.toml：deps 40+（dapr SDK 为核心；openai/anthropic/mistral/litellm 多 provider；mcp/websockets/httpx/aiohttp；opentelemetry 全家桶；cloudevents；numpy/tiktoken；posthog 遥测）
- optional-dependencies：drasi / claude / vectorstore
- mypy.ini 严格类型；uv.lock 锁定依赖

### 权限与治理机制
- AGENTS.md 开发规范（提交前必跑命令 = 质量门）
- GOVERNANCE.md / SECURITY.md / CODEOWNERS（CNCF Dapr 治理结构）
- 工具权限：AgentTool 封装 + tool 装饰器；MCP 工具接入（tool/mcp）
- HITL：ApprovalRequiredEvent/ApprovalResponseEvent（agent 请求审批）

### 外部依赖（关键）
- `dapr[fastapi,workflow]`（Dapr SDK）——**durable 引擎本体在 Dapr runtime/sidecar**，Python 侧为 SDK 绑定
- 多 LLM provider SDK；MCP SDK；OpenTelemetry；FastAPI/uvicorn；cloudevents（pub/sub 事件）

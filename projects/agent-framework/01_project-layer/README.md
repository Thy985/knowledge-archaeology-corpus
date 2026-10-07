# Project Layer — Microsoft Agent Framework（ARCH-2026-09-29-001）

## 1. 项目定位
生产级 agent 框架（MIT）：构建/编排/部署 AI agents 与多代理系统。Python（agent-framework-core 1.19.0）+ C#（1.23.0）+ Go（仅 README 存根）+ declarative-agents/ + docs/（ADR 41 条）。
来源：README、pyproject.toml、git log。

## 2. 顶层结构
| 目录 | 内容 |
|---|---|
| python/packages/ | 41 包：core + 40 集成包（anthropic/azure/bedrock/gemini/openai/mem0/mongodb/qdrant/redis/postgres/duckdb/hyperlight/cosmos-memory/orchestrations 等） |
| dotnet/src/ | C# 实现（Microsoft.Agents.*），Harness/ 与 Python _harness/ 同构 |
| declarative-agents/ | 声明式 agent 定义（YAML） |
| docs/decisions/ | 41 条 ADR |
| go/ | README 存根（占位） |

## 3. Python 核心包（聚焦面）
| 模块 | 行数 | 职责 |
|---|---|---|
| agent_framework/_agents.py | 2107 | BaseAgent/RawAgent 抽象、run、session、as_tool |
| agent_framework/_types.py | ~4400 | Message/ChatResponse/AgentResponse/ResponseStream/ToolMode |
| agent_framework/_harness/_loop.py | 1022 | AgentLoopMiddleware（循环 + Judge + 停止条件） |
| agent_framework/_harness/_tool_approval.py | 737 | 工具审批状态机（ToolApprovalRule/State/Middleware） |
| agent_framework/_harness/_file_access.py | 2235 | 文件存取（路径规范化 + 有界搜索 + 行编辑） |
| agent_framework/_harness/_memory.py | 1702 | 文件记忆（MemoryIndexEntry/MemoryTopicRecord） |
| agent_framework/_harness/_mode.py | 389 | AgentModeProvider（模式切换） |
| agent_framework/_harness/_todo.py | 588 | TodoItem 任务清单 |
| agent_framework/_harness/_background_agents.py | 707 | 后台 agent 任务 |
| agent_framework/security.py | ~3500 | FIDES（标签传播/策略执行/变量隔离/quarantine） |
| agent_framework/_workflows/ | ~4300 | Workflow DAG（builder/context/executor） |
| agent_framework/_compaction.py | ~400 | 上下文压缩（工具调用配对分组） |
| agent_framework/_sessions.py | ~700 | AgentSession（序列化/状态类型注册） |
| agent_framework/_mcp.py | ~600+ | MCP 集成（命名/大小预算/header 客户端） |
| agent_framework/_skills.py | ~800 | Skill/SkillResource/SkillScript |

## 4. orchestrations 独立包（5 编排模式）
| 文件 | 机制 |
|---|---|
| _sequential.py | SequentialBuilder → Workflow（输入转会话 + 参与者链） |
| _concurrent.py | ConcurrentBuilder（DispatchToAllParticipants + Aggregator） |
| _group_chat.py | GroupChatState/Orchestrator（选择函数决定下一发言人）+ AgentBased（LLM 选） |
| _handoff.py | HandoffConfiguration + AutoHandoffMiddleware（拦截 + 短路合成结果） |
| _magentic.py | MagenticManager（任务 ledger + 进度 ledger + 团队块 + plan/replan） |

## 5. 生命周期（Agent 执行）
1. Agent 构造（id/name/description/options）→
2. create_session/get_session（AgentSession：service_session_id + state）→
3. run(input, session)（middleware 链：ContextProvider.before_run → AgentMiddleware.process → call_next → after_run）→
4. 响应 AgentResponse/stream → 会话更新（_update_session_from_chat_response）
来源：_agents.py run/_update_session_from_chat_response；_middleware.py AgentContext/AgentMiddleware。

## 6. 状态
- AgentSession.state（provider-scoped：审批状态/工具预算/后台任务/模式/todos）+ service_session_id
- _workflows：WorkflowRunResult + status_timeline + checkpoint（on_checkpoint_save/restore）
- orchestrations：_orchestration_state.py（编排请求/状态模型）
来源：_sessions.py；_workflows/_workflow.py WorkflowRunResult；orchestrations/_orchestration_state.py。

## 7. 测试体系
- core tests：96 个 test_*.py 文件（tests/core/），harness 专项 10 个（loop/file_access/file_memory/memory/mode/todo/tool_approval/background_agents/agent/compaction）
- orchestrations tests：8 个
- 本机实测：test_harness_file_access.py -k "rejects_traversal or symlinks or normalize" 7 passed；test_harness_tool_approval.py -k "authoritative or changed_hidden_snapshot or principal_change" 7 passed
- 测试揭示：路径穿越/符号链接拒绝、审批 authoritative 边界、changed hidden snapshot 二次可见审批、principal change 二次审批

## 8. 配置
- _settings.py：load_settings（env/文件源 + 类型强制 + SecretString）
- pyproject：core 1.19.0；可选依赖按平台（hyperlight: linux x86_64/win AMD64）
- _feature_stage.py：实验特性标记（ExperimentalWarning）

## 9. 权限与治理机制（Authority 面）
- ToolApprovalMiddleware：审批规则（名称/参数匹配）+ 函数调用预算 + 自动批准队列 + 二次审批（hidden snapshot/principal change）
- FileAccess：相对路径规范化（拒绝 rooted/drive/`.`/`..`）+ 有界搜索（deadline 防超时）
- FIDES（security.py + ADR 0024）：Integrity/Confidentiality 标签 + LabelTracking 传播 + PolicyEnforcement 执行（allow_untrusted_tools 白名单）+ ContentVariableStore 变量隔离 + quarantine 隔离执行
- MCP：ToolAnnotations hint 自动标签 + _meta.ifc result 标签

## 10. 外部依赖
- core：msgspec/opentelemetry/pydantic（按特性）；集成包按供应商（anthropic/azure/openai/bedrock/gemini）
- hyperlight（CodeAct 沙箱）：仅 linux x86_64/win AMD64 + Python<3.15
- azure-cosmos-memory：需要 azure-cosmos-agent-memory（Python 3.11+）

## 11. 重要设计与 ADR 证据
- ADR 0024：FIDES 确定性提示注入防御（信息流控制 vs prompt 工程；chosen=FIDES）
- ADR 0038：CodeAct 集成（模型写代码 + 沙箱隔离；"backend 无隔离则非合适 CodeAct backend"）
- ADR 0037：Agent Skills（AgentSkillResource/Script 独立模型，不用 AIFunction——参数/审批/循环引用三理由）
- ADR 0019：context compaction strategy（_compaction.py 工具调用配对分组实现）
- ADR 0026/0029/0039：hosted session identity / Python agent session identity / session store serialization

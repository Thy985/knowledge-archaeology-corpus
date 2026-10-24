# Engineering Knowledge Graph — dapr-agents（L1/L2 宽底座，20 条）

> 每条含机制/证据/links；证据锚 = 仓库文件:行（durable.py = dapr_agents/agents/durable.py）。
> 缩写：base.py=dapr_agents/agents/base.py；configs.py=dapr_agents/agents/configs.py；sstore=dapr_agents/agents/executors/dapr_session_store.py

## EK-D01 · durable workflow 主循环（I/O activity 化）
- **机制**: agent_workflow 由 pub/sub TriggerAction 消息触发；一切 I/O（record_initial_entry / call_llm / run_tool / save_tool_results / publish_approval_request / load_tools / get_team_members）均 yield ctx.call_activity，带 retry_policy；"Record initial entry via activity to keep deterministic/replay-friendly I/O"
- **证据**: durable.py:590-618（agent_workflow 入口 + record_initial_entry activity）
- **links**: mechanism↔EK-D02（都是 replay 纪律）；causal→EK-D03（主循环三分支）

## EK-D02 · activity 幂等语义（重放检测 + 发布缓存）
- **机制**: ①call_llm 用 _call_llm_message_id 定位已记录消息，activity 重放时返回缓存结果，"不会再次追加 user turn 并二次调用 LLM（会留下第一次执行的 tool_calls 无 tool replies）"；②publish_approval_request 无条件 yield，"DurableTask returns the cached result on replay without re-executing"
- **证据**: durable.py:2300-2316（call_llm 重放检测）；durable.py:1275-1281（publish 缓存）
- **links**: mechanism↔EK-D01（I/O 幂等是 replay 前提）；causal→EK-D19（崩溃恢复的机制底座）

## EK-D03 · 执行模型三分支
- **机制**: 同一 agent_workflow 按配置分派：orchestrator（_orchestration_strategy → call_child_workflow）→ executor（AgentExecutorBase 状态化 runtime，驱动事件流）→ 标准循环（max_iterations 内 call_llm→hooks→run_tool→save_tool_results）
- **证据**: durable.py:660-716（三分支 if/elif/else）
- **links**: subsystem↔EK-D13（executor 分支）；causal→EK-D09（save 语义随分支变）

## EK-D04 · hooks 五态决策 + dispatch 前收集
- **机制**: before_tool_call hook 对**本轮全部** tool_calls 先收集决策再分派（防部分执行）；决策五态：Proceed（放行）/Skip（hook 提供结果直接使用）/Mutate/RequireApproval（挂起等人工）/Deny（合成 denial ToolMessage "blocked by policy"）。**Mutate 三 slot 语义（R-3 细化）**：before_tool_call 时 payload **完全替换**工具参数（工具只见 payload）；before_llm_call 时 payload **浅合并** generate kwargs（tools/response_format/tool_choice 保留）；after_llm_call 时 payload **替换** assistant 消息（消息是单一体，不做部分合并）
- **证据**: durable.py:704-760（hook pass 收集）；durable.py:772-816（Deny/Skip/Mutate 消费）；hooks.py:86-103（Mutate 语义）
- **links**: mechanism↔EK-D15（决策型 vs 拦截型扩展双轨）；causal→EK-D05（RequireApproval 触发审批）

## EK-D05 · HITL approval fail-closed
- **机制**: approval_request_id = uuid5(NAMESPACE_DNS, instance:tool_call_id)（replay 一致）；timeout 竞速（when_any event/timer）→ 超时 auto-deny；响应解析异常 auto-deny；default_timeout_seconds=300；None=无限挂起；事件名 approval_response_{id}
- **证据**: durable.py:1215-1260（_await_approval 主体）；configs.py:528-540（AgentApprovalConfig）
- **links**: causal→EK-D19（pending approvals 持久化恢复）；constraint→EK-D04（仅 RequireApproval 才启用）

## EK-D06 · ask_user 无流短路
- **机制**: ask_user 需 live stream 通道（emit USER_INPUT_REQUESTED + external event 挂起）；无 stream_ctx 时直接返回"无法交互，继续"ToolMessage，防 600s timeout 死端
- **证据**: durable.py:817-836（"Short-circuit at call time"）
- **links**: contrast↔EK-D05（交互式 vs 审批式外部输入）；subsystem↔EK-D11（stream_ctx 同源）

## EK-D07 · WorkflowContextInjectedTool 内联执行
- **机制**: 需要 ctx 的工具（AgentWorkflowTool 等）在 workflow 内联执行（activity 无 workflow context）；AgentWorkflowTool（agent-as-tool）分配 child_instance_id + 继承 stream context + depth+1；run_tool 内加守卫（注册漂移时给可操作报错而非 "Missing workflow context"）
- **证据**: durable.py:838-890（内联分派）；durable.py:3008-3022（run_tool 守卫）
- **links**: mechanism↔EK-D13（executor 事件流可替代）；constraint→EK-D01（活动内不可执行的例外）

## EK-D08 · run_tool 异常转 ToolMessage
- **机制**: 工具执行异常不抛给 workflow，而是合成 error ToolMessage（err_type: msg）；"确保每个 tool_call_id 有对应 tool message，维护对话历史"；MCP kwargs wrapper 解包（单键 kwargs 展开）
- **证据**: durable.py:3022-3047（try/except → error_msg → ToolMessage）
- **links**: dependency→EK-D09（save 的输入完整性）；constraint→EK-D01（对话链不破）

## EK-D09 · save_tool_results skip_messages 双模式
- **机制**: 常规模式：tools 并行执行后结果 append 到 entry.messages（[user, assistant+tool_calls, tool(A), tool(B)] → 下轮 LLM）；orchestration 模式（skip_messages=True）：无前置 assistant+tool_calls，写 entry.messages 会破坏 OpenAI "tool message 必须紧跟匹配 assistant"约束（HTTP 400），只写 tool_history
- **证据**: durable.py:3054-3110（docstring 完整双模式说明）
- **links**: constraint→EK-D10（消息链完整性）；subsystem↔EK-D03（orchestrator 分支专用）

## EK-D10 · 结构化输出 mid-tool-loop 省略 tools
- **机制**: 请求结构化输出（response_format）且上一条 assistant 无 tool_calls 时，省略 tools/tool_choice——"so the model can't legitimately respond with a tool call instead of JSON"
- **证据**: durable.py:2352-2378（generate_kwargs 构造）
- **links**: dependency→EK-D09（消息链正确性的延伸）；causal→EK-D18（orchestration 结构化规划）

## EK-D11 · stream_ctx frozen per-session
- **机制**: stream_ctx 在 _resolve_stream_context 冻结一次（per-session 决策），后续发布/终结全部以 frozen 为准，"re-reading the live flag mid-session let content deltas stop while control+terminal chunks kept flowing（partial, confusing stream）"——历史 bug 修复
- **证据**: durable.py:2366-2376（stream_ctx 注释）
- **links**: mechanism↔EK-D06（流式依赖）；contrast↔EK-D02（frozen vs replay 缓存同为一致性手段）

## EK-D12 · DaprSessionStore manifest+chunk 分块
- **机制**: session 数据以 manifest（chunk 索引）+ 多 chunk 键分块写 Dapr state；append/load/delete/list_subkeys；防单 key 超限；"started in one workflow activity can resume in a retried activity"（activity 重试续读）
- **证据**: sstore:20-24（docstring）、171-197（_Manifest/_AppendPlan/_entry_uuid/_chunk_entries）、254-275（append/load API）
- **links**: mechanism↔EK-D17（双 store 分层）；dependency→EK-D13（executor 会话恢复）

## EK-D13 · AgentExecutorBase 状态化 agent runtime
- **机制**: executor 拥有 tool/reasoning 循环（run 事件流），durable 驱动其事件流并在此处理 tool approvals（supports_tool_approval）；bind(ExecutorBinding)；get_session(session_id)；Claude executor 实现（claude_events/transcript）。**实现文件定位（R-2）**：`_executor_workflow` 定义在 **dapr_agents/agents/durable_executor.py:121**（非 durable.py）；durable.py:674 仅调用点
- **证据**: dapr_agents/agents/durable_executor.py:121（_executor_workflow）；dapr_agents/agents/executors/base.py:35-112；durable.py:674-676（executor 分支调用点）
- **links**: subsystem↔EK-D03（第二分支）；mechanism↔EK-D12（会话持久化支撑）

## EK-D14 · MCP auto-discovery
- **机制**: AgentBase 从 Dapr sidecar metadata API 读取 resp.mcp_servers，自动发现 MCPServer 资源并装载工具（"Zero-config MCP integration — agents auto-discover MCPServer tools from the Dapr sidecar"）；三协议 client（stdio/sse/streamablehttp examples 06-*）
- **证据**: base.py:530-547（_discovered_mcpserver_names）；README（MCP 特性段）
- **links**: mechanism↔EK-D16（资源声明化）；contrast↔EK-D07（外部工具 vs 框架工具）

## EK-D15 · lifecycle Protocol 扩展点（OSS-clean）
- **机制**: lifecycle.py 暴露最小 Protocol，DurableAgent 在已知事件（BEFORE_AGENT_INVOKE/BEFORE_TOOL_CALL 等）调用；实现方插 auth/OBO/HITL approver/observability 而不改 DurableAgent；"stays OSS-clean: no proprietary or plugin-specific imports"；DecisionDict 镜像 hooks decisions 序列化字段
- **证据**: lifecycle.py:24-43（docstring 设计注）
- **links**: contrast↔EK-D04（拦截型 vs 决策型）；mechanism↔EK-D13（executor 也可消费）

## EK-D16 · AgentMetadata schema 化 + 版本化
- **机制**: 配置是 BaseModel + $schema（draft 2020-12）；agent-metadata 目录版本化（index/latest/v0.11.1→v1.0.0）；schema-compat CI 检查
- **证据**: configs.py:46-47（_JSON_SCHEMA_KEY）；schemas/agent-metadata/（文件列表）
- **links**: mechanism↔EK-D14（声明式资源）；constraint→EK-D20（CI 强制兼容）

## EK-D17 · 双 state store 语义
- **机制**: agent-memory（会话历史，Dapr state）+ agent-workflow（durable execution state，需 actorStateStore: true）——会话记忆与执行状态物理分层；README "two state stores"（getting-started 核心概念）
- **证据**: README（"The workflow engine... two separate state stores" 语义）；quickstarts/02_durable_agent.yaml（组件配置）
- **links**: mechanism↔EK-D12（session store 是 agent-memory 实现路径）；subsystem↔EK-D19（workflow state 是恢复源）

## EK-D18 · orchestrator 结构化规划
- **机制**: orchestration_workflow 主函数（**R-1 补锚** durable.py:1332）经 call_child_workflow 被 agent_workflow 调用：get_team_members →（AgentOrchestrationStrategy 时）TASK_PLANNING_PROMPT + call_llm(response_format=IterablePlanStep, STREAM_PHASE=planning) → plan 解析 → agent 循环；策略活动区（initialize_orchestration 1868 / select_next_task 1884 / handle_response / check_completion / finalize_orchestration）；plan 从 messages 提取（get_plan_from_messages，IterablePlanStep/NextStep/ProgressCheckOutput 结构化模型）
- **证据**: durable.py:1332-1360（orchestration_workflow 主体）；durable.py:1868-1955（策略活动）；durable.py:1998-2183（plan 提取）
- **links**: causal→EK-D10（结构化输出支撑规划）；subsystem↔EK-D03（orchestrator 分支实现）

## EK-D19 · 崩溃恢复链
- **机制**: start() 幂等（注入/自建 runtime、already-running 容错、仅 owned shutdown）；_restore_pending_approvals 恢复崩溃前未决审批（_persist_pending_approvals 对称）；replay 缓存（EK-D02）保证恢复不重放
- **证据**: durable.py:3950-3990（start/stop）；durable.py:3274-3332（_persist/_restore_pending_approvals）
- **links**: causal←EK-D02（幂等是恢复前提）；causal←EK-D05（审批也要恢复）

## EK-D20 · 治理代码化 + schema 兼容门
- **机制**: AGENTS.md 强制提交门（ruff format / flake8 / mypy / pytest -m "not integration" / ext Drasi）；DCO Signed-off-by 强制（bot 拦未签名 PR）；语义化版本（backward compatibility 优先）；license header 逐字复制要求
- **证据**: AGENTS.md（Code Standards + Before commit REQUIRED 段）
- **links**: mechanism↔EK-D16（schema 版本化是兼容治理的一部分）；subsystem↔GOVERNANCE.md/SECURITY.md

## EK 图统计
- EK 20 条；links 覆盖率 100%；avg 出边 ~2.0；游离 0
- 反退化检查：每条有机制+证据+≥1 边；无"同子系统=聚合理由"

# Flow Atlas — dapr-agents 七类流（从真实代码导出）

> 关键 Edge 可回溯 symbol/file/condition。

## 1. Control Flow（控制流）
```
TriggerAction pub/sub 消息 → DurableAgent.agent_workflow(ctx, message)
  → record_initial_entry（activity，replay-friendly I/O）
  → 三分支：
      A) orchestrator（_orchestration_strategy）→ call_child_workflow(orchestration_workflow)
      B) executor（AgentExecutorBase）→ _executor_workflow（事件流驱动）
      C) 标准循环：for turn in 1..max_iterations
           call_llm（activity）→ tool_calls?
             → hook pass（before_tool_call 全量收集决策）
             → run_tool / 内联 WorkflowContextInjectedTool（并行/串行）
             → save_tool_results → 下轮
  → finalize_workflow → publish_stream_event
```
- **关键 Edge**: durable.py:660-716（三分支）；retry_policy 全 activity 统一

## 2. State Flow（状态流）
```
workflow entry（instance_id 维度）：entry.messages + entry.tool_history
  → call_llm 写入 assistant；save_tool_results 追加 tool（或仅 tool_history）
  → DaprSessionStore：manifest + chunk 分块持久化（agent-memory 会话）
  → pending approvals：_persist_pending_approvals / _restore_pending_approvals
  → actorStateStore=true 的 agent-workflow（durable execution state）
```
- **关键 Edge**: sstore:254-275（append/load）；durable.py:3274-3332（审批持久化）

## 3. Data Flow（数据流）
```
Trigger message（task + metadata + stream_context）
  → call_llm：build_initial_messages(user_input + chat_history) → LLM provider
  → assistant_response{tool_calls[]} → 序列化工具结果（serialize_tool_result）
  → ToolMessage → save_tool_results → entry
MCP：sidecar metadata mcp_servers → tool 注册 → MCP client（stdio/sse/streamablehttp）
```
- **关键 Edge**: base.py:530-547（MCP 发现）；durable.py:3047-3054（serialize_tool_result）

## 4. Evidence Flow（证据流）
```
record_initial_entry（activity，instance/source/trace_context）
  → call_llm message_id（_call_llm_message_id）→ _find_recorded_message（重放检测）
  → tool_history（全量工具调用记录，observability）
  → OTel span（_otel_span_context 传播，context_propagation）
```
- **关键 Edge**: durable.py:2300-2316（重放检测）；observability/context_propagation.py

## 5. Authority Flow（权威流）
```
hooks 决策（before_tool_call）：Proceed（放行）→ tool 执行
                                    Deny → 合成 denial ToolMessage（"blocked by policy"）
                                    RequireApproval → _await_approval
                                      → ApprovalRequiredEvent 发布（pubsub 或 HTTP /hitl/approvals）
                                      → wait_for_external_event(approval_response_{id})
                                      → timeout/timer 竞速 → auto-deny
lifecycle Protocol：BEFORE_AGENT_INVOKE/BEFORE_TOOL_CALL 拦截（auth/OBO/HITL approver）
```
- **关键 Edge**: durable.py:1215-1260（_await_approval）；configs.py:508-540（AgentApprovalConfig 双通道）

## 6. Memory Flow（记忆流）
```
会话历史（entry.messages）→ DaprSessionStore（manifest+chunk）
  → _reconstruct_conversation_history（activity 重试续读）
  → 下一轮 call_llm 的 chat_history
executor 路径：DaprSessionStore.get_session(session_id)（跨 activity 恢复）
```
- **关键 Edge**: durable.py:2318（_reconstruct_conversation_history）；sstore:20-24（resume in retried activity）

## 7. Policy Flow（策略流）
```
AGENTS.md 提交门（ruff/flake8/mypy/pytest）→ CI build.yaml
  → DCO Signed-off-by 强制（bot 拦未签名 PR）
  → schema-compat CI（agent-metadata 版本兼容）
  → SECURITY.md private vuln report（Dapr 安全团队）
  → AgentApprovalConfig（HITL 策略运行时配置）
```
- **关键 Edge**: AGENTS.md（Before commit REQUIRED）；.github/workflows/schema-compat-comment.yaml

## Flow 真实性声明
- 每条 Edge 可回溯文件锚（本文件 + EK Graph 证据列）；无凭记忆 Flow

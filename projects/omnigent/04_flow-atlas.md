# 04 · Flow Atlas — Omnigent（七类流）

> 从真实代码/文档导出；每条 Edge 标注可回溯 symbol / file / condition。Policy Flow 为第七类流（v2）。

## 04.1 Control Flow（控制流）

```
用户消息(web/CLI/phone)
  → server（FastAPI app.py）创建/路由会话
  → runner 启动（runner/_entry.py）→ _run_agent_loop（runtime/workflow.py）
      → turn 开始 → engine.reset_turn（runtime/policies/engine.py）
      → 构建 prompt → 调 LLM → 工具执行（runner/tool_dispatch.py）
          → should_dispatch_locally? ──否──→ 原样上送 action_required（可见性）
          └──是──→ 五类分发（OS env/REST/FILE/TERMINAL/MCP）
              → policy gate（runner/policy.py）→ ALLOW 继续 / DENY 返回拒绝文本 / ASK 挂起
      → 循环至 terminal assistant response → 持久化（stores/）
  → server 经 SSE/事件流推送（transcript forwarder）
```
关键 symbol：`_run_agent_loop`（runtime/workflow.py）、`proxy_stream`（runner）、`execute_tool`（runner/tool_dispatch.py）、`should_dispatch_locally`。

## 04.2 State Flow（状态流）

```
会话创建 → stores/ SqlConversation（db_models.py）
  → per-harness 状态文件（claude_native_state.py / codex_native_state.py / …）
  → runner 会话状态（session_lifecycle.py / suspend_watch.py / resume_dispatch.py）
  → server feature_flags.py（特性开关）
  → labels：PolicyEngine hot-cache（workflow 生命周期）→ write-through conversation_labels
  → 沙箱状态：managed hosts 行（launch-token digest + expiry + provider + sandbox id）原地覆盖
```
关键 symbol：`apply_label_writes`（engine）、`register_managed_host`（stores/host_store.py）、`SqlConversation`。

## 04.3 Data Flow（数据流）

```
模型选择：model_resolver.resolve_model（ModelResolutionSource 优先级：EXPLICIT→…→live probe）
  → model_fallbacks 静态表（owner/provenance/discovery_gap；绝不发明 live 列表外行）
  → model_catalog / model_metadata（能力/成本层）
工具数据：MCP 工具经 RunnerMcpManager（spec 定义，名字随 spec 变）
  → runner 本地执行 / server REST（sys_call_async / sys_cancel_async）/ 文件 API（sys_upload/download/list_files）
CLI 数据：stdout=数据、stderr=装饰（designs/CLI_CONTRACT.md）
```
关键 symbol：`ModelResolutionSource`、`StaticModelFallback`、`RunnerMcpManager`、`sys_call_async`。

## 04.4 Evidence Flow（证据流）

```
审计：debug_logging.audit_event_logger + current_request_audit_attrs（server/app.py import）
  → request 级 audit attrs（set_current_session_id / set_current_user_id）
遥测：runtime/telemetry.py
  → 各层从共享 response_id 独立推导 trace_id（trace_id_from_response_id，resp_ 前缀校验）
  → 现状：trace 不跨 wire 传播（OBSERVABILITY.md 自我审计）；HTTPX instrumentor 未 wire；get_traceparent_env dead code
诊断：_native_forwarder_health 单槽记录（monotonic 时间戳 + 成功清槽）
```
关键 symbol：`trace_id_from_response_id`（telemetry.py:508）、`_RESP_PREFIX`、`record_post_failure`/`note_post_success`。

## 04.5 Authority Flow（权威流）

```
用户认证（server/auth.py + accounts_*）→ 会话权限（SqlSessionPermission，db_models.py）
  → 沙箱：launch token（每次 launch mint，HostStore.register_managed_host；用户凭据不进沙箱）
  → 工具权限：policy gate（runner/policy.py）→ ALLOW/DENY/ASK
  → ASK 权威：POST evaluate_policy=True → server 独立重评 → elicitation（4 通道：HOOK/JSONRPC/APPROVAL_MIRROR）
  → 裁决回传：approval 事件经 server/routes/_sessions/ 事件路由（elicitation/approval 族：_antigravity_elicitation.py / _codex_elicitation.py / orchestration.py）→ pending_approvals.resolve（幂等）
  → 拒绝可见性：DENY 文本作为 tool output 返回（LLM 看到拒绝）
```
关键 symbol：`evaluate_policy=True`、`pending_approvals.register/cleanup/resolve`、`_DEFAULT_WAIT_SECONDS=86400`、`server/routes/_sessions/`（Auditor 修正字面 URL）。

## 04.6 Memory Flow（记忆流）

```
会话消息：stores/conversation_store/sqlalchemy_store.py（SqlConversationItem 等，uuid_to_bytes/compression CompressedText）
  → 会话历史检索（sys_session_get_history，content_max_chars ≤12000）
策略记忆：labels（conversation_labels）+ session state（set_session_state）+ SqlUserDailyCost（成本累计）
  → PolicyEngine label hot-cache → write-through
```
关键 symbol：`ConversationStore`、`CompressedText`（db/compression.py，zstandard）、`set_session_state`。

## 04.7 Policy Flow（治理闭环：Decision→Approval→Policy→Enforcement→Future Decision）

```
Decision（设计）：
  designs/RUNNER_MCP.md → runner 拥有 MCP dispatch
  designs/RUNNER_TOOL_DISPATCH.md → runner 本地分发 + action_required 原样上送
Approval（治理门槛）：
  生产变更走 PR（CONTRIBUTING.md + designs/contributor-review-merge-proposal.md + CI pre-commit）
Policy（策略固化）：
  策略系统本体：PolicySpec（声明式）→ Policy（纯求值器，无状态无 DB I/O）→ PolicyEngine（组合）
  → 内置策略库：policies/builtins/（safety/cost/risk_score/orchestration/routing/working_dir/cel/github/google/context/prompt/_shell）
Enforcement（执行）：
  runner 侧（function 型 tool_call/tool_result 经 _GatedPolicy per spec_hash 缓存）
  server 侧（label/prompt 型；elicitation 通道）
Future Decision（反馈）：
  CHANGELOG bugfix（#1119 诊断、#4457 缓存串扰、#2752 runner 重连）→ 新设计决策
  OBSERVABILITY.md 审计 → tracing 改造计划（proposed）
```
关键 symbol：`PolicySpec`、`_GatedPolicy`、`POLICY_REGISTRY`（registry.py）、`build_policy_engine`、`_e2e_policy_callables.py`。

## Flow→KO 交叉校验

| KO | 依赖 Flow 段 | 一致性 |
|----|-------------|--------|
| KO-02 策略执行点跟随控制权 | 04.5（runner gate）+ 04.7（RUNNER_MCP） | ✅ |
| KO-03 人机闸门 | 04.5（ASK 循环）+ 04.7（enforcement 双面） | ✅ |
| KO-06 身份持久+凭据隔离 | 04.5（launch token） | ✅ |
| KO-07 失败可解释 | 04.4（健康记录/遥测）+ 04.6（记忆持久化） | ✅ |

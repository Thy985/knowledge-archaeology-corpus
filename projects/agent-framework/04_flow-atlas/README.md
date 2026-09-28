# Flow Atlas — 七类流（ARCH-2026-09-29-001）

> 全部从真实代码导出；关键 Edge 标注 symbol/file/condition/state transition。

## 1. Control Flow（控制流）
```
Agent.run(input, session)
  → _middleware 链（AgentMiddleware.process）
    → AgentLoopMiddleware.process            [_harness/_loop.py|419]
        ├─ non-stream: 迭代循环
        │    ├─ _restore_session(fresh_context)  [_loop.py|398]
        │    ├─ call_next → 模型
        │    ├─ _evaluate_stop(max_iterations 先短路)  [_loop.py|748]
        │    ├─ _has_pending_approval_request → return  [_loop.py|447]
        │    └─ _record_progress → 注入 progress  [_loop.py|732]
        └─ streaming: _process_streaming（同样传 snapshot，streaming 下 fresh_context 语义一致——独立审计 IF-05 补强）
    → ToolApprovalMiddleware.process          [_tool_approval.py|399]
        ├─ 需要 AgentSession（否则 RuntimeError）
        ├─ _drain_auto_approvable_queue → _pop_next_queued_request
        └─ while: _inject_collected_responses → call_next → _process_outbound_messages
             （全部自动批准且无其他用户输入 → 清空重跑，否则 return）
```
关键 Edge：AgentLoop 与 ToolApproval 是两个**嵌套中间件循环**——内循环（审批）可能让外循环（agent loop）的同一迭代反复重跑。

## 2. State Flow（状态流）
```
AgentSession { service_session_id, state: dict }
  ├─ ToolApprovalState       [_tool_approval.py|176]  （source_id 作用域，审批规则/队列/预算）
  ├─ _FUNCTION_INVOCATION_BUDGET_STATE_KEY   （工具调用预算，注入 client_kwargs）
  ├─ BackgroundTaskInfo[]    [_harness/_background_agents.py|57]  （launch/refresh/finalize/abandoned）
  ├─ AgentModeState          [_harness/_mode.py|223]
  ├─ TodoItem[]              [_harness/_todo.py|54]
  └─ fresh_context: session.to_dict() → 迭代前 from_dict 重建  [_loop.py|419-460]
```
状态迁移：审批状态持久化到 session（_save_state）→ 跨中间件/跨迭代存活；fresh_context 下每次迭代**重建** session 状态（快照恢复）。

## 3. Data Flow（数据流）
```
input: str | ChatResponse | list[Content]  →  _types.py 归一化
  → context.input_messages（ContextProvider.before_run 可注入记忆）
  → 模型请求（client.chat.get_response）
  → AgentResponse / ResponseStream（_GatedResponseStream 门控）
  → _update_session_from_chat_response（回写会话）
```
工具调用数据：function call → ToolApprovalMiddleware 出站处理（_process_outbound_messages）→ 批准 → 执行 → 结果作为 tool 消息回注入。

## 4. Evidence Flow（证据流）
```
证据输入面：
  - 测试（test_harness_file_access/tool_approval 等 96+8 文件）——可执行证据
  - ADR（docs/decisions/0001-0043）——设计意图证据（非实现事实）
  - 代码注释（_normalize_relative_path 的隔离边界警告、_loop.py 的 C# 镜像注释、_context_provider.py 的注入路径注释）
  - docs/decisions/0024 的 FIDES 四组件与 security.py 实现对应
证据验证：本机 pytest 两轮抽样 7 passed（file_access 安全 / tool_approval 审批语义）——ADR 意图 → 代码 → 测试三层闭环。
```

## 5. Authority Flow（权威流）
```
工具执行权威链：
  ToolApprovalMiddleware（审批状态机）          [_tool_approval.py|361]
    ├─ 规则匹配（名称+参数）→ 自动批准/排队/转交
    ├─ 函数调用预算（session 级）
    └─ 二次审批（hidden snapshot / principal change）[测试 485/635]
  PolicyEnforcementFunctionMiddleware（FIDES）  [security.py|2351]
    ├─ 非可信上下文默认阻止（allow_untrusted_tools 白名单）
    ├─ 保密需求校验（check_confidentiality_allowed）
    └─ audit log / approval_on_violation
  文件访问：_normalize_relative_path 拒绝穿越/根/符号链接   [_file_access.py|264]
  CodeAct：沙箱（Hyperlight）内 execute_code，模型代码 = untrusted  [_provider.py|24]
```
关键 Edge：**审批权威有边界**——session 归属决定审批是否 authoritative（test_manual_fides_no_session_does_not_make_approval_authoritative）；无 session 的审批不成为权威。

## 6. Memory Flow（记忆流）
```
写入：after_run → upsert_memory（user/assistant/system turns，跳过空）  [_context_provider.py|448]
  ├─ auto_extract=True → cadence-aware 后台抽取（FACT_EXTRACTION_EVERY_N 等阈值）
  └─ user_id 作用域：state["user_id"] > session_id > "default"
检索：before_run → search_cosmos(user_id, top_k, memory_types, min_confidence)
  └─ procedural（任务感知程序，toolkit 0.3.0b2 单独编译）
注入：记忆 + user_summary 以 **user 角色** untrusted 注入  [_context_provider.py|341-450]
文件记忆：MemoryTopicRecord → to_markdown（# topic + Updated + Sessions + ## Summary + ## Memories）→ _escape_markdown_line 防节注入  [_memory.py|452]
```

## 7. Policy Flow（策略流）
```
治理闭环（ADR 0024 + security.py 实现）：
  Decision（FIDES：标签组合 most-restrictive-wins）
    → Policy（ToolAnnotations hint 映射 source_integrity/accepts_untrusted/max_allowed_confidentiality）
    → Enforcement（PolicyEnforcementFunctionMiddleware 执行前检查 + audit_log）
    → Future Decision（quarantine 隔离执行 + 审计留痕 → 下次决策参考）
实验治理闭环（ADR 0033 + _feature_stage）：
  Feature Stage → ExperimentalWarning（实验 API 标记）→ feature usage bitmask 遥测上报
```

## Flow → KO 交叉校验
| Flow Edge | 对应 KO | 校验 |
|---|---|---|
| Approval 循环（审批状态机 + 逃生舱） | KO-01 | ✅ |
| 标签传播 → 策略执行 → quarantine | KO-02 | ✅ |
| 路径规范化 + 沙箱 + 白名单 | KO-03 | ✅ |
| 记忆注入（untrusted 通道） | KO-04 | ✅ |
| max_iterations 短路 + fresh_context + 停止谓词 | KO-05 | ✅ |
| Workflow DAG → 五编排模式 | KO-06 | ✅ |
| CodeAct 沙箱边界 | KO-07 | ✅ |
| Python/C# 同构 + 实验治理 | KO-08 | ✅ |

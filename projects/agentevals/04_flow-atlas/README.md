# 04 — Flow Atlas（七类流，全部从真实代码导出）

> 每条 Edge 标注可回溯位置（symbol / file / condition / state transition）。不凭架构想象画流程。

## 4.1 Control Flow（控制流）

```
CLI (cli.py) ──config──> eval_config_loader.py ──EvalParams──> runner.run_evaluation_from_traces()
  │                                                                   │
  └─traces 路径──> loader/auto.load_traces ──Trace[]──> convert_traces() (converter.py: get_extractor)
                                                                      │
                                          ┌───────────────────────────┘
                                          ▼
                          _evaluate_trace_bounded (runner.py:251)
                          ├─ trace_semaphore (runner.py:max_concurrent_traces)
                          └─ eval_semaphore (runner.py:max_concurrent_evals)
                                          │
                                          ▼
                    evaluate_builtin_metric (builtin_metrics.py:388)
                    ├─ build_eval_metric → get_evaluator（轻量直接 import，builtin_metrics.py:245）
                    └─ credential_ref → get_resolved_credential → _inject_judge_credential
                                          │
                                          ▼
                    evaluator.evaluate_invocations（async 或 asyncio.to_thread）
```
关键 Edge：
- C1 `cli → eval_config_loader → EvalParams.model_validate`（config.py:field_validator trajectory_match_type 非法值拒绝——**Control 面校验**）
- C2 `runner → _evaluate_trace_bounded`（双 semaphore 获取顺序：trace 槽先于 eval 槽——runner.py:262-266）
- C3 `get_evaluator` 直接 import `google.adk.evaluation.trajectory_evaluator`（builtin_metrics.py:247-254——**依赖裁剪分支**）

## 4.2 State Flow（状态流）

```
Run:  submit → QUEUED ──worker 领取──> RUNNING ──完成──> SUCCEEDED
        │                │                               ├─→ FAILED (error_text)
        │                └─cancel──> CANCELLED           └─（terminal 状态设 finished_at）
        └─已存在且 spec 相同 → 返回已有 run（幂等）
        └─已存在且 spec 不同 → RunSubmitConflict → HTTP 409（service.py:28-35）
Result: PASSED / FAILED / ERRORED / SKIPPED（models.py:26-31）
result_id = SHA-256(run_id|eval_set_item_id|evaluator_name)（models.py:33-36）
```
关键 Edge：
- S1 `RunRepository.claim_next`：只取 QUEUED 且 `attempt < max_attempts`，按 created_at 排序（oldest first）；**lease 租约**（timedelta）防多 worker 双重领取；heartbeat 续租——租约丢失/取消 → 返回 False（repos/__init__.py:50；memory.py:85-90；tests 锚定：test_claim_next_picks_oldest_queued / test_claim_respects_max_attempts）
- S2 `heartbeat`：未知 run → False；cancel 已请求 → False（test_heartbeat_returns_false_for_unknown_run / test_heartbeat_returns_false_when_cancel_requested）——**状态流转有测试锚定**
- S3 `cancel`：queued→cancelled（直接）；running→置 `cancel_requested=True`，worker 在**下次 heartbeat 观察**（cooperative cancellation，memory.py:104 `return not run.cancel_requested`）；terminal→返回 False（test_cancel_* 三测）

## 4.3 Data Flow（数据流）

```
OTel trace 文件/流
  → loader/auto.detect_format（.jsonl→OTLP；resourceSpans/batches→OTLP；data→Jaeger；失败→None）
  → JaegerJsonLoader / OtlpJsonLoader → Trace/Span（loader/base.py）
  → converter.get_extractor：ADK（gcp.vertex.agent scope）优先，否则 GenAI semconv
  → genai_converter / extraction：span → Invocation（user_content / final_response / intermediate_data.tool_uses/.tool_responses）
  → EvalSet（ADK schema：eval_set_id / eval_cases[conversation: Invocation[]]）
  → evaluate_builtin_metric：actual vs expected invocations → score
  → run/sinks：Stdout / File / HttpWebhook / Fanout
```
关键 Edge：
- D1 `_enrich_app_details`（builtin_metrics.py:120-156）：从 tool_names 合成 FunctionDeclaration → AgentDetails → AppDetails，**使无 schema 的 multi-turn 指标仍可评工具质量**——数据合成补全路径（有 bypass 味道，见 06 反例）
- D2 消息三通道归一：span attrs / log records（需双 exporter）/ span events（deprecated，streaming/processor 提升为 span 级）——docs/otel-compatibility.md
- D3 消息双 schema：content-based ↔ parts-based 按消息自动检测，tool_calls 归一 `{name,id,arguments}`（genai_converter.py:375）

## 4.4 Evidence Flow（证据流）★ 本系统核心

```
trace 落盘（一次性采集）──> loader ──> 归一 Invocation ──> evaluator（确定性/LLM judge）──> Result
    │                          │                              │                            │
    │ 采集面（代理运行时）      │ 证据面（不可变）             │ 评判面（可重复）           │ 记录面（幂等）
    │                          │                              │                            │
    └─ 遥测缺口：Claude Code/Codex/OpenCode 不发射 GenAI      └─ 无 golden → error          └─ result_id SHA-256 去重
       semconv（README 边界声明）                                （METRICS_NEEDING_EXPECTED）   （重试幂等）
```
关键 Edge：
- E1 证据不可变性：once trace captured，evaluate 任意次不重执行（EK-01）
- E2 证据保真边界：非 GenAI-semconv harness 需数千行胶水（EK-15）——**证据流的供给上限**
- E3 评判可重复：纯函数消费 + result_id 确定性（EK-08）——**重放幂等**

## 4.5 Authority Flow（权威流）

```
credential_refs（RunSpec 逻辑名）
  → get_resolved_credential（resolvers/__init__.py:162——逻辑名 → 实际 key）
  → _inject_judge_credential（builtin_metrics.py:307——ADK 私有 seam _judge_model_options/_judge_model）
  → judge model 以注入 key 构建（_build_judge_model，cached_property api_client 预置）
  ├─ credential_ref 未提供/未解析 → MetricResult(error)（fail-closed，builtin_metrics.py:395-404）
  ├─ evaluator 非 judge-backed → warning + skip（builtin_metrics.py:331-335）
  └─ 并发安全：get_evaluator 每次新实例，变异无共享状态
```
关键 Edge：
- A1 权威最小化：凭据以逻辑名引用，明文值只出现在运行期解析（config.py:30-33 注释）
- A2 fail-closed：未解析 → error 而非空 key 调用（test_credential_injection.py 覆盖）
- A3 技术债：依赖 ADK 私有属性 seam（TODO(upstream) 明示）——**权威通道的脆弱点**

## 4.6 Memory Flow（记忆流）

```
storage/：RunRepository（run 记录）/ ResultRepository（结果，result_id 去重）/ SessionRepository（trace 会话）
  ├─ memory 实现（测试默认）
  └─ postgres 实现（storage/postgres/migrations——持久化）
  ├─ run 历史 UI（docs/run-history.md）
  └─ 流式：StreamingTraceManager（streaming/，WebSocket 增量）
```
关键 Edge：
- M1 结果可重放：ResultRepository.upsert_many 幂等（ON CONFLICT DO UPDATE）——记忆不重复
- M2 会话↔trace 关联：SessionRepository.find_by_trace_id（test_find_by_trace_id）——trace 是会话记忆的索引键

## 4.7 Policy Flow（策略流）—— 治理闭环

```
Decision（README: no re-execution / ADK 复用 / golden gating）
  → Policy 固化：
     - 指标策略：METRICS_NEEDING_EXPECTED（无 golden → error，builtin_metrics.py:392）
     - 凭据策略：credential_refs fail-closed（config.py + builtin_metrics.py:395）
     - 并发策略：max_concurrent_traces / max_concurrent_evals（EvalParams）
     - 匹配策略：trajectory_match_type 合法值校验（config.py:14）
  → Enforcement：
     - CLI --match-type Choice 约束（cli.py:115）
     - config field_validator 拒绝非法值（config.py:_normalize_trajectory_match_type）
     - sink auth 从 env 提取（sinks.py:_extract_env_headers——不落配置文件）
  → Future Decision：README "Expect breaking changes" + CI/CD ready（门禁即未来部署决策）
```
关键 Edge：
- P1 策略显式化：匹配类型合法集合（EXACT/IN_ORDER/ANY_ORDER）在 config 与 CLI 双处校验——策略单一来源 + 接口层重复校验
- P2 凭据策略闭环：定义（credential_refs）→ 解析（get_resolved_credential）→ 失败行为（error）→ 测试（test_credential_injection）——**治理闭环完整**

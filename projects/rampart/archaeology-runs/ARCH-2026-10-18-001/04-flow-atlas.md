# 04 · Flow Atlas（七类流）— RAMPART refresh（ARCH-2026-10-18-001）

> 本文件为 refresh 增量流（trace/serialization/population 子系统的七类流）；09-04 基线七类流（corpus 04_flow-atlas.md）保持有效。关键 Edge 可回溯 symbol/file/condition。

## 1. Control Flow — 线性 trace 执行与终局评测
```
run_trace_async（core/trace.py:113）
  ├─ max_turns < 0 → ValueError（:128）
  ├─ loop turn_index in range(max_turns)（:133）
  │   ├─ driver.next_prompt_async(history=list(run.turns)) → decision
  │   │     └─ decision is None → trace_end_reason=DRIVER_EXHAUSTED → return（:136-139）
  │   ├─ session.send_async(decision.request) → response（:141）
  │   ├─ raw_turn 构造 → run.raw_turns.append（:143-151）
  │   ├─ stop_when is None → run.turns.append(raw_turn) → continue（:152-154）
  │   ├─ stop_when 存在 → _evaluation_context(raw_turns=run.raw_turns)（:156-159）
  │   │     ├─ stop_when.evaluate_async(context) → evaluation（:160-161）
  │   │     ├─ run.latest_stop_check = EvaluationRecord(...)（:162-166）
  │   │     ├─ run.turns.append(replace(raw_turn, eval_result, eval_purpose=STOP_CHECK))（:167-171）
  │   │     └─ evaluation.detected → trace_end_reason=STOP_CONDITION_MET → return（:172-174）
  │   └─ （loop 自然耗尽 → trace_end_reason=MAX_TURNS_REACHED）
evaluate_final_trace_async（core/trace.py:180）
  ├─ run.raw_turns 空 → return None（:195-196）
  ├─ latest_stop_check 且 evaluator 身份相同 且 _matches_final_trace_context → 复用（:197-208）
  │     └─ replace(record.result, evidence=safe_str_list, undetermined_operands=safe_str_list)（:205-208）
  └─ 否则 → 重新 _evaluation_context + evaluator.evaluate_async（:210-216）
```

## 2. State Flow — Result 终局状态字段与 trace 终止
```
状态入口: Result（core/result.py:140）
  ├─ status（权威，bool(result)=safe）
  ├─ final_trace_evaluation: EvalResult|None — 未迁移策略=None（:185）
  ├─ trace_end_reason: TraceEndReason|None — 终止原因（:187）
  │     └─ TraceEndReason 枚举（core/types.py:278）: DRIVER_EXHAUSTED / MAX_TURNS_REACHED / STOP_CONDITION_MET
  ├─ population: PopulationRef|None — trial 溯源（:191）
  └─ turn_evaluations 属性（:207）— 在线评估列表（排除终局）
状态转换: 迁移是渐进的——XPIA/SingleTurn 的 final_trace_evaluation 保持 None（docs/results-and-reporting.md:66-69）
```

## 3. Data Flow — 版本化序列化管道
```
serialize_record（core/serialization.py:131）
  ├─ _result_adapter().validate_python(record.result, strict=True)（:139）
  ├─ adapter.dump_python(mode="json", warnings="error")（:140）
  ├─ {"version": record.VERSION, "result": body} + pytest_nodeid? + result_index?（:145-149）
  └─ json.dumps(data, allow_nan=False)（:151）
deserialize_record（core/serialization.py:165）
  ├─ json.loads(parse_constant=_reject_json_constant)（:178）
  ├─ 非 Mapping → SchemaError（:181-183）
  ├─ version 查 _DECODERS（:182）→ 无 → UnsupportedSchemaVersionError（:183-185）
  └─ _decode_v2 → _validate_body（:233, :195）
      └─ json_value（JSON domain 递归校验）→ _result_adapter().validate_json(strict=True)（_schema.py:280, serialization.py:195-225）
```

## 4. Evidence Flow — 在线证据 vs 终局证据
```
执行期证据链:
  driver.next_prompt → session.send → raw_turn
    → stop_when.evaluate_async → EvaluationRecord（latest_stop_check）
    → turns 追加带 STOP_CHECK 标注的 turn（driver/report 视图）
终局证据链:
  evaluate_final_trace_async → final_trace_evaluation（输入 status，非重复 status）
  → Result 持久化（json_file sink / xdist merge）
隔离点: raw_turns 永不带标注（evaluator 消费）；turns 带标注（driver/report 消费）
不推断原则: codec 永不从 status/turns 推断终局评测或 stop reason（docs/concepts/trace-schema.md:62-65）
```

## 5. Authority Flow — xdist controller 权威归属
```
worker（untrusted payload）:
  serialize → {"schema": "rampart.xdist.v2", nodeid, results...}（_xdist.py SCHEMA_VERSION=58）
controller:
  deserialize_report_data → _validate_schema（schema 版本/枚举/深度校验）
    → envelope nodeid != TestReport nodeid → WorkerOutputError
    → 权威覆盖: metadata["_pytest_nodeid"]（来自 envelope nodeid）
              metadata["_rampart_result_index"]（来自 list position）
    → "cross-worker ordering is total and independent of any (untrusted) serialized values"（_xdist.py 头注释/deserialize_report_data）
```

## 6. Memory Flow — stop-check 记忆与复用
```
TraceRun.latest_stop_check（core/trace.py:59）: 最近一次 stop_when 检查（EvaluationRecord）
复用判定（evaluate_final_trace_async）:
  record.evaluator is evaluator（身份，eq=False 对象）
  _matches_final_trace_context（observability_level 相同 ∧ manifest 相同 ∧ len(raw_turns) 相同 ∧ 逐 turn 相等）
  → 命中则复用（避免重复 LLM 调用）；未命中则重评
```
> 记忆边界：raw_turns 快照在 EvalContext 中以 list(raw_turns) 浅拷贝（trace.py:_evaluation_context:81-88）——evaluator 拿到的是快照，不共享可变状态。

## 7. Policy Flow — trace 契约治理闭环
```
契约变更（任何 SOURCES 文件: result/types/serialization/_schema/_population）
  → CI 运行 check_trace_compatibility.py（scripts/check_trace_compatibility.py）
    → _fingerprint 计算 contract_sha256（_contract_files）
    → 与 schemas/trace-compatibility.json 比对
      ├─ previous is None → 必须 INITIAL 且无 previous_sha（_validate_decision:139-146）
      ├─ previous_sha 不匹配 → ValueError（:148-151）
      ├─ 同 major → 必须 COMPATIBLE（:152-157）
      └─ major+1 → 必须 NEW_MAJOR（:158-170）
    → 决策通过 → 版本演化合法；决策失败 → CI 红
未来决策依据: migration_note（如何迁移 v1→v2 的权威指引）
```
> Policy Flow 闭环完整：变更→指纹→决策→门禁→未来决策（v2 决策文件本身又是 v3 变更的 previous 锚点）。

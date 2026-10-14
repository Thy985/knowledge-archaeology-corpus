# 04 Flow Atlas · RAMPART（七类流，从真实代码导出）

关键 Edge 均可回溯 symbol/file/condition/state transition。

## 1. Control Flow（执行控制）
```
pytest collect
  → plugin.pytest_collection_modifyitems: trial 声明校验（无 trial_config fixture → UsageError）[plugin.py:208-228]
  → test body（@pytest.mark.harm/trial 标注）
      → Attacks.xpia()/Probes.single_turn() 返回 BaseExecution [_factory.py:99]
      → execute_async(adapter) [execution.py:227]
          → _execute_once_async: fire ON_PRE_EXECUTE → _execute_async（strategy）→ ON_POST_EXECUTE [execution.py:264-316]
              → XPIA._execute_async: _run_phases_async [xpia.py:100-160]
                  loop turn_index in range(max_turns)（XPIA/SingleTurn 各自内联实现，非 trace.py）:
                      driver.next_prompt_async(history) → None? break（driver exhausted）
                      session.send_async(request) → Response
                      evaluate_turn_async → Turn(eval_result)
                      eval_result.detected? break（early stop）
                      （trace.py run_trace_async 为公共导出 helper，单测覆盖，当前策略未接线）
              → 异常路径: except Exception → ON_ERROR → Result(ERROR) [execution.py:284-305]
  → Result 收集: ResultCollectionHandler（ON_POST_EXECUTE）→ ResultCollector [_collection.py:94]
  → plugin.pytest_runtest_logreport: xdist worker 流式传输 / controller 合并 [plugin.py:364-399]
  → plugin.pytest_sessionfinish: 聚合 → sinks 发射 → 强制 exit status [plugin.py:465-527]
```
关键分支：内置策略走内联 `turn.eval_result.detected → break`；trace.py helper 提供 stop_when.detected → STOP_CONDITION_MET（未接线）；driver.next_prompt None → break（DRIVER_EXHAUSTED 语义）。

## 2. State Flow（状态）
| 状态 | 载体 | 转换 |
|---|---|---|
| RampartSession | config.stash[_rampart_key]（pytest_configure 建 / unconfigure 删）[plugin.py:191-205] | absorb() 累积 results_by_nodeid；mark_incomplete/mark_emitted 标记；build_report() 缓存失效 [session.py:76-234] |
| driver-side conversation | PyRIT CentralMemory（LLMDriver._conversation_id）[drivers/llm.py:248] | 每 send 一用户轮；_assert_conversations_consistent 校验轮数一致 |
| Turn 不可变 | dataclass frozen [types.py:292] | provisional → replace(eval_result) → history |
| PayloadStore 磁盘 | .rampart/payloads/<collection>/{manifest.json,payloads.jsonl,artifacts/} | 原子写 temp-dir-then-rename [store.py:272-280] |
| _DefaultHandlerRegistry.factory | 模块级单例 [execution.py:149] | register/clear 在 pytest configure/unconfigure |

## 3. Data Flow（数据）
```
Payload(content/format/artifact) 
  → Surface.inject(payload) → InjectionHandle [injection.py:56-89]
  → XPIA: AsyncExitStack 进入 → wait_until_ready_async（索引延迟）
  → Request(prompt 或 attachments) [types.py:239-264]
  → session.send_async → Response(text/tool_calls/side_effects) [types.py:220-236]
  → evaluate_turn_async: Turn(provisional) → EvalContext(turns+observability+manifest) → EvalResult [execution.py:429-475]
  → Result(turns/injections/status) [result.py:139-231]
  → ResultCollectionHandler → RampartSession.absorb → TestRunReport → ReportSink.emit_async [sink.py:120-169]
xdist: Result → JSON-safe 序列化（_serialize_result）→ workeroutput → controller merge_report_results → 反序列化校验 [xdist.py:494-541]
```
数据保真约束：JSON-only 值 + 无 coercion + Payload ID 记录而非生成 [trace-schema.md]。

## 4. Evidence Flow（证据）
```
适配器观测（ObservabilityLevel 声明通道）[types.py:24-58]
  → Response.tool_calls/side_effects 由适配器填充（空列表=声明通道下确实未发生）
  → 评估器读取 EvalContext.all_tool_calls/all_side_effects（受 observability 门控）[types.py:431-441]
  → EvalResult.evidence（如 ToolCalled: evidence=[f"{name}({args})"]）[tool_called.py:96]
  → 判定: resolve_as_attack/probe（优先级 DETECTED>UNDETERMINED>NOT_DETECTED 等）[result.py:389-441]
  → Result.turns + final_trace_evaluation（终端评估）
  → 报告: terminal summary（ANSI 剥离）+ JsonFileReportSink + xdist 合并
```
关键不变量：观测通道缺失 → UNDETERMINED（非 NOT_DETECTED）[tool_called.py:96]；XPIA 观测降级（SAFE+RESPONSE_ONLY+零工具→UNDETERMINED）[xpia.py:247-284]。

## 5. Authority Flow（权威）
```
pytest（runner 权威）→ plugin（收集/配置/会话权威）→ BaseExecution（执行权威：固定骨架）[execution.py:187]
  → AgentAdapter 声明 manifest（agent 能做什么）+ observability_profile（能看什么）[adapter.py:37-90]
  → Evaluator 获得"读证据"的授权取决于 adapter 声明的观测通道
  → HarmCategory 门控: @pytest.mark.harm(*categories) → Result.harm_category → 报告分组 [result.py:52-80]
  → ReportSink 由 conftest hook 提供（controller 解析）[hooks.py:17-34]
  → 安全门: PopulationResult.status（ERROR 优先 → threshold → UNSAFE → UNDETERMINED）[result.py:285-293]
```

## 6. Memory Flow（记忆）
| 记忆类型 | 载体 | 生命周期 |
|---|---|---|
| 对话记忆（driver 侧） | PyRIT CentralMemory | LLMDriver 会话；一致性校验防跨测试复用 [llm.py:248-278] |
| payload 缓存 | PayloadStore 磁盘 | CI 生成一次缓存后续加载 [store.py:40-50] |
| run 聚合 | RampartSession | pytest configure→sessionfinish [session.py:76-234] |
| trace 历史 | Turn 列表（不可变） | 每执行一次；raw_turns 与 turns 双视图 [trace.py:45-67] |
| schema 策略 | _schema.py 递归重写 pydantic core schema（definition-ref 缓存） | 每次序列化 [schema.py:27-100] |

## 7. Policy Flow（策略/治理闭环）
```
Decision（设计意图：判定诚实 + 故障分离 + 信任边界）
  → Approval（docs/concepts/* 文档化策略：overview/pyrit/trace-schema）
  → Policy（代码固化）:
      · 观测声明 per-channel 保证 [types.py:33-36]
      · 判定优先级 [result.py:396-403, 422-428]
      · observability 降级 [xpia.py:262-266]
      · ERROR 从攻击率分母剔除 [sink.py:196-202]
      · incomplete 强制非零退出 [plugin.py:456-462]
      · 反序列化校验（schema/enum/depth/ANSI）[xdist.py:1-50]
      · payload 路径包含 [store.py:220-234]
  → Enforcement（执行）: 测试揭示（test_xpia.py/test_xdist.py/test_payload_store_security.py 固化策略）
  → Future Decision（迭代）: trace schema 版本化 + 兼容门禁 [check_trace_compatibility.py]
```

## Flow 交叉校验（Flow→KO）
- KO-01 ← Evidence Flow（观测通道→判定）✓ 每条 L1 事实回溯 Flow Edge（tool_called.py:96 → Edge E4）
- KO-02 ← Policy Flow（ERROR 语义）✓
- KO-03 ← Data Flow + Policy Flow（xdist 传输信任边界）✓
- KO-04 ← Evidence Flow（组合短路）✓
- KO-05 ← Control Flow（生命周期骨架）✓
- KO-06 ← Control Flow + State Flow（注入生命周期）✓
- KO-07 ← Data Flow（bridge 边界）✓
- KO-08 ← Evidence Flow（judge 契约）✓

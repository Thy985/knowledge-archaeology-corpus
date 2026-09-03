# 04 · Flow Atlas（七类流）

> 所有 Edge 从真实代码导出，可回溯到 symbol/file/condition/state transition。不按架构想象画流程。

---

## 1. Control Flow — 执行控制

```
pytest 收集测试
  → [plugin.py] pytest_collection_modifyitems（trylast，trial 克隆展开）
      → _create_trial_clones（复制 markers + xdist_group）
  → [plugin.py] autouse fixture _rampart_collect（activate ContextVar collector）
  → 测试体调用 Attacks.xpia(...).execute_async(adapter) | Probes.behavior(...)
      → [core/execution.py] BaseExecution.execute_async
          → _execute_once_async: fire ON_PRE_EXECUTE
              → _execute_async（子类实现）
                  → [attacks/_xpia.py] XPIA: _run_phases_async（注入→索引→会话→驱动循环）
                      → 循环内: driver.next_prompt_async → session.send_async → evaluate_turn_async → 早停
                  → [probes/_single_turn.py] SingleTurn: 会话→驱动循环→早停
              → 异常 → ON_ERROR → Result(ERROR)
          → fire ON_POST_EXECUTE（ResultCollectionHandler 写入 collector）
  → fixture teardown: deactivate_collector → absorb 进 RampartSession
  → [plugin.py] pytest_sessionfinish: 聚合 trial → evaluate gates → emit sinks
```
**关键 Edge**：
- `BaseExecution.execute_async` → `_execute_once_async` → `_execute_async`（Template Method，core/execution.py:162-174,180-222）
- XPIA 阶段顺序：`_activate_handles_async` → `stack.enter_async_context(session)` → 驱动循环（attacks/_xpia.py:85-106）
- 早停条件：`if turn.eval_result and turn.eval_result.detected: break`（attacks/_xpia.py:102, probes/_single_turn.py:84）
- trial 克隆：`pytest_collection_modifyitems` 中 `expanded.extend(clones)`（plugin.py:213-250）

## 2. State Flow — 判定状态转换

```
EvalOutcome(DETECTED|NOT_DETECTED|UNDETERMINED)        [core/types.py]
   │  resolve_as_attack: DETECTED→UNSAFE, NOTD→SAFE, 全UNDET→UNDETERMINED
   │  resolve_as_probe:  NOTD→UNSAFE, DETECTED→SAFE, 全UNDET→UNDETERMINED   [core/result.py:191-239]
   ▼
SafetyStatus(SAFE|UNSAFE|UNDETERMINED|ERROR)
   │  XPIA 特例: SAFE + RESPONSE_ONLY + 零工具 → UNDETERMINED               [attacks/_xpia.py:216-246]
   ▼
Result（唯一结果类型，bool==safe）                       [core/result.py:77-127]
   │  PopulationResult.status 聚合:
   │    any ERROR→ERROR; pass_rate≥threshold→SAFE; any UNSAFE→UNSAFE; else UNDETERMINED  [core/result.py:140-170]
   ▼
trial group verdict（Gate: ERROR→FAIL / ≥threshold→PASS / else FAIL）       [pytest_plugin/_session.py:109-152]
```
**关键状态转换**（可回溯）：
- `PopulationResult.status`：ERROR 优先 → threshold → UNSAFE → UNDETERMINED（result.py:153-167）

## 3. Data Flow — 数据流向

```
Payload(content,format,artifact)            [core/types.py]
  → Surface.inject(payload) → InjectionHandle（async ctx mgr）  [core/injection.py]
  → Request(prompt, attachments)           [core/types.py]
  → Session.send_async(request) → Response(text, tool_calls, side_effects)  [core/adapter.py]
  → evaluate_turn_async: provisional Turn → EvalContext(turns+provisional, observability_level, manifest)
      → evaluator.evaluate_async(context) → EvalResult(outcome,evidence,rationale,undetermined_operands)
      → replace(provisional, eval_result) 冻结 Turn          [core/execution.py:488-510]
  → turns[] → resolve → Result(turns, injections, metadata)
  → ResultCollectionHandler(ON_POST_EXECUTE) → ContextVar collector → RampartSession.results_by_nodeid
  → pytest_sessionfinish → build_report → ReportSink.emit_async（JsonFileReportSink）  [reporting/sink.py]
```
**关键 Edge**：
- `evaluate_turn_async`：provisional Turn（eval_result=None）→ `dataclasses.replace` 冻结（core/execution.py:496-510）
- Turn 不可变（`@dataclass(frozen=True)`，core/types.py:203-239）

## 4. Evidence Flow — 证据流

```
adapter.observability_profile（声明可观测通道）    [core/adapter.py]
  → EvalContext.observability_level（必填）       [core/types.py EvalContext]
  → evaluator 判断: "空列表 = 没发生"（通道有）vs "空列表 = 看不到"（通道无）
  → EvalResult.evidence + undetermined_operands（哪部分没确定）
  → resolver 语义: 只有 DETECTED 的 evidence 才进 UNSAFE summary（attack）  [core/result.py:317-333]
  → _summarize_undetermined_operands: 定论 + "但部分评估未确定: ..."  [core/result.py:241-280]
```
**关键 Edge**：
- UNSAFE summary 只取 DETECTED evaluator 的 evidence（`_build_summary` attack，attacks/_xpia.py:317-333）
- UNDETERMINED summary 用 `_explain_undetermined`（读 UNDETERMINED 结果的 rationale 或 operand reasons，result.py:343-400）

## 5. Authority Flow — 判定权威边界

> 本项目无真实权限/授权模型；"Authority"体现为**谁有权把什么状态判定为安全**。

```
判定权威分层：
  L1 evaluator（检测器）: 无权判定好坏——只报 DETECTED/NOT_DETECTED/UNDETERMINED  [core/evaluator.py]
  L2 resolver（语义层）: 有权把 DETECTED 映射为 UNSAFE(attack)/SAFE(probe)  [core/result.py]
  L3 observability 调整: 有权把 SAFE 降级 UNDETERMINED（当观测通道缺失）  [attacks/_xpia.py]
  L4 trial gate / incomplete 门: 有权把"未完整执行"判为 FAIL  [pytest_plugin/plugin.py]
  L5 消费者: 有权用 assert result 决定 CI 通过/失败  [docs/concepts/overview.md Result Contract]
```
**关键边界**：evaluator **不得**自行宣布好坏（protocol 设计 + resolver 独立）；LLM judge 的判定必须经 `_validate_outcome` 白名单校验（llm_judge.py:165-186）。

## 6. Memory Flow — 记忆/状态保留

```
短时（单测试）: turns[] 全对话（EvalContext.turns，含历史）   [core/types.py]
  → collector（ContextVar per-task，async 并发隔离）           [pytest_plugin/_collection.py]
中时（单会话）: RampartSession（config.stash）:
  trial_specs → trial_groups / results_by_nodeid / incomplete_reasons  [pytest_plugin/_session.py]
跨会话（磁盘）: PayloadStore（.rampart/payloads/ 原子写 + 路径防护）  [payloads/_store.py]
  → manifest.json（provenance）+ payloads.jsonl + artifacts/
跨 worker（xdist）: Result 序列化 → call report 传输 → controller merge（SCHEMA_VERSION=rampart.xdist.v2）  [pytest_plugin/_xdist.py]
```
**关键 Edge**：xdist 传输边界——setup/call 报告的 Result 快照被传输，teardown-only Results 留在传输边界外（plugin.py:333-355）。

## 7. Policy Flow — 策略/治理闭环

```
策略定义（pyproject.toml markers）:
  @pytest.mark.harm(categories)  → 分类策略
  @pytest.mark.trial(n=,threshold=) → 概率验证策略（阈值）
执行策略落地:
  trial gate: ERROR→FAIL / pass_rate≥threshold→PASS / else FAIL   [plugin.py _evaluate_gates]
  incomplete 策略: worker 丢失或序列化超限 → is_incomplete → sessionfinish 无条件强制非零退出（非 xdist 也生效；_absorb_results 异常仅 warning，C-07）  [plugin.py _enforce_incomplete_exit_status]
  size 策略: 序列化 >16MiB → 截断 + 标记 incomplete  [pytest_plugin/_xdist.py]
  observability 策略: 观测不足 → 降级 UNDETERMINED  [attacks/_xpia.py]
  judge 策略: LLM 输出非白名单 → 拒绝/UNDETERMINED  [evaluators/llm_judge.py]
反馈闭环:
  测试结果 → RampartSession 聚合 → sinks 报告（JsonFileReportSink）→ 消费方（CI dashboard）→ 未来测试调整
```

---

## Flow 交叉校验声明
- 每条 KO 的核心 flow edge 均可回溯（见 03 层 scope/evidence）
- 未发现"架构文档声称但代码不存在"的流程（docs/concepts/overview.md sequenceDiagram 与 execution.py 实现一致）
- 已知 deviation：plugin.py 用 `register_default_handler_factory/clear_default_handler_factory` 替代 architecture 声明的"直接写模块级 Callable"——已在 plugin.py 头注释 + EK-11 记录

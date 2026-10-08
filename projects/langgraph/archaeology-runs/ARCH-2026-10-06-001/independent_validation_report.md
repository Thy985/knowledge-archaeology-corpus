# Independent Validation Report — ARCH-2026-10-06-001 · langgraph

> 独立 Auditor 盲重建：不把考古包当事实来源，独立重读仓库建立 Independent Findings 后对比。禁止修改原考古产物。修正并入阶段⑥ Reconciliation。

## 0. 审计方法
- 独立重读面（与考古包错开路径）：`channels/base.py` 契约、`pregel/_runner.py`、`pregel/_retry.py:573-830`（run_with_retry/arun_with_retry）、`pregel/main.py:386-416`（stream_mode v3）、`graph/state.py:667-700`（add_node 实现体）、`types.py:833-870`（Command.resume 字段）、`tests/test_interruption.py`（resume id 映射/并行 interrupt）、`checkpoint/memory/__init__.py`（InMemorySaver）。
- 攻击面：单案例→Pattern、Pattern→L4、项目经验→通用 Principle、ADR→实现事实、Flow Edge 真实性、bypass/override/exception/alternate/direct call/admin path/fallback/legacy path、Epistemic 状态混淆。

## 1. 判定统计
| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 5 | F4 add_node 参数面 / F5 retry 退避公式 / F6 channel 契约 / T-01~17 全数（包内回读） / F1 部分（顺序匹配成立） |
| PARTIALLY_CONFIRMED | 1 | F1 resume 语义（顺序匹配成立但不完整，缺 id 映射） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 4 | F2 _runner 执行层 / F3 stream_mode v3 / F7 InMemorySaver 实例 / F8 func 模块 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 2. 3 成功（独立发现与包结论一致）
- **S1 add_node 参数面**：独立读 state.py:667 实现体，确认 retry_policy（序列取首个匹配）/cache_policy/error_handler/destinations（仅渲染）/timeout/trace_policy/defer/metadata/input_schema 与包内 EK-22 附件描述一致。→ CONFIRMED
- **S2 retry 退避公式**：独立读 _retry.py:667/823，`interval * backoff_factor ** (attempts - 1)`、max_attempts 判定（660/816）与 EK-26 一致。→ CONFIRMED
- **S3 通道抽象契约**：独立读 channels/base.py:49-112，checkpoint/from_checkpoint/get/is_available/update/consume/finish 七方法语义与包内 EK-01~07 依赖一致（consume 推进版本、finish 通知末超步）。→ CONFIRMED

## 3. 3 错误（包内不完整/需修正）
- **E1 [PARTIALLY_CONFIRMED] EK-21 resume 语义不完整**：包写"resume 值按调用顺序匹配"。独立证据：types.py:842-857 `Command.resume` 支持**两种形态**——"Mapping of interrupt ids to resume values"（推荐，按 interrupt id 精确恢复）与"单一值恢复下一个 interrupt"（顺序匹配）。tests/test_interruption.py 大量使用 `Command(resume={_interrupt_by_value(snapshot,"A1").id: ...})`，且覆盖"并行任务各自 interrupt、只恢复部分"场景。→ 修正：EK-21 补充 id 映射形态与并行任务隔离。
- **E2 [MISSING] 执行层 _runner 被弱化**：包 EK-10 将执行描述为 BackgroundExecutor（提交机制）。独立证据：pregel/_runner.py 是实际任务执行层（run_with_retry/arun_with_retry 在 _retry.py:573/685，被 _runner 调度；内含 backpressure、链路追踪、错误冒泡 GraphBubbleUp）。→ 修正：执行=BackgroundExecutor 提交（并发池）+ _runner 执行（重试/超时/背压/冒泡）两层。
- **E3 [MISSING] Evidence Flow 未覆盖 stream_mode v3**：包 04 章把 stream 描述为"分发 debug/values"。独立证据：main.py:386-416，v3 用 transformer mux 收集 stream_mode（`_collect_stream_modes`），且 `_V3_INVARIANT_KWARGS = ("stream_mode","subgraphs")` 禁止调用方覆盖。→ 修正：Evidence Flow 补 v3 transformer 管线。

## 4. 遗漏（Omissions，并入 Reconciliation）
- **F7**：InMemorySaver 实现存在（checkpoint/memory/__init__.py:33；get_tuple:230/put:421/put_writes:467）——EK-12 的实现实例，建议补充为 EK-12 的 evidence 锚点。
- **F8**：`langgraph/func/` 模块存在（函数式入口）——包 01 章已声明未深读，审计确认存在且不改变核心结论。
- **F9**：test_interruption.py 的"并行任务部分恢复"场景——建议在 03 KO-03 反例集补一条（resume 按 id 隔离解决并行 interrupt 顺序歧义）。

## 5. Benchmark case（对照 KO-04 版本不变量）
**盲重建**：独立重读 _algo.py:232-345，不参考包结论，回答"并发节点同时写一个通道时正确性靠什么保证"。
独立答案：①任务排序确定（path[:3]）②versions_seen 先于写入记账 ③通道版本由 get_next_version 单调推进 ④LastValue 多值抛错（fail-fast）⑤末超步 finish 通知。
**对比**：与 KO-04 三条不变量（单调版本/未记账不触发/确定写序）+ EK-04 fail-fast 完全一致。→ **CONFIRMED（Benchmark pass）**。附加观察：修复史 #9142/#9165/#9170 表明 fork/update_state 路径曾破坏不变量——支持包内"S5 证据强度"评级。

## 6. 重点攻击面检查
- **单案例→Pattern**：未发现（48 EK 均锚定多文件/多证据；KO 均有 ≥2 EK + 跨实例论证）。
- **Pattern→L4**：KO-01/03/04/05/06 均标 L4 并含反例约束；无越级。
- **项目经验→通用 Principle**：CP-01/CP-02 明确标"暂不升维"，未冒充 Principle。
- **ADR→实现事实**：commit HEAD（#9170 修复）与 _loop/_checkpoint 实际代码一致（fork 前重放逻辑存在）。
- **Flow Edge 真实性**：抽查 6 条关键 Edge（tick→after_tick / apply_writes→tasks / Send→TASKS / interrupt→Command / versions_seen 记账 / exit 判定），全部命中真实代码。
- **bypass/override/exception/alternate/direct call/admin path/fallback/legacy path**：
  - fallback：未知通道写 warning 忽略（_algo.py:310）——已覆盖
  - alternate：error-handler 节点替代主逻辑（state.py:1292）——已覆盖
  - legacy：create_react_agent deprecated（chat_agent_executor.py:278）——已覆盖
  - direct/exception：GraphBubbleUp 冒泡 + _runner 背压——**本次审计补充**（并入 E2）
- **Epistemic 状态混淆**：无（Candidates 全部标 Hypothesis + 缺失证据 + 验证路径；KO 层级与证据强度匹配）。

## 7. 审计结论
- 考古包总体真实（17 条 Truth 断言全 CONFIRMED；Benchmark case 通过）。
- 需修正 3 处（E1/E2/E3：resume 语义、执行层、stream v3），均为**补充性修正**，无 CONTRADICTED、无降级。
- 修正并入阶段⑥ Reconciliation（修改 Corpus Artifact，不修改生产 Skill）。

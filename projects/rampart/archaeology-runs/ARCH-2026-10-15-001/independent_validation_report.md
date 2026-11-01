# Independent Validation Report · RAMPART（盲重建审计）

- **run_id**: ARCH-2026-10-15-001
- **auditor**: 独立 Auditor（不把考古产物当事实来源；独立重读仓库建立 Independent Findings 后对比）
- **审计日期**: 2026-10-15
- **原始考古产物未被修改**（本报告独立存放；Reconciliation 修正见 06 §6.7 与阶段⑥ PR）

## 1. 审计方法
1. 独立重读仓库关键文件（不参考 02/03/04 结论）：result.py / execution.py / trace.py / _xpia.py / _single_turn.py / _collection.py / _trial.py / plugin.py / _xdist.py / tool_called.py / side_effect.py / llm_judge.py / sink.py / manifest.py / _facade.py / personas.py / templates.py / prompt_driver.py
2. 对考古的每条关键断言做符号级行号核验（grep 定位）
3. 攻击点检查：单案例→Pattern、Pattern→L4、项目经验→通用 Principle、ADR→实现事实、Flow Edge 真实性、bypass/override/exception/alternate/direct call/admin path/fallback/legacy path、Epistemic 状态混淆

## 2. 判定统计
| 判定 | 数量 | 说明 |
|---|---|---|
| CONFIRMED | 21 | 核心语义与考古一致（判定优先级/观测降级/信任边界/生命周期/协议反转/报告语义） |
| PARTIALLY_CONFIRMED | 7 | 机制正确但 scope/行号需收紧（见 §3 发现 1/2/3） |
| DOWNGRADED | 1 | EK-13/14/15 从"执行循环机制"降级为"公共 helper（未接入当前策略）" |
| OVER_GENERALIZED | 1 | 同 06（CI 门普遍性） |
| MISSING | 2 | ①SingleTurnExecution 独立内联循环未被 02 精确表述；②NEUTRAL_EVALUATOR 的 "ambiguous→NOT_DETECTED" persona 语义未记录 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 2 | C-07、C-09（同 05） |

## 3. 关键发现

### 发现 1（DOWNGRADED，最重要）：trace.py 是"公共 helper，未接入当前内置策略"
- **独立观察**：grep 全仓库 `run_trace_async|evaluate_final_trace_async` 只在 `core/__init__.py`（导出）与 `tests/unit/core/test_trace.py` 出现；`rampart/attacks/_xpia.py` 与 `rampart/probes/_single_turn.py` 均**各自内联**实现驱动循环（`for turn_index in range(max_turns): next_prompt → send → evaluate_turn_async → detected break`）
- **考古原表述**：02 EK-13/14/15 将 trace 双视图/在线停止/final-trace 复用写成"执行循环机制的一部分"
- **修正**：降级为"共享线性 trace 运行器 + final-trace 评估复用 helper（公共 API + 单元测试覆盖，future-ready），当前内置策略未调用；stop_when 在线停止是 trace.py 的能力而非当前内置策略的运行时路径"
- **影响**：04 Control Flow 的表述需同步修正（XPIA/SingleTurn 走内联循环 + resolve 语义；trace.py 是独立 helper 导出）

### 发现 2（PARTIALLY_CONFIRMED）：SingleTurn 与 XPIA 的循环实现同构但独立
- 独立观察：两个策略都内联循环 + `turn.eval_result.detected → break`；差异只在 resolve 语义（resolve_as_probe vs resolve_as_attack）与前置阶段（XPIA 有注入阶段，SingleTurn 无）
- 考古 02 EK-13 表述为统一机制 → 收紧为"两个策略各自实现驱动循环（相同模式：driver→send→evaluate→detected break），trace.py 未参与"

### 发现 3（PARTIALLY_CONFIRMED）：行号核验修正
| 考古引用 | 实测 | 修正 |
|---|---|---|
| sink.py:196-202（ERROR 剔除） | sink.py:136-138 注释 "503 is not a safety finding"、138-149 计算 | 已确认语义，修正行号 |
| plugin.py:456（TESTS_FAILED） | plugin.py:448-459（"green must mean ran and passed" 448；exitstatus=TESTS_FAILED 459） | 修正行号区间 |
| result.py:389-441（resolve） | result.py:389 resolve_as_attack 确认 | 无 |

### 发现 4（MISSING）：NEUTRAL_EVALUATOR persona 语义
- 独立观察：`personas.py` 内置 NEUTRAL_EVALUATOR 的 system_prompt 说 "When evidence is ambiguous, respond NOT_DETECTED"——这是**给 judge 的文本解读指引**（judge 对证据文本内容模糊时倾向 NOT_DETECTED）；与框架层"LLM 调用失败→UNDETERMINED"（llm_judge.py）是**两个不同层级的语义**，不冲突但需记录
- 修正：02 补充 EK（persona 层指引 vs 框架层降级）——并入 EK-18/19 边界说明

### 发现 5（CONFIRMED 强化）：unused-but-tested 是 RAMPART 的刻意模式
- 独立观察：`_facade.py`（Payloads.generate_async 独立于 PayloadStore）、`templates.py`（PromptTemplate.from_yaml + StrictUndefined + autoescape）、`converters/docx.py`（PyRIT WordDocConverter 封装）构成"已实现、测试覆盖、逐步接入"的发布节奏——与发现 1 的 trace.py 一致
- 结论：RAMPART 采用"先导出核心 helper 并测试，策略后续接入"的演进模式（fallback/legacy path 审计：无死代码删除痕迹，所有导出均被测试引用）

## 4. Benchmark case（盲重建对照基准）
| 维度 | 盲重建独立结论 | 考古结论 | 判定 |
|---|---|---|---|
| 判定优先级（attack/probe 不对称） | result.py:389-441 独立推导 | EK-04 | CONFIRMED |
| 观测降级（RESPONSE_ONLY+零工具→UNDETERMINED） | _xpia.py:247-284 独立推导 | EK-03 | CONFIRMED |
| worker payload 不受信（schema/enum/depth/ANSI） | _xdist.py:1-50 + test_xdist.py:655-742 | EK-24 | CONFIRMED |
| ERROR≠安全发现（分母剔除） | sink.py:136-149 独立推导 | EK-06 | CONFIRMED |
| trace.py 为未接线 helper | 全仓库 grep 独立发现 | EK-13/14/15 | DOWNGRADED |
| SingleTurn 独立内联循环 | _single_turn.py:76-112 独立读 | EK-13 scope | PARTIALLY_CONFIRMED |
| persona 层 ambiguous→NOT_DETECTED | personas.py 独立读 | （未记录） | MISSING |

## 5. 攻击路径检查（bypass/override/exception/alternate/direct call/admin path/fallback/legacy path）
| 路径类型 | 检查结果 |
|---|---|
| exception path | EK-06 确认：_execute_async 异常→ERROR Result（execution.py:284-305）；handler 异常 log+swallow（:346-355）✓ |
| fallback path | LLM judge 重试耗尽→UNDETERMINED（llm_judge.py:481-531）✓；duration sanitize 巨大值→0（test_xdist.py:742）✓ |
| override path | LLMJudge._send_async 为测试 override 点（llm_judge.py:570-595）；_HARDENING_SUFFIX 子类继承 ✓ |
| alternate path | 单进程 vs xdist 三模式（plugin.py:465-527）✓；inline XPIA（无 surface，attachments 携带）✓ |
| bypass 尝试 | 反序列化校验禁 bypass：未知/legacy schema version 拒绝（test_xdist.py:667-676）；eval_purpose 无 eval_result 拒绝（:719）✓ |
| admin/direct call | ResultCollectionHandler 是默认 handler 工厂（plugin.py:191-205）；record_result 手动记录（无 collector 时 no-op）✓ |

## 6. 遗漏项（阶段⑥ Reconciliation 必须并入）
1. 02 EK-13/14/15 降级（trace.py 未接线）→ 修正 02 表述 + 04 Control Flow 如实描述
2. 02 补 persona 语义（NEUTRAL_EVALUATOR ambiguous→NOT_DETECTED 为文本解读指引，框架层 UNDETERMINED 为调用失败降级）
3. 行号修正（sink.py:136-149；plugin.py:448-459）
4. 02 EK-34 的 XPIA 触发语义补充 SingleTurn（probe 触发=期望行为 prompt）对照

## 7. Epistemic 审计（独立视角）
- 考古 KO-01~08 全部标 Validated Pattern：盲重建确认均为"单项目内多机制印证"→ 标 Validated Pattern 恰当，未过度升维 ✓
- CK-03 标 cross-project pending ✓ 诚实
- C-01~C-09 保持 Hypothesis ✓
- **审计侧新增提醒**：NEUTRAL_EVALUATOR 的 "ambiguous→NOT_DETECTED" 指引若被用户误读为"框架默认把模糊当安全"，会导致假阴性理解——已在 05 Candidates 补记（C-10 待补，见 Reconciliation）

## 8. 审计结论
盲重建未发现考古结论的实质性错误（CONTRADICTED=0）；1 处层级需下调（trace.py 接线状态）、2 处遗漏需补充（SingleTurn 内联循环、persona 语义）、行号局部修正。考古核心三条主线（观测即契约/判定与故障分离/信任边界贯穿）全部独立成立。

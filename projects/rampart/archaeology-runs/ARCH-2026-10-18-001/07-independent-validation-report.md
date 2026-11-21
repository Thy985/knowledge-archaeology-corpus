# 07 · Independent Validation Report — ARCH-2026-10-18-001（RAMPART refresh）

> **Auditor 盲重建**：以下 Independent Findings 全部在**不读取主考古产物（00-06）**的前提下，对 HEAD `0fb9cc2` 独立重读仓库得出；写完 Findings 后才与主考古对比。主考古产物未被修改。

## 一、Independent Findings（盲建，独立来源）

| # | Finding | 证据（文件:行/符号） | 状态 |
|---|---|---|---|
| F1 | `ResultRecord` 信封 version const 校验；未知版本抛 `UnsupportedSchemaVersionError` | serialization.py:52, 65, 180-185 | Fact |
| F2 | `TraceRun.turns`（带标注视图）与 `raw_turns`（无标注视图）双列表；evaluator 只消费 raw | trace.py:51-59, 143-171 | Fact |
| F3 | `evaluate_final_trace_async` 复用 stop-check 的判据是 evaluator **身份**（eq=False）+ 上下文逐项匹配 | trace.py:28-38, 95-111, 197-208 | Fact |
| F4 | `PopulationRef` 校验 id 非空 / index<size / size≥1 / threshold finite∈[0,1]；`type() is not int` 拒绝 bool | result.py:103-131; _population.py:14-118 | Fact |
| F5 | `--rampart-trials` 只覆盖 n，threshold 始终来自 marker（无 CLI 覆盖阈值路径） | _trial.py:88-112 | Fact |
| F6 | **method marker 遮蔽（shadow）class marker，不合并**——`trial(n=2)` 在 `trial(n=5, threshold=0.9)` 类内时，生效的是 method 的 n=2 + 默认 threshold=1.0 | test_trial_integration.py:72-96; _trial.py:72（get_closest_marker） | Fact（主考古未覆盖） |
| F7 | **`eval_purpose requires eval_result` 不变量**：Turn.__post_init__ 拒绝"有 purpose 无 result" | types.py:318-325 | Fact（主考古未覆盖） |
| F8 | 信封 JSON Schema `additionalProperties: True`（开放），与 body 严格校验并存——strict 指字段值而非信封封闭性 | serialization.py:113, 285 | Observation（主考古未覆盖 nuance） |
| F9 | `result.py:264` 使用 `validate_population_parameters(allow_empty=True)`——存在允许空 population 的调用路径 | result.py:264 | Fact（主考古未覆盖） |
| F10 | `tests/unit/pytest_plugin/test_trial.py` + `test_trial_integration.py` 存在，专项覆盖 trial 解析/遮蔽/CLI override | 两文件全部 test_ 列表 | Fact（主考古 06 层遗漏列出） |
| F11 | XPIA/SingleTurn 均无 `run_trace_async` 引用 → final_trace_evaluation 实际保持 None（渐进迁移属实） | grep attacks/probes 全仓 | Fact |
| F12 | xdist SCHEMA_VERSION="rampart.xdist.v2" 与 trace "rampart.trace.v2" 是**两个独立版本空间** | _xdist.py:58 vs serialization.py:52 | Fact |
| F13 | compatibility gate 追踪 5 个 SOURCES 文件指纹；首次必须 INITIAL 且无 previous | check_trace_compatibility.py:139-146 | Fact |
| F14 | RAMPARTner skill 要求 build 前显式询问 source（main vs release），不推断 | skills/rampartner/SKILL.md:30-48 | Fact |

## 二、主考古 vs 独立验证对比（判定）

| 主考古声明 | Auditor 判定 | 说明 |
|---|---|---|
| EK-R01 双视图隔离 | **CONFIRMED**（F2） | 实现与语义一致 |
| EK-R02 证据分层/不推断 | **CONFIRMED**（F11 + docs） | codec 无推断路径 |
| EK-R03 版本化信封+decoder registry | **CONFIRMED**（F1） | 完全一致 |
| EK-R04 身份复用 | **CONFIRMED**（F3） | eq=False 身份边界属实 |
| EK-R05 渐进迁移（未迁移） | **CONFIRMED**（F11） | grep 证实 |
| EK-R06 population provenance | **CONFIRMED**（F4） | 校验集一致 |
| EK-R07 compatibility gate | **CONFIRMED**（F13） | INITIAL/COMPATIBLE/NEW_MAJOR 决策约束属实 |
| EK-R08 无 upcaster/禁 relabel | **CONFIRMED**（schemas/trace-compatibility.json migration_note） | 权威文件原文 |
| EK-R09 adapter-local schema | **CONFIRMED**（_schema.py:18-38 原文） | 声明即实现 |
| EK-R10 strict JSON-mode | **PARTIALLY_CONFIRMED** | strict 属实；但 Auditor 补充 nuance：**信封层 `additionalProperties: True`**（F8）——"严格"指字段值类型与 JSON domain，非信封封闭性。主考古未区分此边界 |
| EK-R11 xdist 权威归属 | **CONFIRMED**（_xdist.py deserialize_report_data 原文） | 权威覆盖属实 |
| EK-R12 duplicate timestamp 修复 | **CONFIRMED**（json_file.py:67-75） | uuid 后缀属实 |
| EK-R13 RAMPARTner 契约 | **CONFIRMED**（F14） | source choice 纪律属实 |
| EK-R14 fixture 消费强制 | **CONFIRMED**（plugin.py:212-230 + F10） | UsageError 属实 |
| EK-R15 CLI override 只覆盖 n | **CONFIRMED**（F5） | threshold 无 CLI 路径 |
| KO-R01~R05 抽象层级 | **CONFIRMED（L3/L4 判定合理）** | 均标 cross-project pending，无过度升维 |
| 06 层测试体系清单 | **MISSING** | 遗漏 `test_trial.py` / `test_trial_integration.py`（F10）——06 层列出 serialization/trace 测试但未列 trial 专项测试 |

## 三、3 个成功 / 3 个错误 / 遗漏

**3 个成功（Auditor 验证主考古正确且全面）**：
1. EK-R05 渐进迁移断言——盲 grep 证实 attacks/probes 无 run_trace_async 引用，且 docs 明示 None 语义，主考古未臆断"已迁移"
2. EK-R04 身份复用边界——eq=False + `_matches_final_trace_context` 逐项匹配，主考古准确抓住了"身份即复用边界"这个非显而易见设计
3. EK-R07 compatibility gate——主考古把"契约=显式决策对象"提升为 KO-R02（L4）并标 cross-project pending，层级判定与证据强度匹配，未过度升维

**3 个错误/不足（Auditor 纠正或补充）**：
1. **遗漏（MISSING）**：06_validation 测试清单漏列 `tests/unit/pytest_plugin/test_trial.py` 与 `test_trial_integration.py`（trial 解析/遮蔽/override 的专项覆盖）
2. **补充 nuance（PARTIALLY_CONFIRMED→需收紧 scope）**：EK-R10 "strict JSON-mode" 需补充信封 `additionalProperties: True` 边界——strict 作用于字段值/JSON domain，不封闭信封
3. **遗漏（MISSING）**：`Turn.eval_purpose requires eval_result` 不变量（F7）与 `validate_population_parameters(allow_empty=True)` 调用路径（F9）未被主考古 EK 捕获——两者都是类型纪律族（KO-R05）的额外实例

**其他 Auditor 独立发现（主考古未涉及但相关）**：
- F6 method marker 遮蔽语义（shadow 而非 merge）——trial 声明解析的边界行为
- F12 trace/xdist 双版本空间分轨——已进主考古 C-R06，Auditor 独立复现

## 四、Benchmark case（盲重建失败案例/方法论对照）
**盲重建成功点**：Auditor 独立列出的 14 条 Findings 中 12 条与主考古 EK 直接对应（CONFIRMED），说明主考古证据链扎实、无臆造。
**盲重建捕获点**：3 处遗漏/边界均来自"测试目录全列清单 + 代码边界搜索（additionalProperties / allow_empty / __post_init__）"这类主考古未做穷举的路径——验证了"Auditor 应独立列测试清单与边界参数"这一方法。
**方法论改进信号**：主考古 06 层测试清单应包含"pytest 插件专项测试"（test_trial/test_trial_integration），且 EK 边界描述应显式检查 schema 开放属性与 allow_* 参数——**该信号为本轮最接近 Skill Evolution 的候选，但属单案例，不构成系统性 skill 缺陷，不触发 P0/P1 proposal**。

## 五、Reconciliation（合并到 Corpus 的修正）
1. **06 层修正**：在 corpus 版本 06_validation 中补充 test_trial/test_trial_integration 覆盖记录（reconciliation 标注）
2. **EK-R10 scope 收紧**：补充"信封 additionalProperties: True（开放），strict 指字段值与 JSON domain 校验"边界说明
3. **EK 补充 2 条（R16/R17）**：
   - EK-R16 · `Turn.eval_purpose requires eval_result`：eval 标注一致性不变量（types.py:318-325）
   - EK-R17 · population 参数存在 allow_empty 调用路径（result.py:264, PopulationResult 聚合）——"size 可空"仅限聚合场景，execution 路径强制 ≥1
4. **Candidates 追加**：C-R07 · method marker 遮蔽（shadow）语义是否为 pytest get_closest_marker 通用行为（跨项目待验）

## 六、判定统计
| 判定 | 数量 |
|---|---|
| CONFIRMED | 13 |
| PARTIALLY_CONFIRMED | 1（EK-R10 scope 收紧） |
| DOWNGRADED | 0 |
| OVER_GENERALIZED | 0 |
| MISSING | 3（test_trial 清单 / eval_purpose 不变量 / allow_empty 路径） |
| CONTRADICTED | 0 |
| NEEDS_HUMAN_REVIEW | 0 |

## 七、Epistemic 状态复核
- 主考古无 Hypothesis 冒充 Fact：C-R01~R06 均标注 Hypothesis/Observation ✓
- 无 Principle 级断言：所有 L4 标 cross-project pending ✓
- KO-R02 的"兼容性=显式决策"为 L4 合理（test 锁定 + 跨领域可迁移），未过度 ✓

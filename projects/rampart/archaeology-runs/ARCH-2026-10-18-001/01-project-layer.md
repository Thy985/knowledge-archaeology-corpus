# 01 · Project Layer — RAMPART 项目地图（refresh @ 0fb9cc2）

> 基于 09-04 基线（corpus projects/rampart/01_project-layer.md）的 delta 更新。标注 **[新增]** / **[变化]** 的为 09-04 后引入；未标注的保持原状（原事实仍追溯 125595c）。

## 1. 仓库结构（delta）
```
rampart/
├── core/
│   ├── trace.py            [新增 #148] shared linear trace runner（TraceRun/EvaluationRecord/run_trace_async/evaluate_final_trace_async）
│   ├── serialization.py    [新增 #185] canonical versioned ResultRecord 序列化（TRACE_SCHEMA_VERSION="rampart.trace.v2"）
│   ├── _schema.py          [新增 #185] adapter-local trace schema 策略（pydantic schema 副本，不动 live classes）
│   ├── _population.py      [新增 #123] trial population 校验（id/size/threshold/index/parameters）
│   ├── result.py           [变化 #146] + PopulationRef + final_trace_evaluation + trace_end_reason；eval_results→turn_evaluations
│   ├── types.py            [变化 #148] + EvaluationPurpose + TraceEndReason + Turn.eval_purpose
│   └── ...（execution/evaluator/adapter/... 保持）
├── pytest_plugin/
│   ├── _trial.py           [新增 #123] TrialConfig / resolve_trial_config / parse_positive_int
│   ├── plugin.py           [变化 #123] 移除 _create_trial_clones；trial marker 校验 + trial_config fixture
│   ├── _xdist.py           [变化 #146] SCHEMA_VERSION="rampart.xdist.v2"；controller 权威 nodeid/index
│   └── ...（_session/_collection 保持）
├── skills/
│   └── rampartner/SKILL.md [新增 #195] RAMPARTner onboarding skill（Agent Skills 格式，随包分发）
├── reporting/json_file.py  [变化 #179] duplicate timestamp → _<uuid4>.hex 后缀
├── schemas/                [新增 #185] trace.v1.schema.json / trace.v2.schema.json / trace-compatibility.json
├── scripts/                [新增 #185] check_trace_compatibility.py / generate_trace_schema.py
├── docs/concepts/trace-schema.md  [新增 #185] 序列化与迁移策略权威文档
├── docs/getting-started/onboarding-skill.md [新增 #195] RAMPARTner 引导文档
└── docs/usage/results-and-reporting.md [变化 #146] final_trace_evaluation / turn_evaluations 语义
```

## 2. 入口
- pytest 插件入口不变：`[project.entry-points.pytest11] rampart = rampart.pytest_plugin.plugin`（pyproject.toml:50）
- **新增第二入口**：`rampart/skills/rampartner/SKILL.md`——由 coding assistant 按 Agent Skills 规范发现/加载，不是运行时入口
- 无独立 CLI；pytest markers：`harm(*categories)` / `trial(n=1, threshold=1.0)` / `slow`（pyproject markers 段）

## 3. 核心模块职责（delta 表）
| 模块 | 职责 | 关键符号 |
|---|---|---|
| core/trace.py [新增] | 共享线性对话执行 + 终局评测复用 | `run_trace_async` / `evaluate_final_trace_async` / `TraceRun` / `EvaluationRecord` / `_matches_final_trace_context` |
| core/serialization.py [新增] | 版本化 ResultRecord 信封序列化/反序列化 | `serialize_record` / `deserialize_record` / `ResultRecord` / `_DECODERS` / `UnsupportedSchemaVersionError` |
| core/_schema.py [新增] | 序列化时施加 canonical 策略的 schema 副本 | `trace_schema` / `json_value` / `_trace_payload` / `_validate_population` |
| core/_population.py [新增] | population 参数/溯源校验（类型纪律） | `validate_population_id/size/threshold/index/parameters` |
| pytest_plugin/_trial.py [新增] | trial 声明解析 + CLI override | `TrialConfig` / `resolve_trial_config` / `parse_positive_int` |
| core/result.py [变化] | 单结果 + population 溯源 + 终局评测证据 | `Result.final_trace_evaluation` / `Result.trace_end_reason` / `PopulationRef` / `turn_evaluations` |

## 4. 核心数据结构（delta）
- `ResultRecord{version: const "rampart.trace.v2", result, pytest_nodeid?, result_index?}`——版本化信封；version 校验、attribution 类型校验（serialization.py:57-113）
- `PopulationRef{id: 非空 str, index: 0<=index<size, size: ≥1, threshold: finite∈[0,1]}`——population provenance；`__post_init__` 交叉校验（result.py:103-131 + _population.py）
- `TraceRun{trace_end_reason, observability_level, manifest, turns, raw_turns, latest_stop_check}`——**turns = driver/report 视图（带在线 evidence）；raw_turns = evaluator 视图（无框架标注）**（trace.py:51-59）
- `EvaluationRecord{evaluator, context, result}`——frozen dataclass、eq=False；**evaluator 身份 = 复用边界**（trace.py:28-38）
- `Turn.eval_purpose: EvaluationPurpose | None`——None 表示未记录（含 predates-trace-runner 执行）（types.py:314）
- `Result.final_trace_evaluation: EvalResult | None` / `Result.trace_end_reason: TraceEndReason | None`——None = 未迁移策略/错误/manual 结果（result.py:185-187）

## 5. 状态
- 同 09-04：无全局可变状态；RampartSession（config.stash）+ ResultCollector（ContextVar）
- **新增状态**：trace schema 版本状态（`schemas/trace.v1/v2` + `trace-compatibility.json` 指纹链）；xdist SCHEMA_VERSION="rampart.xdist.v2"

## 6. 测试体系（delta）
- [新增] tests/unit/core/test_trace.py / test_serialization.py / test_serialization_properties.py / test_serialization_evaluations.py
- [新增] tests/scripts/test_trace_compatibility.py（compatibility gate：INITIAL/COMPATIBLE/NEW_MAJOR 决策约束、指纹链校验）
- [新增] tests/unit/pytest_plugin/test_xdist.py（+241）、test_xdist_aggregation.py（+94）
- 质量门不变：coverage fail_under=80、CI 4 版本矩阵（3.11-3.14）、ruff ALL

## 7. 主要配置（delta）
- `schemas/trace-compatibility.json`：version / contract_sha256 / previous_contract_sha256 / decision(initial|compatible|new-major) / rationale / migration_note——**trace 契约变更必须显式决策**
- `scripts/check_trace_compatibility.py`：CI 门禁，追踪 5 个 SOURCES 文件（result/types/serialization/_schema/_population）的契约指纹
- `--rampart-trials N`：CLI override `@pytest.mark.trial` 的 n
- pyproject markers：`trial(n=1, threshold=1.0)`（原 clone 语义废弃）

## 8. 权限 / policy / governance（delta）
- **新增 trace compatibility gate**：contract 变更必须有显式 decision（same-major→COMPATIBLE；major+1→NEW_MAJOR；首次→INITIAL 且无 previous 指纹）（check_trace_compatibility.py `_validate_decision`）
- **serialization 严格模式**：no coercion（strict=True）；`allow_nan=False`；json-only values 递归校验（循环/非有限/非字符串键拒绝）；null 仅 nullable 字段（_schema.py json_value）
- **xdist 权威归属**：controller 从 envelope nodeid + list position 设置 authoritative `_pytest_nodeid`/`_rampart_result_index`，不信任 worker 序列化值（_xdist.py deserialize_report_data）
- 09-04 安全边界全部保留（terminal sanitize / payload store path guard / judge outcome 三枚举 / 16MiB 截断 / ANSI 剥离）

## 9. 外部依赖
- pyrit==0.13.0 不变；pydantic>=2.13.4（序列化核心载体）/ jinja2 / pytest>=9.0.3 / pytest-asyncio / pyyaml
- onedrive 可选：msgraph-sdk>=1.57.0 + azure-identity>=1.25.3

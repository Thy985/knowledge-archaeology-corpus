# 01 Project Layer · RAMPART（项目地图，L0 事实）

## 1.1 定位与形态
- pytest-native 安全测试框架；MIT；Python >=3.11；Microsoft AI Red Team 官方（2026-05-20 开源）；PyPI 包名 `RAMPART`
- 安装后作为 pytest 插件激活（entry-point `pytest11: rampart = rampart.pytest_plugin.plugin`），测试写法即普通 pytest 测试 + `@pytest.mark.harm(...)` / `@pytest.mark.trial(...)`
- 仓库顶层：`rampart/`（包 10,501 行）、`tests/`（unit+integration+scripts）、`docs/`（mkdocs）、`schemas/`（trace.v1/v2 + compatibility）、`scripts/`（schema 生成/兼容门禁/版本 bump）、`tools/`（flake8 自定义规则）、`.github/workflows/` + `.pipelines/`

## 1.2 架构分层（官方四层）
| 层 | 内容 | 归属 |
|---|---|---|
| L1 | PyRIT（LLM 通信/提示词归一化/对话记忆/文档转换） | 依赖 `pyrit==0.13.0` |
| L2 | RAMPART（攻击/探测执行 + 评估器 + 报告 + pytest 插件） | 本包 |
| L3 | Consumer Package（用户实现 AgentAdapter 的薄集成层） | 用户代码 |
| L4 | Agent Under Test（被测 agent） | 用户系统 |

## 1.3 核心模块与职责
| 模块 | 文件（行数） | 职责 |
|---|---|---|
| pytest_plugin | _xdist.py 1651 / plugin.py 746 / _session.py 271 / _collection.py 134 / _trial.py 112 / _hooks.py 43 | pytest 集成、trial 解析、结果收集、xdist 序列化/聚合/信任边界、terminal summary、sinks hook |
| core | result.py 587 / execution.py 475 / types.py 468 / _schema.py 365 / evaluator.py 352 / serialization.py 324 / trace.py 230 / manifest.py 127 / _population.py 124 / adapter.py 90 / injection.py 89 / converter.py 74 / prompt_driver.py 66 / errors.py 57 / llm.py 35 | 类型系统、执行生命周期、判定语义、评估器协议、trace 运行器、schema 策略、序列化、manifest |
| evaluators | llm_judge.py 624 / response_contains.py 223 / tool_called.py 96 / side_effect.py 93 / personas.py | 内置评估器 + LLM judge + 组合 |
| attacks | _xpia.py 356 / _factory.py 99 | XPIA 攻击 + Attacks 工厂 |
| probes | _factory.py / _single_turn.py 76+ | SingleTurn（Behavioral）探测 |
| payloads | _store.py 453 / _facade.py 189 / _generator.py 137 / template.py 106 | payload 存储（原子写 + 路径安全）/生成/模板 |
| drivers | llm.py 369 / static.py 51 / _utils.py | StaticDriver（无状态）/ LLMDriver（PyRIT 对话记忆）/ coerce_driver |
| pyrit_bridge | llm_bridge.py 300 | PyRIT 类型翻译（create_prompt_target/send_*） |
| reporting | json_file.py 216 / sink.py 169 | TestRunReport / JsonFileReportSink |
| surfaces | onedrive.py 248 | OneDriveSurface（注入数据源） |
| converters | docx.py | DocxConverter（PyRIT WordDocConverter 封装） |
| common | templates.py / text.py | Jinja2 模板加载；safe_str/safe_float/strip_ansi 消毒 |
| skills | rampartner/SKILL.md | RAMPARTner onboarding skill（引导首个测试/集成/扩展判断） |

## 1.4 核心数据结构
- `ObservabilityLevel`（TOOL_AND_SIDE_EFFECTS / TOOL_ONLY / RESPONSE_ONLY；per-channel 保证）
- `Payload`（content 语义文本 + format + artifact；text↔binary 一致性校验）
- `ToolCall` / `SideEffect` / `Response`（text+tool_calls+side_effects+metadata）
- `Request`（prompt 或 attachments 至少其一）
- `Turn`（frozen 不可变；eval_result + eval_purpose 一致性校验）
- `EvalOutcome`（DETECTED/NOT_DETECTED/UNDETERMINED）+ `EvalResult`（outcome/confidence/evidence/rationale/undetermined_operands）
- `EvalContext`（turns + observability_level + manifest + metadata）
- `SafetyStatus`（SAFE/UNSAFE/UNDETERMINED/ERROR）+ `Result`（status/summary/turns/trace_end_reason/harm_category/strategy/injections/population）
- `PopulationRef`（id/index/size/threshold）+ `PopulationResult`
- `TraceRun`（turns 驱动视图 + raw_turns 评估视图 + latest_stop_check）
- `AppManifest`（name/tools[ToolDeclaration]/data_sources[DataSource writable_by_untrusted]）
- `InjectionRecord`（payload_id + surface_name）

## 1.5 生命周期
- **执行**：pytest 收集 → `Attacks.xpia()`/`Probes.single_turn()` 建 BaseExecution → `execute_async(adapter)` → 事件 ON_PRE_EXECUTE → `_execute_async`（strategy）→ per-turn（driver.next_prompt → session.send → evaluate_turn，early stop on detected）→ ON_POST_EXECUTE → Result
- **trial**：`@pytest.mark.trial(n, threshold)` → trial_config fixture → execute_trials_async（每 trial 新 execution factory，顺序执行）→ PopulationResult
- **会话**：pytest_configure 注册 handler factory + RampartSession → runtest 收集/吸收 → sessionfinish 聚合 → sinks 发射 → unconfigure 清理
- **xdist**：worker 在 call 阶段报告流式序列化 Result → controller 增量合并 → 单一报告

## 1.6 配置体系
- pyproject：markers（harm/trial/slow）；coverage fail_under=80；ruff select=ALL + RMP001/RMP002（tools/flake8_rampart.py）；pre-commit
- CLI：`--rampart-trials N`（覆盖 trial n）；`--rampart-xdist-max-bytes`（xdist Result 大小上限，默认 16MB）
- ini：`rampart_xdist_max_bytes`
- conftest hook：`pytest_rampart_sinks(config) -> list[ReportSink]`
- 环境：LLM 端点经 LLMConfig；onedrive 可选依赖（msgraph-sdk/azure-identity）

## 1.7 权限与治理
- CI：ci.yml / codeql.yml / coverage.yml / docs.yml / scorecard.yml / scripts.yml；.pipelines/integration-tests.yml
- trace schema 兼容门禁：scripts/check_trace_compatibility.py（trace-compatibility.json + trace.v1/v2.schema.json）
- 自定义 lint 规则 RMP001/RMP002（tools/flake8_rampart.py）
- OpenSSF Scorecard badge 在 README

## 1.8 测试体系
- unit：core（17 文件）、drivers、evaluators、payloads（含 payload store security）、pytest_plugin（plugin/trial/xdist/xdist_aggregation/collection）、reporting、surfaces、attacks/test_xpia.py、probes、pyrit_bridge、converters、common、public_api、tools/flake8
- integration：smoke + llm_judge（.env.local.example）
- scripts：hatch_build / trace_compatibility
- 测试揭示的关键行为见 02 EK（T 前缀条目）

## 1.9 外部依赖与边界
- 运行时：jinja2 / pydantic>=2.13.4 / pyrit==0.13.0（精确 pin）/ pytest>=9.0.3 / pytest-asyncio / pyyaml
- 可选：onedrive（msgraph-sdk + azure-identity）
- 边界：RAMPART 公共 API 不暴露 PyRIT 类型（唯一例外 LLMDriver.from_target 接受 PromptChatTarget）；execution strategy 只依赖 InjectionHandle 协议，从不 import Surface 实现
- 扩展机制：AgentAdapter/Session/Surface/InjectionHandle/PromptDriver/ReportSink 全协议可自定义；attacks/probes 工厂；evaluator 组合；RAMPARTner skill 指导"本地扩展 vs 框架 feature request"

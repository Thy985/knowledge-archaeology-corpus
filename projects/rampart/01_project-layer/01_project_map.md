# 01 · Project Layer — RAMPART 项目地图

> 所有事实可追溯到仓库实际内容（commit 125595c）。L0 Project Facts，非升维知识。

## 1. 仓库结构
```
rampart/                        # 主包（源树按 concern 组织）
├── core/                       # 基础类型与生命周期（共享词汇表）
│   ├── types.py                #   Payload/Request/Response/Turn/EvalContext/EvalResult/ObservabilityLevel
│   ├── execution.py            #   BaseExecution ABC + 事件生命周期 + trials
│   ├── result.py               #   Result/SafetyStatus/HarmCategory/PopulationResult + resolvers
│   ├── evaluator.py            #   Evaluator 协议 + BaseEvaluator + &/|/~ 组合
│   ├── adapter.py              #   Session / AgentAdapter 协议（消费者实现）
│   ├── injection.py            #   Surface / InjectionHandle 协议
│   ├── manifest.py             #   AppManifest / ToolDeclaration / DataSource
│   ├── persona.py              #   Persona（对抗角色）
│   ├── prompt_driver.py        #   PromptDriver 协议 + PromptDecision
│   ├── llm.py                  #   LLMConfig
│   ├── errors.py               #   DriverError / EvaluatorError / InfrastructureError
│   ├── converter.py            #   PayloadConverter 协议
│   └── __init__.py             #   公共 API 重导出（含懒加载）
├── attacks/_xpia.py            # XPIA（跨插件间接攻击）执行策略
├── probes/_single_turn.py      # SingleTurn（行为探针）执行策略
├── attacks/_factory.py         # Attacks 静态工厂（coerce_driver 等）
├── probes/_factory.py          # Probes 静态工厂
├── evaluators/                 # llm_judge / response_contains / side_effect / tool_called / personas
├── drivers/                    # llm.py（LLMDriver）/ static.py（StaticDriver）
├── payloads/                   # _generator / _store / _facade / template
├── pyrit_bridge/               # llm_bridge（PyRIT 通信隔离）
├── pytest_plugin/              # plugin.py / _session / _collection / _xdist
├── reporting/                  # sink.py / json_file.py
├── surfaces/onedrive.py        # OneDriveSurface（注入面）
├── converters/docx.py          # DocxConverter
└── common/                     # templates.py / text.py
tests/
├── unit/                       # 覆盖每个模块（含 test_public_api, test_payload_store_security）
├── integration/                # 真实验证（llm_judge / smoke）
└── scripts/                    # hatch_build
docs/                           # 概念/教程/贡献/API（mkdocs）
scripts/ tools/                 # hatch_build.py / flake8_rampart.py / bump_pyrit_version.sh
.github/workflows/              # ci / codeql / coverage / docs / scorecard / scripts
.pipelines/integration-tests.yml
```

## 2. 主要语言与运行入口
- **语言**：Python ≥3.11（单一语言）
- **运行入口**：pytest 插件（`[project.entry-points.pytest11] rampart = rampart.pytest_plugin.plugin`），pip 安装后自动注册
- **CLI**：无独立 CLI；通过 `pytest` + markers（`@pytest.mark.harm` / `@pytest.mark.trial`）驱动

## 3. 核心模块职责
| 模块 | 职责 | 关键符号 |
|---|---|---|
| core/execution.py | 执行生命周期骨架（Template Method） | `BaseExecution.execute_async` / `execute_trials_async` / `evaluate_turn_async` |
| core/evaluator.py | 评估组合（三值逻辑） | `_AnyEvaluator` / `_AllEvaluator` / `_NotEvaluator` / `_merge_undetermined` |
| core/result.py | 结果与语义解析 | `resolve_as_attack` / `resolve_as_probe` / `PopulationResult` |
| attacks/_xpia.py | 跨插件间接攻击全生命周期 | `XPIAExecution` / `_activate_handles_async` / `_adjust_for_observability` |
| probes/_single_turn.py | 行为探针 | `SingleTurnExecution` |
| pytest_plugin/plugin.py | pytest 集成（markers/收集/trial 克隆/sinks） | `pytest_collection_modifyitems` / `_create_trial_clones` / `_enforce_incomplete_exit_status` |
| pytest_plugin/_xdist.py | xdist 并行 + 信任边界 | `attach_report_results` / `merge_report_results` / `SCHEMA_VERSION` |
| evaluators/llm_judge.py | LLM 裁判（从 pyrit target） | `LLMJudge` / `_validate_outcome` / `TranscriptScope` |
| payloads/_store.py | 载荷持久化（原子写 + 路径防护） | `PayloadStore.save/load` / `_ensure_within_directory` |

## 4. 核心数据结构
- `Payload{content,id,format,artifact,metadata}` — 注入载荷（text/binary 分类）
- `Turn{request,response,eval_result,turn_number,timestamp,driver_reasoning}` — **不可变**对话轮
- `EvalResult{outcome,confidence,evidence,rationale,undetermined_operands}` — 条件检测信号（非安全判断）
- `EvalContext{turns,observability_level,manifest,metadata}` — 评估上下文
- `Result{status,summary,observability_level,turns,...,harm_category,strategy,injections,population}` — 唯一结果类型
- `PopulationResult{results,threshold}` — 统计试验聚合
- `SafetyStatus`(SAFE/UNSAFE/UNDETERMINED/ERROR) / `EvalOutcome`(DETECTED/NOT_DETECTED/UNDETERMINED) / `ObservabilityLevel`(TOOL_AND_SIDE_EFFECTS/TOOL_ONLY/RESPONSE_ONLY)

## 5. 核心状态
- **运行态**：无全局可变状态（除 `_default_handler_factory` 单例注册表，由 pytest configure/unconfigure 生命周期管理）
- **会话态**：`RampartSession`（config.stash）持有 sinks/trial_specs/trial_groups/results_by_nodeid/incomplete 标志
- **每测试态**：`ResultCollector`（ContextVar 作用域，async 并发下每 task 独立）

## 6. 测试体系
- **单元**：tests/unit/ 覆盖全部 11 个子包（attacks/core/drivers/evaluators/payloads/probes/pyrit_bridge/pytest_plugin/reporting/surfaces/tools/common）
- **安全专项**：tests/unit/payloads/test_payload_store_security.py（路径逃逸/symlink/非字符串）
- **公共契约**：tests/unit/test_public_api.py（懒加载不拉 pyrit）
- **集成**：tests/integration/（llm_judge 真实验证，需 .env）
- **质量门**：coverage fail_under=80；CI 4 版本矩阵（3.11-3.14）

## 7. 主要配置
- pyproject.toml：markers（harm/trial/slow）、asyncio_mode=auto、coverage 80、ruff ALL 规则、ty 类型检查
- .flake8 / tools/flake8_rampart.py：自定义 RAMPART 约定（RMP001/002）
- .pre-commit-config.yaml：ruff + ty + flake8-rampart
- CI：lint（ruff/ty/flake8）+ test（4 版本）+ codeql + coverage + scorecard + docs；`permissions: {}` 最小权限

## 8. 权限 / policy / governance 机制
- **框架自身安全边界**（对攻击者输入）：
  - 终端注入防护：`_sanitize_for_terminal` → `strip_ansi`（plugin.py）
  - xdist 信任边界：worker payload 视为可能被攻击者控制 → JSON-safe 序列化 + schema/enum/depth 校验 + ANSI 剥离（_xdist.py 头注释）
  - 序列化大小上限：`--rampart-xdist-max-bytes`（默认 16MiB，超限截断 + 标记 incomplete）
  - 载荷存储路径防护：`_validate_collection_name` / `_ensure_within_directory` / `_validate_artifact_reference`（payloads/_store.py）
  - LLM judge 输出校验：`_validate_outcome`（只接受三个枚举字面量）+ JSON 容错解析
- **测试语义治理**：trial gate（ERROR→FAIL；pass_rate≥threshold→PASS）；incomplete run 强制非零退出
- **"green 必须 ran and passed"**：`_enforce_incomplete_exit_status`（_xdist worker 丢失不能静默通过）

## 9. 主要外部依赖
- pyrit==0.13.0（git rev 6dc8b94）— 上游攻防原语（converters/prompt generation/judge target）
- pydantic / jinja2（模板）/ pytest / pytest-asyncio / pyyaml
- 可选：onedrive → msgraph-sdk + azure-identity；dev → flake8/pytest-xdist/ruff/ty

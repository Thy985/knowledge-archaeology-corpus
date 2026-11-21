# Snapshot Artifact · RAMPART（ARCH-2026-10-15-001）

## 快照事实表

| 字段 | 值 |
|---|---|
| repository | https://github.com/microsoft/RAMPART.git |
| commit SHA | `2c9067dff437deaa6562ec3979416ddda04f5a28` |
| branch | main（浅克隆 --depth 1） |
| commit message | "[FEAT]: Add RAMPART onboarding skill (#195)" |
| commit author | Nina Chikanov |
| commit timestamp | 2026-10-13 14:35:56 -0700 |
| version | hatch-vcs 动态版本（semver-pep440-release-branch，浅克隆无 tag；PyPI 可见） |
| license | MIT |
| language | Python ≥3.11（classifiers: 3.11-3.14） |
| 规模 | 132 个 .py 文件 / 30,891 行（rampart/ 包 10,501 行，不含 tests） |
| 创建时间 | GitHub: 2026-03-24（★421，Python） |

## 项目基础地图

**定位**：pytest-native 的 Agentic AI 安全测试框架（Risk Assessment & Measurement Platform for Agentic Red Teaming）——覆盖对抗攻击、良性失败、多类危害，评估驱动断言 + pytest 无缝集成。Microsoft AI Red Team 官方（2026-05-20 发布）。

**架构分层**（docs/concepts/overview.md 官方四层）：
- L1 PyRIT（pyrit==0.13.0 精确 pin，LLM 通信/提示词归一化/对话记忆/文档转换）
- L2 RAMPART（attacks/probes 执行策略 + evaluator + 报告 + pytest 插件）
- L3 Consumer Package（用户产品团队 repo 的薄集成层——AgentAdapter 实现）
- L4 Agent Under Test（被测 agent）

**核心模块**（rampart/ 10,501 行）：
| 模块 | 文件 | 职责 |
|---|---|---|
| pytest_plugin/ | _xdist.py 1651 / plugin.py 746 / _session.py 271 / _collection.py 134 / _trial.py 112 / _hooks.py 43 | pytest 集成：entry-point `rampart = rampart.pytest_plugin.plugin`；markers（harm/trial/slow）；trial 重复；xdist 分布式聚合；terminal summary |
| core/ | result.py 587 / execution.py 475 / types.py 468 / _schema.py 365 / evaluator.py 352 / serialization.py 324 / trace.py 230 / manifest.py 127 / adapter.py 90 / injection.py 89 / _population.py 124 / llm.py 35 / persona.py / prompt_driver.py 66 / converter.py 74 / errors.py 57 | Result/执行生命周期（BaseExecution 驱动 per-turn 循环：ON_PRE_EXECUTE→strategy→adapter→evaluator→early stopping）/ 类型 / schema / 评估器协议 / trace / 注入 surface 协议 |
| evaluators/ | llm_judge.py 624 / response_contains.py 223 / tool_called.py 96 / side_effect.py 93 / personas.py | 内置评估器（ToolCalled/ResponseContains/SideEffectOccurred/LLMJudge）；`\|` `&` `~` 组合算子 |
| attacks/ | _xpia.py 356 / _factory.py 99 | 内置攻击：XPIA（Cross-Prompt Injection Attack）+ 工厂 |
| probes/ | _factory.py / _single_turn.py | 内置探测：Behavioral（验证预期行为） |
| payloads/ | _store.py 453 / _facade.py 189 / _generator.py 137 / template.py 106 | payload 存储/生成/模板（harm 分类） |
| drivers/ | llm.py 369 / static.py 51 / _utils.py | PromptDriver：StaticDriver / LLMDriver（PyRIT 内部使用） |
| pyrit_bridge/ | llm_bridge.py 300 | PyRIT 类型翻译（公共 API 不暴露 PyRIT 类型；唯一例外 LLMDriver.from_target 接受 PyRIT PromptChatTarget） |
| reporting/ | json_file.py 216 / sink.py 169 | ReportSink / JsonFileReportSink |
| surfaces/ | onedrive.py 248 | 内置 OneDriveSurface（数据源注入面） |
| converters/ | docx.py | DocxConverter（PyRIT WordDocConverter 封装） |
| common/ | templates.py / text.py | 模板与文本工具 |
| skills/ | rampartner/SKILL.md | RAMPARTner onboarding skill（引导开发者建首个测试/集成/运行/扩展判断） |

**核心数据结构**：Result（safe/status/verdict）、Payload、Persona、LLMConfig、Surface/InjectionHandle 协议、Evaluator 组合表达式、trace（trace.v1/trace.v2 schema，版本化）。

**执行生命周期**（BaseExecution）：Test→execute_async(adapter)→fire ON_PRE_EXECUTE→strategy._execute_async→loop each turn (max_turns): driver.next_prompt_async(history)→session.send_async→Response→evaluator.evaluate_async→EvalResult→early stopping→AsyncExitStack 清理。

**Attack/Probe 语义**：Attack=测"不该有的坏行为"（detected=攻击成功→UNSAFE）；Probe=测"该有的好行为"（detected=预期行为存在→SAFE）。两者共用 Result 类型，差别在 evaluator 输出→安全判定映射。

**配置**：pyproject.toml（tool.pytest.ini_options markers: harm(*categories)/trial(n,threshold)/slow；coverage fail_under=80）；环境变量/配置文档 docs/usage/configuration.md；onedrive 可选依赖。

**权限与治理机制**：pre-commit（.pre-commit-config.yaml）；flake8 本地插件 tools/flake8_rampart.py（RMP001/RMP002 自定义规则）；ruff select=ALL + RUF 忽略项；CI：.github/workflows/{ci,codeql,coverage,docs,scorecard,scripts}.yml + .pipelines/integration-tests.yml；trace schema 兼容性门禁 scripts/check_trace_compatibility.py（schemas/trace-compatibility.json + trace.v1/v2.schema.json）。

**测试体系**（tests/）：unit 全覆盖（core 17 文件/drivers/evaluators/payloads/pytest_plugin(含 xdist 聚合、trial 集成、payload store security)/reporting/surfaces/attacks/test_xpia.py/probes/pyrit_bridge/converter/common/public_api）+ integration（smoke + llm_judge，.env.local.example）+ scripts（hatch build / trace compatibility）。pytest markers 区分 slow（spawn 子进程）。

**外部依赖**：jinja2, pydantic>=2.13.4, **pyrit==0.13.0（精确 pin）**, pytest>=9.0.3, pytest-asyncio, pyyaml, typing-extensions；可选 onedrive（msgraph-sdk, azure-identity）；dev: flake8/hatch/hypothesis/jsonschema/pre-commit/pytest-cov/pytest-xdist/ruff/ty。

**边界与扩展机制**：AgentAdapter/Session/Surface/InjectionHandle/PromptDriver/ReportSink 全协议可自定义；Evaluator 组合算子；attacks/probes 工厂扩展（RAMPARTner skill 指导"本地扩展 vs 框架 feature request"）；RAMPART 不暴露 PyRIT 类型（LLMDriver.from_target 例外）。

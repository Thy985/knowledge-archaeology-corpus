# 01 · Project Layer — PyRIT（ARCH-2026-10-03-001）

> 本层只记录可回溯到仓库实际内容的 L0/L1 事实。证据格式：`<文件>:<行号>` 或文件路径。

## 1. 项目定位与历史
- **定位**（README.md）："open source framework built to empower security professionals and engineers to proactively identify risks in generative AI systems"——生成式 AI 主动风险识别框架。
- **归属**：microsoft/PyRIT，MIT License（LICENSE），MSRC 漏洞策略（SECURITY.md）。
- **论文锚**：CITATION.cff 引用《PyRIT: Democratizing AI Red Teaming Through Open-Source Tooling》（2026-08 白皮书 PDF 链接于 README）。
- **版本轨道**：当前 `1.2.0.dev0`（pyproject.toml）——1.x 系列（GitHub releases：v1.1.0 / v1.0.1 / v1.0.0 / v0.14.0 / v0.13.0）。

## 2. 语言与规模
| 项 | 值 | 证据 |
|---|---|---|
| 主语言 | Python（1662 .py 文件） | `find . -name '*.py' | wc -l` |
| 包内行数 | 190,466 行 | `find pyrit -name '*.py' | xargs wc -l` 合计 |
| Python 版本 | `>=3.11, <3.15` | pyproject.toml `requires-python` |
| 包管理 | uv（uv.lock 1.26MB） | pyproject `[tool.uv]` |
| 前端 | TS/TSX（157+146 文件，frontend/） | 文件计数 |
| 测试 | 775 文件（unit 718 / integration 40 / end_to_end 5 / partner_integration 12） | tests/ 统计 |

## 3. 入口
- **CLI**（`pyrit/cli/`）：`pyrit_scan.py`（1273 行，场景扫描主入口）、`pyrit_shell.py`（1035 行，交互 shell）、`_server_launcher.py`（GUI 服务启动）、`api_client.py`、`_results.py`、`_config_reader.py`。
- **GUI**（frontend/ + gui-deploy.yml）：Web 前端对接 backend 服务。
- **库入口**：`pyrit/__init__.py` + 各子模块包。

## 4. 核心模块地图（行数可回溯）
```
pyrit/
├── executor/ (22,646)    攻击执行引擎
│   ├── attack/           攻击策略族
│   │   ├── single_turn/ multi_turn/ streaming/  攻击类型
│   │   ├── component/    adversarial_conversation_manager(972) prepended_conversation_config prepended_history_send_context modality_router conversation_manager
│   │   ├── compound/     sequential_attack(SequentialAttack/SequenceCompletionPolicy)
│   │   └── core/         attack_strategy(998) attack_executor attack_config attack_parameters attack_preparation attack_scoring attack_result_attribution
│   ├── promptgen/        gcg/attack/base/attack_manager(2539) fuzzer/fuzzer(1334)
│   ├── workflow/ benchmark/ core/(strategy.py config.py)
├── score/ (22,168)       评分体系
│   ├── message_scorer(1738) scorer(1284) scorable batch_scorer conversation_scorer llm_scoring
│   ├── fallback_scorer(15:class _FallbackScorer)  true_false/ float_scale/ _classifiers
│   ├── video_scorer audio_transcript_scorer text_chunking scorer_evaluation observation system_prompt
├── memory/ (19,911)      持久化
│   ├── memory_interface(9478) memory_models(2298) central_memory alembic/versions/
├── datasets/ (17,937)    攻击数据集（prompt/seed 集）
├── prompt_target/ (17,153) 目标抽象：openai/openai_response_target(989) common/discover_target_capabilities(1004) target_requirements
├── converter/ (16,460)   转换器（载荷变换）
├── scenario/ (15,529)    场景：core/scenario(1792) core/dataset_configuration(1018) scenarios/benchmark/adversarial(972)
├── backend/ (14,028)     服务：services/scenario_run_service(2556) scenario_progress_read_model(956)
├── models/ (13,950)      数据模型：score/score.py identifiers/ Message/AttackResult/SeedPrompt
├── cli/ (7,264)          pyrit_scan(1273) pyrit_shell(1035)
├── registry/ (4,432)     registry.py instance_registry.py resolution.py discovery.py tag_query
│   └── components/       converter_registry scorer_registry target_registry attack_technique_registry scenario_registry initializer_registry
├── auth/ (2,045) message_normalizer/ (1,400) prompt_normalizer/ (753) common/ (2,484) analytics/ (803) embedding/ (150)
└── exceptions/           exception_classes(PyritException 族) retry_collector exception_context
```

## 5. 核心抽象（L1 事实）
| 抽象 | 定义位置 | 关键点 |
|---|---|---|
| `Strategy` | executor/core/strategy.py:142 | ABC，强制生命周期：`_setup_async → _perform_async → _teardown_async`，`execute_with_context_async`(310) 编排；`StrategyEvent`(51)/`StrategyEventData`(76)/`StrategyEventHandler`(94) 事件化；`_StrategyRuntimeError`(33) |
| `AttackStrategy` | attack/core/attack_strategy.py | 泛型 `AttackStrategyContextT/ResultT`；`AttackParameters`；`attack_outcome_from_score`(score→outcome 契约) |
| `AttackExecutor` | attack/core/attack_executor.py:113 | 并发执行器：`max_concurrency`→asyncio.Semaphore(150)；`execute_attack_from_seed_groups_async`(169)；`_raise_first_fatal_exception`(474)；`return_partial_on_failure` |
| `Registry` | registry/registry.py | 通用注册表：discover→introspect→construct；单一路径 `_discover→register_class`；`resolve_constructor_args` 统一构造 |
| `InstanceRegistry` | registry/instance_registry.py:60 | 预构建组件对象注册表（Protocol）；`DefaultInstanceRegistry`(170) |
| `MemoryInterface` | memory/memory_interface.py:349 | ABC；`get_session_async`/`initialize_async`/keyset cursor（AttackResultKeysetCursor:158）；legacy override 兼容（`_uses_legacy_session_override`:432） |
| `Scenario` | scenario/core/scenario.py:113 | ABC，keyword-only 构造（121）；`BaselineAttackPolicy`(92)；atomic_attack_count/active_atomic_group_ids；`_apply_scorer_block_policy`(497)；`get_run_size_estimate_async`(631) |
| `Scorer` | score/scorer.py:1284 | 评分基类；`Scorable`；`ScoringExpectation`；`UndeterminedScoreError`（models/score/score.py:53） |
| `MessageScorer` | score/message_scorer.py:1738 | 响应评分主实现（objective + auxiliary） |
| `_FallbackScorer` | score/fallback_scorer.py:15 | primary→fallback 链，`_validate_comparable_results`(106) |
| `SequentialAttack` | attack/compound/sequential_attack.py:159 | 子攻击序列组合；`SequenceCompletionPolicy`(48)；`SequentialAttackResult`(114)（envelope 持有一等 AttackResult 行） |
| `ScenarioRunService` | backend/services/scenario_run_service.py:179 | 运行编排：`max_concurrent_runs`；`start_run_async`/`resume_run_async`(253)；`ScenarioRunConflictError`(133)；`_restore_launch_request`(311) |
| `AdversarialConversationManager` | attack/component/adversarial_conversation_manager.py:376 | 对抗对话管理（系统 prompt/首条/后续用户消息/schema 校验 `_parse_adversarial_reply`:304） |
| `PrependedHistorySendContext` | attack/component/prepended_history_send_context.py:16 | 发送上下文：seed_count/bootstrap_count/`begin_send`/`mark_target_invoked`/`finish_send`/`select_history`(125)/`remap_for_duplicate_conversation`(154) |

## 6. 核心数据结构（L1）
- `Score`（models/score/score.py:68）：pydantic BaseModel；`ScoreStatus`(35)；value 读取遇 undetermined 抛 `UndeterminedScoreError`(314-318)。
- `UnvalidatedScore`（models/score/score.py:346）：规范化前的原始分数。
- `AttackParameters` / `AttackSeedGroup`（attack/core/attack_parameters.py + executor 使用）：`params_type.from_seed_group()` 自动提取参数（attack_executor.py:238 docstring）。
- `AttackResult` + `AttackOutcome`（models/）：outcome 由 score 映射（attack_strategy.py `attack_outcome_from_score`）。
- `Identifiable` 标识体系（models/identifiers/）：`AttackIdentifier/ComponentIdentifier/TargetIdentifier/ScorerIdentifier/ConverterIdentifier`——memory 表镜像该层级（memory_models.py：ComponentIdentifierEntry:455 的"具体标识符表继承抽象表"模式）。
- Memory 表：`PromptMemoryEntries`（memory_models.py:264）、`TargetIdentifiers`(585)/`ConverterIdentifiers`(666)/`ScorerIdentifiers`(717) + Child 表（618/694/753）+ `TargetIdentifierChildren`(638)。

## 7. 状态与生命周期（L1）
- **攻击生命周期**：AttackExecutor 并发执行 → 每任务 `AttackContext`（带 attribution）→ AttackStrategy `execute_with_context_async`（setup/perform/teardown）→ 组件链（conversation manager + send context）→ PromptTarget → Message → score（objective+auxiliary）→ AttackResult 持久化（attribution_parent_id + objective_sha256）。
- **Scenario 运行生命周期**：ScenarioRunService 受理 `RunScenarioRequest`（start 或 resume，resume 凭 scenario_result_id:253）→ prepare（drain initialization tasks:581）→ 执行 → progress read model（956）→ 完成。并发闸门 `max_concurrent_runs`；`has_active_work`(230)。
- **SequentialAttack**：子攻击逐个执行（`_run_child_attack_async`:288），`_should_stop_after`(345) 按 policy 提前终止，`_compute_outcome`(355) 聚合 outcome。

## 8. 配置（L1）
- YAML 配置文件（.pyrit_conf_example）：`memory_db_type: sqlite`（in_memory/sqlite/azure_sql）；`Initializers` 列表（target/converter/scorer/technique/load_default_datasets/preload_scenario_metadata）；默认路径 `~/.pyrit/.pyrit_conf` 或 `--config-file`。
- 环境变量：.env_example（13KB）——API keys、memory 连接、backend 配置。
- pyproject：`[tool.pytest.ini_options]`、`[tool.ty.*]`（ty 类型检查）、pyrightconfig.json。

## 9. 权限与治理（L1）
- SECURITY.md：MSRC 私密报告渠道，禁止 public issue。
- policheck.yml + .azuredevops/：微软合规门禁。
- .pre-commit-config.yaml；CI：build_and_test/diff_cover（覆盖率门禁）/docker_build/docs/frontend_tests/triage-feedback。
- LICENSE：MIT；NOTICE.txt（1.2MB）/THIRD_PARTY_NOTICES.txt。

## 10. 外部依赖（L1）
- 运行时：SQLAlchemy（memory）、OpenAI SDK（openai_response_target）、pydantic（models BaseModel）、asyncio（并发）、NumPy/音频视频库（converter 信号处理，见 FIX commit "PCM silence"/"WAV samples"）、FastAPI（backend）。
- 开发：pytest、ty、pyright、pre-commit、diff-cover、uv。

## 11. 重要设计与 ADR 证据
- 仓库无 docs/adr 目录，但模块 docstring 承担 ADR 职能（设计意图显式声明）：
  - registry.py docstring：注册时机校验（"Validation therefore happens once, at registration time; there is no separate post-hoc sweep"）——设计决策。
  - attack_strategy.py：`attack_outcome_from_score` docstring（"stated once so no attack invents its own"）——统一契约决策。
  - sequential_attack.py docstring：子攻击持久化为一等 AttackResult 行（"persists as its own first-class AttackResult row"）——组合审计决策。
  - memory_models.py:105：EvaluationIdentifier 重算/重盖章说明——标识可追溯设计。
  - strategy.py docstring：生命周期强制（"enforced lifecycle management"）。

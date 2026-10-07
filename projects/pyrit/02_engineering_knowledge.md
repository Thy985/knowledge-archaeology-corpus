# 02 · Engineering Knowledge（EK Graph）— PyRIT（ARCH-2026-10-03-001）

> 宽底座：45 条 EK，每条含 `evidence`（文件:符号可回溯）与 `links`（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）。
> 铁律：孤立 EK 不删除——不能连接的标 `[D] 项目局部`，保留在底座。

## A. 组件化攻击架构（mechanism/subsystem 簇）

**EK-01 攻击 = 组件组合而非整体执行** [S4]
组件目录 `attack/component/`（adversarial_conversation_manager.py / prepended_history_send_context.py / conversation_manager.py / modality_router.py）与 `attack/compound/sequential_attack.py` 并存——攻击由 ConversationManager（对话状态）+ SendContext（发送语义）+ Converter（载荷变换）组合，SequentialAttack 组合子攻击。
evidence: `pyrit/executor/attack/component/` 目录结构；`sequential_attack.py:159 class SequentialAttack`
links: mechanism→EK-02, subsystem→EK-05

**EK-02 Strategy 强制生命周期（setup→perform→teardown）** [S4]
`Strategy.execute_with_context_async`（strategy.py:310）统一编排 `_setup_async`(214)/`_perform_async`(225)/`_teardown_async`(238)，子类只实现三段之一；`_StrategyRuntimeError`(33) 是执行失败专用类型。
evidence: `pyrit/executor/core/strategy.py:142-393`
links: mechanism→EK-01, subsystem→EK-03, constraint→EK-08

**EK-03 策略事件化（StrategyEvent handler 可插拔）** [S4]
`StrategyEvent`(51) 枚举 + `StrategyEventData`(76) 泛型 + `StrategyEventHandler`(94) ABC + `_register_event_handler`(190) + `_handle_event_async`(248)——策略运行过程对外发布事件，handler 可注入。
evidence: `pyrit/executor/core/strategy.py:51-280`
links: mechanism→EK-02, subsystem→EK-02

**EK-04 攻击类型三分：single/multi-turn/streaming** [S3]
`attack/` 下 single_turn/、multi_turn/（含 tree_of_attacks.py 2698 行 = TAP 攻击）、streaming/ 三个攻击类型目录并列。
evidence: `pyrit/executor/attack/{single_turn,multi_turn,streaming}/`
links: subsystem→EK-01, contrast→EK-06

**EK-05 PromptGen 与攻击策略分离（gcg/fuzzer）** [S3]
`executor/promptgen/gcg/attack/base/attack_manager.py`（2539 行，GCG 攻击管理器）与 `fuzzer/fuzzer.py`（1334 行）独立于 attack/ 策略族——载荷生成与执行编排解耦。
evidence: `pyrit/executor/promptgen/` 目录
links: subsystem→EK-04, contrast→EK-04

**EK-06 SequentialAttack：子攻击组合 + 一等结果行** [S4]
`SequentialAttack`(159) 是 `AttackStrategy[AttackContext[AttackParameters], SequentialAttackResult]`；`SequentialAttackResult`(114) envelope 持有一等 `AttackResult` 行（docstring："persists as its own first-class AttackResult row"）；`SequenceCompletionPolicy`(48) 决定提前终止（`_should_stop_after`:345）；`_compute_outcome`(355) 聚合子结果。
evidence: `pyrit/executor/attack/compound/sequential_attack.py:48-355`
links: mechanism→EK-01, causal→EK-07, contrast→EK-04

**EK-07 Attribution 可追溯链（attribution_parent_id + objective_sha256）** [S4]
AttackExecutor 把 `AttackResultAttribution`（attack_result_attribution.py:16）盖章到每个任务 AttackContext，持久化行带 `attribution_parent_id` + `attribution_data`；per-task 身份由 `objective_sha256` 重建（attack_executor.py:198-214 docstring）；shared attribution vs per-seed-group attributions 二选一（"Provide attribution or attributions, not both"）。
evidence: `pyrit/executor/attack/core/attack_executor.py:198-214`；`attack_result_attribution.py:16`
links: causal→EK-28, mechanism→EK-36

## B. Registry 与扩展机制（authority/subsystem 簇）

**EK-08 Registry 单一路径注册（注册时校验，无事后清扫）** [S4]
registry.py docstring："It owns a single add path: `_discover()` ... `register_class()` ... validates the class ... before it is stored. Validation therefore happens once, at registration time; there is no separate post-hoc sweep."
evidence: `pyrit/registry/registry.py` docstring（第 8-19 行）
links: mechanism→EK-09, constraint→EK-10, causal→EK-12

**EK-09 统一构造：resolve_constructor_args 处理嵌套 registry-reference** [S4]
`resolve_constructor_args`（registry/resolution.py）统一构造：简单值强制类型 + registry-reference 参数按名解析（docstring："registry-reference parameters ... are resolved by name — the same mechanism for every domain"）。
evidence: `pyrit/registry/registry.py` docstring；`resolution.py` 模块
links: mechanism→EK-08, dependency→EK-11

**EK-10 注册校验 = build contract 可推导 + reference 参数有 wired registry** [S4]
register_class 校验两条：build contract（构造函数）必须可推导（derive_parameters，resolution.py:311），每个 reference 参数必须映射到已接线的 registry。
evidence: `pyrit/registry/registry.py` docstring；`resolution.py:311 derive_parameters`
links: constraint→EK-08, constraint→EK-09

**EK-11 InstanceRegistry 分层：class catalog 与实例注册分离** [S4]
Registry 管类目录；`InstanceRegistry`（instance_registry.py:60 Protocol + :170 DefaultInstanceRegistry）只装预构建组件对象；"instance" 仅指 built component，绝不指 registry 单例（docstring）。
evidence: `pyrit/registry/instance_registry.py:60-170`；registry.py docstring
links: subsystem→EK-08, contrast→EK-08

**EK-12 领域注册表族：六个 components 注册表** [S3]
`registry/components/`：converter_registry / scorer_registry / target_registry / attack_technique_registry / scenario_registry / initializer_registry——每个领域一个注册表，共享 Registry 基类机制。
evidence: `pyrit/registry/components/` 目录
links: subsystem→EK-08, subsystem→EK-13

**EK-13 Initializer 机制：配置驱动的组件预注册** [S4]
`.pyrit_conf_example` Initializers 列表（target/converter/scorer/technique/load_default_datasets/preload_scenario_metadata）+ `registry/components/initializer_registry.py`——初始化时按配置注册默认组件。
evidence: `.pyrit_conf_example`（Initializers 段）；`registry/components/initializer_registry.py`
links: causal→EK-12, subsystem→EK-38

## C. 评分权威链（authority/evidence 簇）

**EK-14 Score 是攻击判定的唯一权威** [S4]
`attack_outcome_from_score(score) → AttackOutcome`（attack_strategy.py）——攻击结果判定只由 score 决定；"stated once so no attack invents its own"。
evidence: `pyrit/executor/attack/core/attack_strategy.py` attack_outcome_from_score
links: causal→EK-16, authority→EK-15

**EK-15 Undetermined 是一等状态：不弱化为失败** [S4]
`UndeterminedScoreError`（models/score/score.py:53，ValueError 子类）；Score.value 读取遇 undetermined 抛错（score.py:314-318）；outcome 映射：undetermined 既不 success 也不 failure（attack_strategy docstring）。
evidence: `pyrit/models/score/score.py:53,314-318`；`attack_strategy.py` docstring
links: contrast→EK-17, mechanism→EK-14

**EK-16 评分双轨：objective + auxiliary scorers** [S4]
`prepare_attack_scoring`（attack_scoring.py:33）组装 objective scorer + auxiliary scorers + expectations；`score_attack_response_async`(74) → `MessageScorer.score_response_async` 返回 `{objective_scores, auxiliary_scores}`。
evidence: `pyrit/executor/attack/core/attack_scoring.py:33-100`
links: causal→EK-14, subsystem→EK-19

**EK-17 FallbackScorer：primary→fallback 链 + comparable 三重校验** [S4]
`_FallbackScorer`(fallback_scorer.py:15)：`_get_child_scorers`(58) 组链；`_validate_comparable_results`(106) 校验 primary/fallback 分数**同证据（scorable/message_piece_id）+ 同类别（score_category）+ 同期望（scored_expectation）**，逐条 ValueError（Reconciliation E-01：由"同型"细化为三重校验）；`_fallback_rationale`(166) 记录回退理由。
evidence: `pyrit/score/fallback_scorer.py:15-166`（106-126 三条校验）
links: contrast→EK-15, mechanism→EK-16

**EK-18 Scorable 抽象：什么可以被评分** [S3]
`Scorable`（score/scorable.py）+ `message_scorable_resolver.py` + `text_chunking.py`（长文本分块评分）——评分对象抽象与评分执行分离。
evidence: `pyrit/score/scorable.py`；`text_chunking.py`
links: subsystem→EK-16, dependency→EK-19

**EK-19 Scorer 领域族：分类器/量表/多模态** [S3]
score/ 下 `_classifiers/`、`true_false/`、`float_scale/`、`video_scorer`、`audio_transcript_scorer`、`conversation_scorer`、`batch_scorer`、`llm_scoring`、`scorer_evaluation`、`observation`——评分器按答案类型与模态分族。
evidence: `pyrit/score/` 目录
links: subsystem→EK-16, mechanism→EK-17

**EK-20 未命中目标响应 = 错误回退路径（AdversarialChatResponseBlockedException）** [S4]
异常族（exception_classes.py:228 `AdversarialChatResponseBlockedException(BadRequestException)`）+ adversarial_conversation_manager 的 `_raise_for_adversarial_error`(204)/`_first_response_error`(162)——对抗对话中被目标拒绝/阻断走显式异常而非静默成功。
evidence: `pyrit/exceptions/exception_classes.py:228`；`adversarial_conversation_manager.py:204`
links: causal→EK-21, contrast→EK-15

## D. 并发执行与失败策略（control 簇）

**EK-21 并发执行：asyncio.Semaphore 限流（max_concurrency）** [S4]
`AttackExecutor.__init__`(122, `max_concurrency: int = 1`)；`_get_semaphore`(150) 建 asyncio.Semaphore；`build_params_async`/`run_one_async` 共享信号量（169-404）。
evidence: `pyrit/executor/attack/core/attack_executor.py:122-404`
links: mechanism→EK-22, dependency→EK-23

**EK-22 参数构建失败与执行失败分离治理（return_partial_on_failure）** [S4]
`execute_attack_from_seed_groups_async`(169)：参数构建（from_seed_group）失败时，`return_partial_on_failure=True` 返回部分结果 + failures；False（默认）则压制执行并抛首个异常（"a parameter-build failure suppresses execution and raises the first exception by input order"）。
evidence: `pyrit/executor/attack/core/attack_executor.py:169-240`
links: causal→EK-23, contrast→EK-21

**EK-23 结果聚合：AttackExecutorResult + raise_if_incomplete** [S4]
`AttackExecutorResult`(37)：iterable、`has_incomplete`(83)/`all_completed`(88)/`exceptions`(93)/`raise_if_incomplete`(97)/`get_results`(102)——部分完成显式暴露，非静默丢弃。
evidence: `pyrit/executor/attack/core/attack_executor.py:37-102`
links: mechanism→EK-22, dependency→EK-24

**EK-24 致命异常策略：_raise_first_fatal_exception（按输入顺序）** [S4]
`_raise_first_fatal_exception`(474)——多个任务异常时按输入顺序取首个致命异常抛出，保证可复现。
evidence: `pyrit/executor/attack/core/attack_executor.py:474`
links: causal→EK-22, constraint→EK-23

**EK-25 动态重试：环境变量驱动的 stop/wait 策略** [S4]
`_DynamicStopAfterAttempt`(exception_classes.py:74) 每次重试检查读环境变量；`_DynamicWaitRandomExponential`(90) 每次等待计算读环境变量——重试预算可运行时调参，无需改代码。
evidence: `pyrit/exceptions/exception_classes.py:74-95`
links: mechanism→EK-26, constraint→EK-20

**EK-26 RetryCollector：重试事件集中收集 + ContextVar 跨任务可见** [S4]
`exceptions/retry_collector.py` + `common/logger`——重试事件进入全局收集器（`get_retry_collector()`，attack_strategy.py 导入）；**关键实现**（Reconciliation E-02）：collector 在 `execute_with_context_async` 父任务通过 `set_retry_collector` 注入 ContextVar（strategy.py:335-343），事件 handler 在子任务（asyncio.create_task）继承 ContextVar 副本——因此执行路径与事件 handler 两路都可见，结束 `finally` 清空。
evidence: `pyrit/exceptions/retry_collector.py`；`attack_strategy.py` 导入；`strategy.py:335-343`
links: mechanism→EK-25, subsystem→EK-31, causal→EK-03

## E. Memory 与持久化（memory/subsystem 簇）

**EK-27 CentralMemory：统一持久化入口** [S3]
`pyrit/memory/central_memory.py` + `MemoryInterface`(memory_interface.py:349)——所有攻击/评分/会话数据经此读写；`memory_db_type` 配置三选（in_memory/sqlite/azure_sql）。
evidence: `.pyrit_conf_example`；`memory_interface.py:349`
links: mechanism→EK-28, subsystem→EK-29

**EK-28 AttackResult 持久化行 + keyset 游标分页** [S4]
`AttackResultKeysetCursor`(memory_interface.py:158, from_attack_result:174) + `_attack_results_keyset_seek_condition`(794) + `_attack_results_recency_order_by`(778)——大结果集 keyset 分页（非 offset）。
evidence: `pyrit/memory/memory_interface.py:158-187,778-794`
links: mechanism→EK-07, subsystem→EK-27

**EK-29 标识符表层级镜像（DomainBackedEntry 继承模式）** [S4]
`DomainBackedEntry`(memory_models.py:390, Generic[TDomain]) + `ComponentIdentifierEntry`(455, "concrete identifier tables inherit the abstract" docstring)——`TargetIdentifiers`(585)/`ConverterIdentifiers`(666)/`ScorerIdentifiers`(717) 继承抽象标识表，子表（618/694/753）存扩展字段。
evidence: `pyrit/memory/memory_models.py:390-753`
links: mechanism→EK-30, subsystem→EK-27

**EK-30 标识 = 哈希主键 + 可追溯重构** [S4]
标识表主键为 hash（如 `TargetIdentifierEntry.hash`），`PromptMemoryEntry.id` + 外键链（647/702/708）——行间关系靠标识哈希而非自增 id 关联，支持跨环境复现查询。
evidence: `pyrit/memory/memory_models.py:585-753`
links: mechanism→EK-29, causal→EK-07

**EK-31 异步引擎与会话生命周期管理（loop 资源检查）** [S4]
`_create_async_engine`(399)/`get_session_async`(416)/`_run_database_operation_async`(468)/`dispose_engine_async`(563)/`_check_loop_resources`(584)/`_discard_closed_loop_resources`(577)——AsyncEngine 与会话绑定 event loop，跨 loop 复用显式检查。
evidence: `pyrit/memory/memory_interface.py:399-584`
links: mechanism→EK-27, dependency→EK-32

**EK-32 legacy override 兼容层（_uses_legacy_*）** [S4]
`_uses_legacy_session_override`(432)/`_uses_legacy_memory_override`(522)/`_dispatch_memory_operation`(532)——新旧接口并存，老调用路径经 dispatch 兼容，不破坏历史 API。
evidence: `pyrit/memory/memory_interface.py:432-532`
links: contrast→EK-31, constraint→EK-27

## F. Scenario 编排（control/policy 簇）

**EK-33 Scenario = 多 AtomicAttack 分组编排** [S4]
`Scenario`(scenario.py:113, ABC)：`atomic_attack_count`(275)/`active_atomic_group_ids`(295)/`set_params_from_args`(518)/`get_run_size_estimate_async`(631)——场景把多个原子攻击组织为组，支持子集激活。
evidence: `pyrit/scenario/core/scenario.py:113-685`
links: mechanism→EK-34, subsystem→EK-01

**EK-34 BaselineAttackPolicy：场景级攻击基线策略** [S3]
`BaselineAttackPolicy(Enum)`(scenario.py:92)——场景声明基线攻击策略（供 benchmark 场景使用）。
evidence: `pyrit/scenario/core/scenario.py:92`
links: constraint→EK-33, policy→EK-41

**EK-35 ScorerBlockPolicy：评分器阻断策略** [S4]
`_apply_scorer_block_policy`(scenario.py:497)——场景可对 objective scorer 施加阻断策略（如禁用某类 scorer）。
evidence: `pyrit/scenario/core/scenario.py:497`
links: mechanism→EK-34, causal→EK-16

**EK-36 运行规模预估（RunSizeEstimate + BoundedDatasetSize）** [S3]
`get_default_run_size_estimate_async`(615)/`get_run_size_estimate_async`(631)/`_estimate_run_size_async`(685, budget: BoundedDatasetSize)——执行前预估成本/规模。
evidence: `pyrit/scenario/core/scenario.py:615-685`
links: constraint→EK-33, policy→EK-37

**EK-37 ScenarioRunService：并发闸门 + resume 恢复** [S4]
`ScenarioRunService`(scenario_run_service.py:179)：`max_concurrent_runs`(195)；`start_run_async`(242)/`resume_run_async`(253, 凭 scenario_result_id)；`_validate_resume_admission_async`(282)；`_restore_launch_request`(311, 从 ScenarioResult 恢复)；`ScenarioRunConflictError`(133)——运行可中断可恢复，冲突显式报错。
evidence: `pyrit/backend/services/scenario_run_service.py:133-311`
links: mechanism→EK-36, dependency→EK-31

**EK-38 Scenario 配置解析：DatasetConfiguration + 技术解析** [S3]
`core/dataset_configuration.py`(1018) + `_resolve_scenario_techniques`(596)/`_resolve_objective_target`(554)——场景数据集与攻击技术配置化解析。
evidence: `pyrit/scenario/core/dataset_configuration.py:1018`；`scenario.py:596`
links: subsystem→EK-33, causal→EK-13

## G. 目标与转换（data 簇）

**EK-39 PromptTarget 能力发现（discover_target_capabilities）** [S4]
`prompt_target/common/discover_target_capabilities.py`(1004) + `CapabilityName`——目标能力自描述发现，驱动路由（modality_router）。
evidence: `pyrit/prompt_target/common/discover_target_capabilities.py`；`attack/component/modality_router.py`
links: mechanism→EK-40, causal→EK-41

**EK-40 TargetRequirements：目标前置条件契约** [S3]
`prompt_target/common/target_requirements.py`（TargetRequirements）——目标对输入的要求（能力/模态）作为发送前置校验。
evidence: `pyrit/prompt_target/common/target_requirements.py`
links: constraint→EK-39, dependency→EK-02

**EK-41 ModalityRouter：多模态路由** [S3]
`attack/component/modality_router.py`——按目标能力把消息路由到合适模态路径。
evidence: `pyrit/executor/attack/component/modality_router.py`
links: mechanism→EK-39, subsystem→EK-01

**EK-42 PrependedHistorySendContext：发送语义状态机** [S4]
`PrependedHistorySendContext`(16)：seed_message_count(59)/bootstrap_message_count(64)/should_include_seed(69)/is_seed_consumed(74)/target_invocation_count(79)/begin_send(83)/mark_target_invoked(98)/finish_send(112)/select_history(125)/remap_for_duplicate_conversation(154)——发送前历史选择与重复会话重映射的显式状态机。
evidence: `pyrit/executor/attack/component/prepended_history_send_context.py:16-154`
links: mechanism→EK-02, subsystem→EK-01

**EK-43 AdversarialReply 解析 + JSON schema 校验** [S4]
`_parse_adversarial_reply`(adversarial_conversation_manager.py:304) 按 `response_json_schema` 解析对抗回复；`_build_adversarial_prompt_metadata`(285) 带 schema 元数据——对抗回复结构化契约化。
evidence: `pyrit/executor/attack/component/adversarial_conversation_manager.py:285-304`
links: mechanism→EK-42, constraint→EK-44

## H. 数据模型与错误（data/error 簇）

**EK-44 pydantic 数据模型族（models/）** [S3]
`models/score/score.py:68 Score(BaseModel)`、`UnvalidatedScore`(346)、`Message`/`AttackResult`/`SeedPrompt`/identifiers 族——结构化数据全用 pydantic 校验。
evidence: `pyrit/models/score/score.py:68,346`；`pyrit/models/`
links: subsystem→EK-45, dependency→EK-10

**EK-45 异常族分级（PyritException → BadRequest/RateLimit/ServerError）** [S4]
`PyritException(Exception, ABC)`(exception_classes.py:117) + `BadRequestException`(147)/`RateLimitException`(162)/`ServerErrorException`(177)/`EmptyResponseException`(213, BadRequest 子类)——服务端错误按可重试性分类（RateLimit 可重试，5xx 透明错误 ServerError）。
evidence: `pyrit/exceptions/exception_classes.py:117-228`
links: mechanism→EK-25, causal→EK-20

---
## Reconciliation 新增 EK（Independent Audit 盲重建补充，Reconciliation 合并）

**EK-46 异常执行上下文诊断链（component_role + 根因链 + 读后清空）** [S4]
`execute_with_context_async` 错误路径（strategy.py:355-378）：ON_ERROR 事件 → `get_exception_execution_context(e) or get_execution_context()` 读取执行上下文（含 `component_role.value`）→ 沿 `__cause__` 遍历取根因 → 组装错误消息 → **读后 `clear_execution_context()`**；无上下文时降级为类名错误消息 → 包 `_StrategyRuntimeError`。
evidence: `pyrit/executor/core/strategy.py:355-378`
links: causal→EK-02, mechanism→EK-25, dependency→EK-03

**EK-47 ObjectiveTargetConversationLifecycle：目标会话跟踪/释放生命周期** [S4]
`_ObjectiveTargetConversationLifecycle`（attack_strategy.py:95-130）：`__aenter__` 开始跟踪目标调用（`_conversation_ids` 集合），`__aexit__` 遍历释放每个本次攻击发起的会话，pending cancellation 显式处理（`asyncio.CancelledError`）——目标侧会话资源与攻击执行生命周期绑定。
evidence: `pyrit/executor/attack/core/attack_strategy.py:95-130`
links: mechanism→EK-02, subsystem→EK-42, causal→EK-06

---
## 游离 EK（无法连边，标 D 项目局部，保留）
- **[D] EK-A1** `analytics/`（803 行）分析模块（AnalyticsException/AnalyticsBusyException/AnalyticsTimeoutException——analytics_exception.py:7-27）——当前考古未见调用点，保留为项目局部事实。
- **[D] EK-A2** `embedding/`（150 行）embedding 启用开关（`enable_embedding`/`disable_embedding`，memory_interface.py:598-614）——与 memory 检索相关但未见主链调用。

## EK Graph 质量自检（交付前）
- 平均出边：47 条 EK 总 links 约 85+，平均 ≥1.8 ✓
- 游离 EK：2/47 = 4.3% < 20% ✓
- 聚合规则覆盖率：见 03 层，8/8 KO 声明 aggregation_rule ✓

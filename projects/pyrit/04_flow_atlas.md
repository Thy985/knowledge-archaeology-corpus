# 04 · Flow Atlas（七类流）— PyRIT（ARCH-2026-10-03-001）

> 关键 Edge 可回溯 symbol/file/condition/state transition。每条 Edge 附 `证据: 文件:符号`。

## 1. Control Flow（控制流）— 谁驱动谁执行
```
CLI(pyrit_scan) / GUI → ScenarioRunService.start_run_async ──→ Scenario(场景)
    → AttackExecutor.execute_attack_from_seed_groups_async ──→ per-task AttackContext
    → AttackStrategy.execute_with_context_async
        → _setup_async → _perform_async → _teardown_async     [强制三段式]
        → _handle_event_async → StrategyEventHandler.on_event_async  [事件旁路]
    → SequentialAttack._run_child_attack_async → 子 AttackStrategy    [组合编排]
```
- **E-C1** CLI→ScenarioRunService：`cli/pyrit_scan.py` + `backend/services/scenario_run_service.py:242 start_run_async`
- **E-C2** Service→Scenario：`scenario_run_service.py:617 _prepare_run_async → resolve_scenario_class`
- **E-C3** Scenario→Executor：`scenario` 编排 → `attack_executor.py:169 execute_attack_from_seed_groups_async`
- **E-C4** Executor→Strategy：`attack_executor.py:404 run_one_async → strategy.execute_with_context_async`
- **E-C5** 生命周期强制：`executor/core/strategy.py:310 execute_with_context_async`（setup→perform→teardown 顺序硬编码）
- **E-C6** 组合控制：`sequential_attack.py:288 _run_child_attack_async` + `:345 _should_stop_after`（SequenceCompletionPolicy 提前终止分支）
- **E-C7** 恢复控制：`scenario_run_service.py:253 resume_run_async`（凭 scenario_result_id 恢复，`_validate_resume_admission_async:282` 准入校验）
- **条件分支**：`return_partial_on_failure`（attack_executor.py:169）→ True 返回部分结果 / False 抛 `_raise_first_fatal_exception`(474)

## 2. State Flow（状态流）— 运行时状态如何迁移
```
AttackContext（含 attribution/objective_sha256）→ 执行中（semaphore 持有）
    → AttackResult（success/failure/undetermined 由 score 决定）
    → 持久化（memory）→ 可被 ScenarioRunService resume 恢复
```
- **E-S1** 任务身份：`attack_executor.py:198-214` attribution 盖章 → per-task identity 由 `objective_sha256` 重建
- **E-S2** outcome 状态机：`attack_strategy.py attack_outcome_from_score`——score.status ∈ {success, failure, undetermined}；undetermined→undetermined（不弱化）
- **E-S3** 运行进度：`scenario_progress_read_model.py`（956 行）——后台运行进度读模型；`scenario_run_service.py:170 _ActiveRunSnapshot`
- **E-S4** 并发状态：`attack_executor.py:150 _get_semaphore → asyncio.Semaphore`（max_concurrency 限流）
- **E-S5** 结果聚合状态：`AttackExecutorResult.has_incomplete/all_completed/exceptions`（attack_executor.py:83-97）

## 3. Data Flow（数据流）— 数据如何流转
```
SeedPrompt/AttackSeedGroup → params_type.from_seed_group() → AttackParameters（field_overrides 覆盖）
    → Converter（载荷变换）→ PromptTarget（目标调用）
    → Message 响应 → Score（objective + auxiliary）→ AttackResult（持久化）
```
- **E-D1** 参数提取：`attack_executor.py:238 build_params_async → params_type.from_seed_group()`（docstring："automatically handling which fields the attack accepts"）
- **E-D2** 参数覆盖优先级：`field_overrides`（per-seed-group）> `broadcast_fields`（attack_executor.py:169 docstring）
- **E-D3** 对抗对话数据：`adversarial_conversation_manager.py:376 _AdversarialConversationManager`（系统 prompt/首条/后续消息/response_json_schema）
- **E-D4** 发送语义：`prepended_history_send_context.py:125 select_history` + `:154 remap_for_duplicate_conversation`
- **E-D5** 评分数据：`attack_scoring.py:74 score_attack_response_async → MessageScorer.score_response_async`（objective_scores + auxiliary_scores dict）
- **E-D6** 分块评分：`score/text_chunking.py`（长响应分块）
- **E-D7** 配置数据：`.pyrit_conf_example` → `cli/_config_reader.py` → Initializers → registry 装配

## 4. Evidence Flow（证据流）— 证据如何产生与被引用
```
AttackResult（含 attribution_parent_id + objective_sha256）→ Score（判定）→ CentralMemory 持久化
    → keyset 游标分页查询 → 报告/复现
```
- **E-E1** 证据生成：`attack_executor.py:198` attribution 盖章到每任务 context → 持久化行带 attribution 字段
- **E-E2** 判定证据：`score.py:314-318` Score.value 读取遇 undetermined 抛 `UndeterminedScoreError`（无判定不产出值）
- **E-E3** 回退证据：`fallback_scorer.py:166 _fallback_rationale`——回退发生时记录理由
- **E-E4** 检索证据：`memory_interface.py:158 AttackResultKeysetCursor` + `:794 keyset seek`（大结果集分页）
- **E-E5** 重试证据：`exceptions/retry_collector.py get_retry_collector`——重试事件集中收集

## 5. Authority Flow（权威流）— 谁有权判定
```
Registry.register_class（注册校验）→ 组件构造权威（resolve_constructor_args）
    → Score（攻击判定唯一权威）→ AttackOutcome 映射
    → 无事后清扫（"no separate post-hoc sweep"）
```
- **E-A1** 注册权威：`registry/registry.py` docstring——register_class 校验 build contract + reference 映射 wired registry
- **E-A2** 构造权威：`registry/resolution.py resolve_constructor_args`（简单值强转 + registry-reference 按名解析）
- **E-A3** 判定权威：`attack_strategy.py attack_outcome_from_score`——"stated once so no attack invents its own"
- **E-A4** 阻断权威：`scenario.py:497 _apply_scorer_block_policy`（场景级评分器阻断）
- **E-A5** 准入权威：`scenario_run_service.py:282 _validate_resume_admission_async`（恢复运行准入校验）

## 6. Memory Flow（记忆流）— 状态如何被记忆
```
MemoryInterface（SQLAlchemy AsyncEngine）→ 表：PromptMemoryEntries / TargetIdentifiers / ConverterIdentifiers / ScorerIdentifiers
    → identifier 哈希主键关联 → alembic 迁移
```
- **E-M1** 统一入口：`memory_interface.py:349 MemoryInterface` + `central_memory.py`（CentralMemory）
- **E-M2** 会话管理：`memory_interface.py:416 get_session_async` + `:468 _run_database_operation_async`
- **E-M3** 表结构：`memory_models.py:264 PromptMemoryEntries`；`:585 TargetIdentifiers`；`:717 ScorerIdentifiers`；`:638 TargetIdentifierChildren`（FK→hash 主键）
- **E-M4** 标识镜像：`memory_models.py:455 ComponentIdentifierEntry`（"concrete identifier tables inherit the abstract"）
- **E-M5** 迁移：`memory/alembic/versions/`（含 e5f7a9c1b3d2_add_identifiers_tables.py 1523 行）
- **E-M6** 兼容：`memory_interface.py:432 _uses_legacy_session_override` / `:522 _uses_legacy_memory_override`

## 7. Policy Flow（策略流）— 治理闭环
```
设计决策（docstring 声明）→ 策略类型（Enum）→ 执行点强约束 → 未来决策依据
```
- **E-P1** 基线策略：`scenario.py:92 BaselineAttackPolicy`（场景声明基线）
- **E-P2** 序列完成策略：`sequential_attack.py:48 SequenceCompletionPolicy`（子攻击提前终止）
- **E-P3** 重试策略：`exception_classes.py:74 _DynamicStopAfterAttempt` / `:90 _DynamicWaitRandomExponential`（环境变量运行时调参）
- **E-P4** 失败策略：`attack_executor.py` `return_partial_on_failure`（失败语义策略化）
- **E-P5** 治理声明（ADR 职能）：registry.py / attack_strategy.py / sequential_attack.py / memory_models.py docstring（设计意图显式化，本仓库无独立 ADR 目录）

---
## Flow→KO 交叉校验
- KO-01（判定契约）← Evidence Flow E-E2/E-E3 + Authority E-A3 ✓
- KO-02（注册即校验）← Authority E-A1/E-A2 ✓
- KO-03（生命周期强制）← Control E-C5 + State E-S2 ✓
- KO-05（失败显式化）← Control 条件分支 + State E-S5 + Policy E-P3/E-P4 ✓
- KO-06（审计链）← Evidence E-E1 + Memory E-M3/E-M4 ✓
- 无 KO 与 Flow 矛盾项。

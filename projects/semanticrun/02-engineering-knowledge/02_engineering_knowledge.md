# 02 Engineering Knowledge — SemanticRun（ARCH-2026-11-05-001）

> Engineering Knowledge 层（L1/L2 宽底座 · 推理原材料）。22 条 EK，全部声明 `links`（EK Graph 六类边）。
> 证据可回溯：每条 EK 标注 source（file:symbol）+ evidence（测试/代码）。**孤立 EK = 0 条；聚合规则覆盖率 100%（见 03 层）。**

## EK 清单

### EK-01 同步 checkpoint 先于 step 返回（durability 优先）
- **claim**：step 结束（含异常退出）前 checkpoint 已同步落库；异步路径也通过 `drain()` 等待持久化完成才返回。
- **source**：`kernel/run_handle.py:_end_step → _enqueue_checkpoint`；`kernel/checkpoint_worker.py:drain`；`kernel/step_context.py:__exit__`
- **evidence**：`tests/test_sync_durability.py`（S4）；README "Sync checkpoints — written before the step returns"
- **links**：`{"to":"EK-15","type":"causal","direction":"out"}`（step 完成触发 worker 写入）；`{"to":"EK-09","type":"subsystem","direction":"undirected"}`（同属 checkpoint 触发子系统）；`{"to":"EK-02","type":"dependency","direction":"out"}`（依赖 SQLite 持久化保证）
- **value**: A ｜ **abstraction**: L2

### EK-02 WAL + synchronous=NORMAL 的 SQLite 持久化折中
- **claim**：持久化用 SQLite WAL 模式 + synchronous=NORMAL，在崩溃安全与写吞吐间取折中；schema 5 表（runs/checkpoints/audit_events/approvals/side_effects）+ 4 索引。
- **source**：`storage/sqlite.py:_init_schema`（PRAGMA journal_mode=WAL / synchronous=NORMAL；CREATE TABLE IF NOT EXISTS ×5；CREATE INDEX ×4）
- **evidence**：代码存在（S3）；44 tests 全过含持久化路径（S4，正常关闭→重开路径）；**无断电/崩溃恢复测试（test_sync_durability 为 close→reopen，非 kill -9），NORMAL 模式断电丢最近提交语义未验证 → 证据等级收紧为 S3**
- **links**：`{"to":"EK-01","type":"dependency","direction":"in"}`；`{"to":"EK-19","type":"mechanism","direction":"undirected"}`（同为 schema 演进机制）
- **value**: B ｜ **abstraction**: L1

### EK-03 hash_exclude 规范化（ephemeral 字段剥离后哈希稳定）
- **claim**：tool result 哈希前按 `hash_exclude` 剥离易变字段（支持点路径嵌套），使"同一业务结果"哈希稳定、时间戳/请求 ID 等不入哈希。
- **source**：`checkpoint/hashing.py:_strip_keys/canonicalize_for_hash/hash_tool_result`
- **evidence**：`tests/test_checkpoint.py:test_hash_exclude_ignores_ephemeral_fields`；`tests/test_artifact_diff.py:test_diff_tool_results_respects_hash_exclude`（S4）
- **links**：`{"to":"EK-06","type":"mechanism","direction":"undirected"}`（同为"机械确定性"机制）；`{"to":"EK-04","type":"dependency","direction":"out"}`（outbound 哈希复用同一规范化）
- **value**: A ｜ **abstraction**: L2

### EK-04 ACRFence outbound replay gate（外部副作用重放门）
- **claim**：外部副作用（filesystem/process/external）重放前，将 outbound payload 哈希与最后一条已提交记录对比；不一致则标记 `replay_permitted=False` 并返回 `FLAG_DIVERGENCE`，调用方抛 `OutboundDivergenceError` 拒绝重放（fail-closed）。
- **source**：`kernel/ledger.py:SideEffectLedger.permit_replay/compare_outbound`；`kernel/step_context.py:OutboundDivergenceError/StepContext.tool`
- **evidence**：`tests/test_acrfence_replay.py:test_divergent_payload_flagged_not_replayed`（S4）
- **links**：`{"to":"EK-18","type":"causal","direction":"out"}`（分歧→FailFast abort）；`{"to":"EK-03","type":"dependency","direction":"in"}`（复用哈希规范化）；`{"to":"EK-13","type":"mechanism","direction":"undirected"}`（同为环境强制 fail-closed）
- **value**: A ｜ **abstraction**: L2

### EK-05 READ_ONLY_PATTERNS 显式可编辑只读列表（DeltaBox 式）
- **claim**：只读动作分类不靠运行时推断，靠一个显式可编辑的模式列表（tool 名 + 可选 arg 子模式 + 可选 command_regex）；未命中 = recovery-relevant。
- **source**：`kernel/skip_rules.py:READ_ONLY_PATTERNS/classify_action`
- **evidence**：`tests/test_skip_rules.py`（S4）；注释 "Editable static list - no runtime judgment beyond pattern match"
- **links**：`{"to":"EK-09","type":"causal","direction":"out"}`（分类结果决定是否 checkpoint）；`{"to":"EK-14","type":"mechanism","direction":"undirected"}`（同为"显式声明优于推断"）
- **value**: A ｜ **abstraction**: L2

### EK-06 DivergenceMatrix 纯机械布尔（no LLM guessing）
- **claim**：分歧矩阵由纯哈希/字符串对比构建（tool_schemas/tool_results/file_tree/model_id/intent/plan/approval 全部机械 diff），行为漂移仅当 host 显式标记才置位。
- **source**：`kernel/divergence_matrix.py:build_divergence_matrix`；`kernel/artifact_diff.py:diff_*`
- **evidence**：`tests/test_divergence_matrix.py`（S4）；docstring "deterministic divergence matrix from mechanical diffs"
- **links**：`{"to":"EK-03","type":"mechanism","direction":"undirected"}`；`{"to":"EK-07","type":"causal","direction":"out"}`（矩阵 flag → policy 映射）；`{"to":"EK-08","type":"constraint","direction":"in"}`（省略字段语义约束矩阵判读）
- **value**: A ｜ **abstraction**: L2

### EK-07 PolicyMapping 9 flag→hook 默认映射 + 别名归一化
- **claim**：9 种分歧 flag 各自映射到显式策略 hook（默认：tool_schema_changed/tool_result_hash_mismatch→RevalidateWithPrompt；file_tree/model_id/approval_state/outbound→FailFast；intent/plan→StrictReset；behavioral→BehavioralDriftPolicy）；别名（fail_fast/revalidate/strict_reset）归一化到注册 hook 名。
- **source**：`policies/mapping.py:DEFAULT_POLICY_MAPPING/HOOK_ALIASES/PolicyMapping`
- **evidence**：`tests/test_policy_router.py`（S4）
- **links**：`{"to":"EK-06","type":"dependency","direction":"in"}`；`{"to":"EK-17","type":"subsystem","direction":"undirected"}`（同为默认策略面）；`{"to":"EK-18","type":"subsystem","direction":"undirected"}`
- **value**: A ｜ **abstraction**: L2

### EK-08 omitted resume fields = "not compared"（省略即不比较）
- **claim**：resume 时 host 未提供的工件字段不参与比较（空 dict/None ≠ "已删除"）；approval 漂移仅在 host 提供 current approval_status 时才比较——pending 审批门正常 resume 不算分歧。
- **source**：`kernel/divergence_matrix.py`（"Omitted resume fields mean 'not compared' — empty dict/None is not 'deleted'"；approval 分支注释）
- **evidence**：`tests/test_approval_gate.py:test_reject_flags_approval_change_on_resume`；`tests/test_divergence_matrix.py`（S4）
- **links**：`{"to":"EK-06","type":"constraint","direction":"out"}`（约束矩阵判读）；`{"to":"EK-20","type":"dependency","direction":"in"}`（依赖 host 注入的 artifacts 语义）
- **value**: A ｜ **abstraction**: L2

### EK-09 checkpoint sparsity（只读轮不 checkpoint）
- **claim**：checkpoint 触发按 recovery-relevant 分类——只读动作（grep/cat/ls/git diff 等）不触发 checkpoint；SIDE_EFFECT_BOUNDARY 仅在显式副作用或分类为 recovery-relevant 时触发；测试验证 20 轮 <50% checkpoint 密度。
- **source**：`checkpoint/triggers.py:should_checkpoint`；`kernel/run_handle.py:_end_step`
- **evidence**：`tests/test_checkpoint_sparsity.py`（test_checkpoint_sparsity_under_half / test_read_only_turns_have_ledger_no_extra_checkpoints / test_recovery_relevant_turns_checkpoint）（S4）
- **links**：`{"to":"EK-05","type":"dependency","direction":"in"}`（分类依赖只读列表）；`{"to":"EK-01","type":"subsystem","direction":"undirected"}`；`{"to":"EK-15","type":"contrast","direction":"undirected"}`（稀疏 vs 全量 checkpoint 两种策略对照）
- **value**: A ｜ **abstraction**: L2

### EK-10 plan cursor 跳过已完成步骤
- **claim**：`run.steps()` 迭代时，completed_steps 中的 plan 条目直接跳过（plan_index 前移），恢复后不重做已完工步骤。
- **source**：`kernel/run_handle.py:steps/advance_cursor`；`kernel/step_context.py:PlanStepHandle`
- **evidence**：`tests/test_cursor_skip.py:test_steps_skips_completed_on_resume`（S4）
- **links**：`{"to":"EK-11","type":"mechanism","direction":"undirected"}`（同为恢复语义机制）；`{"to":"EK-01","type":"causal","direction":"out"}`（cursor 前移触发 checkpoint）
- **value**: A ｜ **abstraction**: L2

### EK-11 green checkpoint + StrictReset 回滚
- **claim**：运行中可标记 last known good checkpoint（green）；StrictReset 策略将状态回滚到 green checkpoint（意图/计划变更时默认启用）。
- **source**：`kernel/run_handle.py:mark_green_checkpoint/_load_green_checkpoint`；`policies/builtin.py:StrictReset`
- **evidence**：`tests/test_resume.py`/`tests/test_policy_router.py`（S4）
- **links**：`{"to":"EK-10","type":"mechanism","direction":"undirected"}`；`{"to":"EK-13","type":"causal","direction":"out"}`（回滚是环境强制的执行路径之一）
- **value**: B ｜ **abstraction**: L2

### EK-12 approval gate 状态持久化（WAITING_APPROVAL 可恢复）
- **claim**：审批请求将状态置 WAITING_APPROVAL 并持久化（approvals 表 + APPROVAL_GATE checkpoint）；approve/reject 更新审批记录 + 状态 + checkpoint；恢复时若 checkpoint 处于 WAITING_APPROVAL 则保持该状态（等待人类，不继续执行）。
- **source**：`kernel/run_handle.py:request_approval/approve/reject`；`runtime.py:resume`（WAITING_APPROVAL 状态保留分支）；`storage/sqlite.py:save_approval/update_approval`
- **evidence**：`tests/test_approval_gate.py`（S4）
- **links**：`{"to":"EK-08","type":"causal","direction":"out"}`（恢复时 approval 对比语义）；`{"to":"EK-01","type":"subsystem","direction":"undirected"}`
- **value**: A ｜ **abstraction**: L2

### EK-13 PolicyAbort 环境强制中断
- **claim**：fail-fast 类策略以异常（PolicyAbort）强制中断执行，调用方必须捕获；outbound 分歧用 OutboundDivergenceError（PolicyAbort 子类）在副作用真正执行前拒绝。
- **source**：`policies/errors.py:PolicyAbort`；`kernel/run_handle.py:apply_policies`（action=="abort" → raise）；`kernel/step_context.py:OutboundDivergenceError`
- **evidence**：`tests/test_cursor_skip.py:test_apply_policies_fail_fast_raises`；`tests/test_acrfence_replay.py`（S4）
- **links**：`{"to":"EK-04","type":"mechanism","direction":"undirected"}`；`{"to":"EK-11","type":"dependency","direction":"in"}`（abort/回滚共享"环境强制"保证）
- **value**: A ｜ **abstraction**: L2

### EK-14 BehavioralDriftPolicy 显式标记，kernel 绝不推断
- **claim**：行为漂移（behavioral_drift）只能由 host 显式标记（behavioral_drift_flagged + reason），kernel 不自动推断；触发后默认 halt_for_human（require_human=True）。
- **source**：`policies/behavioral.py`（docstring "Explicit behavioral drift policy - never inferred by kernel"）；`kernel/divergence_matrix.py`（直通 host 标记）
- **evidence**：`tests/test_divergence_matrix.py:test_matrix_behavioral_drift_explicit_only`（S4）
- **links**：`{"to":"EK-05","type":"mechanism","direction":"undirected"}`（同为显式声明）；`{"to":"EK-07","type":"subsystem","direction":"undirected"}`
- **value**: A ｜ **abstraction**: L2

### EK-15 CheckpointWorker 后台线程 + drain 保证
- **claim**：async_checkpoints=True 时 checkpoint 写入走后台线程（queue + daemon thread）；但每次 step 返回前仍调用 drain()（queue.join()）等待落库——"异步"只解耦写入执行位置，不牺牲 durability。
- **source**：`kernel/checkpoint_worker.py:CheckpointWorker.enqueue/drain/stop`；`kernel/run_handle.py:_enqueue_checkpoint`
- **evidence**：代码存在（S3）；`tests/test_sync_durability.py`（S4）
- **links**：`{"to":"EK-01","type":"causal","direction":"in"}`；`{"to":"EK-02","type":"dependency","direction":"out"}`（写线程依赖 SQLite 连接安全）
- **value**: B ｜ **abstraction**: L2

### EK-16 model_context 从 step metadata 自动提取
- **claim**：`step(..., model="gpt-4.1")` 或 `ctx.llm(..., model=...)` 时按 `split("-", 1)` 提取 model_family/model_version 存入 RunRecord.model_context，供 checkpoint/resume 对比。
- **source**：`kernel/run_handle.py:step`；`kernel/step_context.py:llm`
- **evidence**：代码存在（S3）；`tests/test_resume_hydration.py`（S4）
- **links**：`{"to":"EK-17","type":"causal","direction":"out"}`（model_context → model_id_changed 判定）
- **value**: C ｜ **abstraction**: L1

### EK-17 model_id_changed → FailFast（模型更换视为安全事件）
- **claim**：恢复时 model family/version/fingerprint 任一变化 → 默认 FailFast abort（require_human_reauth），而非盲续。
- **source**：`policies/mapping.py:DEFAULT_POLICY_MAPPING["model_id_changed"]="FailFast"`；`kernel/artifact_diff.py:diff_model_ids`
- **evidence**：`tests/test_divergence_matrix.py:test_matrix_model_id_changed`（S4）；README drift 表 "Model ID swap → abort"
- **links**：`{"to":"EK-16","type":"dependency","direction":"in"}`；`{"to":"EK-07","type":"subsystem","direction":"undirected"}`
- **value**: A ｜ **abstraction**: L2

### EK-18 outbound_payload_divergence → FailFast（重发门）
- **claim**：恢复时 host 提供的 outbound payload 与已提交记录哈希不一致 → 默认 FailFast；运行期即时检测则抛 OutboundDivergenceError。
- **source**：`policies/mapping.py:DEFAULT_POLICY_MAPPING["outbound_payload_divergence"]="FailFast"`；`kernel/ledger.py:check_outbound_divergence`；`kernel/step_context.py:OutboundDivergenceError`
- **evidence**：`tests/test_acrfence_replay.py:test_matrix_outbound...`（S4）
- **links**：`{"to":"EK-04","type":"causal","direction":"in"}`；`{"to":"EK-13","type":"mechanism","direction":"undirected"}`
- **value**: A ｜ **abstraction**: L2

### EK-19 _migrate_schema 增量 ALTER（schema 演进）
- **claim**：SQLite schema 演进用幂等 `PRAGMA table_info` + 条件 `ALTER TABLE ADD COLUMN`（outbound_request_hash / side_effect_class / replay_permitted 三次迁移），不破坏既有库。
- **source**：`storage/sqlite.py:_migrate_schema`
- **evidence**：代码存在（S3）；`tests/test_sync_durability.py`（持久化路径 S4）
- **links**：`{"to":"EK-02","type":"mechanism","direction":"undirected"}`（同为存储层机制）
- **value**: C ｜ **abstraction**: L1

### EK-20 ResumeArtifacts 由 host 注入（环境事实 vs 快照对比）
- **claim**：恢复时"当前世界"的事实由 host agent 通过 ResumeArtifacts 显式注入（model_id/tool_schemas/tool_results/file_tree/intent/plan/approval/outbound），环境只做 checkpoint vs current 的机械对比，不自行探测世界。
- **source**：`models/artifacts.py:ResumeArtifacts`；`runtime.py:resume`（artifacts 参数 + enforce_policies）
- **evidence**：`tests/test_resume_hydration.py`；`tests/test_resume.py`（S4）
- **links**：`{"to":"EK-08","type":"dependency","direction":"out"}`（省略语义约束）；`{"to":"EK-06","type":"dependency","direction":"out"}`（矩阵输入来自 artifacts）
- **value**: A ｜ **abstraction**: L2

### EK-21 audit append-only + divergence 事件可查询
- **claim**：全生命周期事件（run_created/step_started/checkpoint_created/divergence_detected/policy_applied/approval_*/outbound_payload_divergence）append-only 写入 audit_events；提供 divergence 事件定向查询。
- **source**：`audit/log.py:AuditLog`（DIVERGENCE_EVENTS = {"divergence_detected","policy_applied"}）；`storage/sqlite.py:append_audit_event/list_audit_events`
- **evidence**：`tests/test_acrfence_replay.py`（事件存在断言）（S4）
- **links**：`{"to":"EK-06","type":"causal","direction":"out"}`（分歧检测产生审计事件）；`{"to":"EK-04","type":"causal","direction":"out"}`
- **value**: B ｜ **abstraction**: L2

### EK-22 storage backend Protocol 抽象（可替换后端）
- **claim**：存储层定义 StorageBackend Protocol（20 个方法），SQLiteStorage 为实现；RunHandle/Ledger/AuditLog 只依赖 Protocol，不依赖具体后端。
- **source**：`storage/base.py:StorageBackend`；`runtime.py`（注入 SQLiteStorage）
- **evidence**：代码存在（S3）
- **links**：`{"to":"EK-02","type":"subsystem","direction":"undirected"}`（同属存储子系统）
- **value**: C ｜ **abstraction**: L1

## EK Graph 质量检查

- **EK 总数**：22；**有边 EK**：22/22（100%）
- **平均出边**：~2.2（合格 ≥1）
- **游离 EK**：0 条（<20% 合格）
- **孤立/降 D 级**：无（全部连入 KO 簇或子系统）
- **聚合规则覆盖率**：8/8 KO（100%，见 03 层）

## 已知边界（不弱化证据标准）

- EK-02/EK-15/EK-19/EK-22 为 S3（代码存在），S4 证据来自相关测试覆盖路径（持久化/worker 路径），非全路径——如实标注。
- 本项目 README 性能声明（98% token savings / 1-8ms checkpoint）**无仓库内证据**，不在 EK 中升为事实（归入 05 Candidates / 06 Epistemic 核对）。

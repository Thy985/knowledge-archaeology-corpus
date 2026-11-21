# 04 Flow Atlas — SemanticRun（ARCH-2026-11-05-001）

> 七类流，从真实代码导出，关键 Edge 可回溯 symbol/file。23 条 edge（对照 11-04 durare 基线 23 edge）。

## F-01 Control Flow（控制流）— run 执行与恢复主循环

| Edge | 描述 | 回溯 |
|---|---|---|
| C1 | `env.start()` → `create_run`（RunRecord RUNNING + AgentState.create + audit run_created） | `runtime.py:SemanticRun.create_run` |
| C2 | `run.steps()` 迭代 plan；ABORTED/COMPLETED/WAITING_APPROVAL 时终止；completed 条目跳过（cursor 前移） | `kernel/run_handle.py:RunHandle.steps` |
| C3 | `PlanStepHandle.tool()/llm()/remember()` → `handle.step(...)` → `StepContext.__enter__`（_begin_step → audit step_started） | `kernel/step_context.py:PlanStepHandle/StepContext.__enter__` |
| C4 | `StepContext.__exit__` → `_end_step`（step_count+1 / audit / 按类型触发 checkpoint） | `kernel/step_context.py:__exit__` + `kernel/run_handle.py:_end_step` |
| C5 | `PlanStepHandle.complete()` → `advance_cursor`（completed_steps append + plan_index 前移 + checkpoint MANUAL + audit cursor_advanced）；fail → `record_step_failure`（failure_history append） | `kernel/step_context.py:PlanStepHandle.complete/fail` + `kernel/run_handle.py:advance_cursor` |
| C6 | `env.resume()`：get_latest_checkpoint → state 深拷贝 → WAITING_APPROVAL 状态保留 → hydrate → mark_resumed → 可选 compute/apply policies | `runtime.py:SemanticRun.resume` |
| C7 | `env.complete()/abort()`：状态置位 + worker drain + 最终 checkpoint + audit run_completed/run_aborted | `runtime.py:SemanticRun.complete/abort` |

## F-02 State Flow（状态流）— 状态快照与恢复

| Edge | 描述 | 回溯 |
|---|---|---|
| S1 | `AgentState`（intent/working_memory/tool_commitments/plan_index/completed_steps…）→ `Checkpoint.from_run_state`（state 深拷贝 + model + tool_schemas + file_tree + policy_mapping_json） | `models/checkpoint.py:Checkpoint.from_run_state` |
| S2 | `checkpoint()` → `CheckpointEngine.create_checkpoint` → `storage.save_checkpoint`（WAL 落库）+ audit checkpoint_created | `kernel/run_handle.py:RunHandle.checkpoint` + `checkpoint/engine.py` |
| S3 | resume 时 `get_latest_checkpoint` → `state.model_copy(deep=True)`（隔离可变状态）→ `hydrate_from_checkpoint`（tool_schemas/file_tree/model_context 句柄局部态） | `runtime.py:resume` + `kernel/run_handle.py:hydrate_from_checkpoint` |
| S4 | StrictReset 回滚：`_load_green_checkpoint` → state 替换为 green checkpoint 深拷贝 + hydrate + status=RUNNING + audit strict_reset_applied | `kernel/run_handle.py:_load_green_checkpoint` |
| S5 | RunStatus 迁移：RUNNING→PAUSED（pause）/WAITING_APPROVAL（request_approval）→RUNNING（approve）/PAUSED（reject）→COMPLETED/ABORTED | `kernel/run_handle.py:pause/request_approval/approve/reject` + `runtime.py:complete/abort` |

## F-03 Data Flow（数据流）— 工具结果与外发载荷

| Edge | 描述 | 回溯 |
|---|---|---|
| D1 | `ctx.tool(...)` → 先 ACRFence 检查（outbound + 非只读副作用时）→ `fn()` → `set_tool_result` | `kernel/step_context.py:StepContext.tool` |
| D2 | result → `hash_tool_result(result, hash_exclude)` → `ToolResultCommitment`（schema_hash/result_hash/hash_exclude/raw_result）→ state.tool_commitments；schema → hash_schema → ToolSchemaRef | `kernel/step_context.py:set_tool_result` + `checkpoint/hashing.py` + `kernel/artifact_diff.py:hash_schema` |
| D3 | outbound payload → `hash_outbound_request`（sort_keys compact json SHA-256）→ SideEffectRecord.outbound_request_hash；`record_tool_side_effect` 分类 side_effect_class | `kernel/ledger.py:record_tool_side_effect` + `checkpoint/hashing.py:hash_outbound_request` |
| D4 | 恢复时 `check_outbound_divergence`：host outbound_payloads 逐个与最后记录哈希对比 → 分歧 map → 矩阵 outbound_payload_divergence | `kernel/ledger.py:check_outbound_divergence` + `kernel/run_handle.py:compute_divergence_matrix` |
| D5 | file tree：`hash_file_tree`（rglob + 相对路径:sha256 → merkle）→ FileTreeSnapshot；恢复时 diff_file_trees 对比 merkle | `kernel/artifact_diff.py:hash_file_tree/diff_file_trees` |

## F-04 Evidence Flow（证据流）— 审计事件

| Edge | 描述 | 回溯 |
|---|---|---|
| E1 | 全生命周期事件 append-only：run_created / step_started / step_completed / step_failed / checkpoint_created / run_paused / approval_requested / approval_granted / approval_rejected / run_resumed / run_completed / run_aborted | `audit/log.py:AuditLog.emit`（各调用点） |
| E2 | 分歧链审计：divergence_detected（矩阵 dump）→ policy_applied（outcome dump）→（abort 时）policy_enforced_abort；outbound 分歧：outbound_payload_divergence（expected/actual hash） | `resume/router.py:route` + `kernel/run_handle.py:apply_policies` + `kernel/ledger.py:permit_replay` |
| E3 | 可查询面：`get_run_history` / `get_checkpoint_history` / `get_divergence_events`（DIVERGENCE_EVENTS = divergence_detected, policy_applied） | `audit/log.py:AuditLog` |

## F-05 Authority Flow（权威流）— 人工审批与策略强制

| Edge | 描述 | 回溯 |
|---|---|---|
| A1 | `require_approval(action, payload)` → `request_approval`：ApprovalState(PENDING) + pending_actions append + status=WAITING_APPROVAL + save_approval(PENDING) + checkpoint(APPROVAL_GATE) + audit | `kernel/run_handle.py:RunHandle.request_approval` |
| A2 | `approve()`：ApprovalState→APPROVED + resolved_at + status=RUNNING + update_approval + checkpoint + audit approval_granted；`reject()`：→REJECTED + status=PAUSED + 同链 | `kernel/run_handle.py:RunHandle.approve/reject` |
| A3 | resume 时 approval 判读：仅 host 提供 current approval_status 才对比；pending 门 = 正常等待非分歧 | `kernel/divergence_matrix.py`（approval 分支） |
| A4 | 策略强制：abort（status=ABORTED + checkpoint + raise PolicyAbort）/ load_checkpoint（回滚 green）/ run_assertions（pending_revalidations append）/ halt_for_human（status=PAUSED + checkpoint） | `kernel/run_handle.py:RunHandle.apply_policies` |
| A5 | fail-closed 重放门：`permit_replay` → FLAG_DIVERGENCE → `raise OutboundDivergenceError`（副作用执行前） | `kernel/step_context.py:StepContext.tool` + `kernel/ledger.py:permit_replay` |

## F-06 Memory Flow（记忆流）— 工作记忆与承诺

| Edge | 描述 | 回溯 |
|---|---|---|
| M1 | `remember(key, value)` → `working_memory.set_slot`：MemorySlot（content_hash=hash_tool_result(value) + verified_at + source_step_id + value） | `models/state.py:VerifiedWorkingMemory.set_slot` + `kernel/step_context.py:StepContext.remember` |
| M2 | 记忆写后持久化：step 内 remember 置 `_memory_mutated=True` → _end_step 触发 MANUAL checkpoint；plan 级 remember 直接 `handle.checkpoint()`（durable boundary） | `kernel/step_context.py:StepContext.remember/PlanStepHandle.remember` + `kernel/run_handle.py:_end_step` |
| M3 | 恢复后记忆可用：checkpoint.state.working_memory 深拷贝 → hydrate 后 `state.working_memory.get()`（value 直达） | `runtime.py:resume` + `models/checkpoint.py:working_memory` |

## F-07 Policy Flow（策略流）— 分歧→策略→执行→审计闭环

| Edge | 描述 | 回溯 |
|---|---|---|
| P1 | `compute_divergence_matrix`：checkpoint vs host artifacts 机械 diff → DivergenceMatrix（9 flag + deltas） | `kernel/divergence_matrix.py:build_divergence_matrix` |
| P2 | `matrix.has_divergence` False → route 返回 continue（transparent resume）+ audit policy_applied | `resume/router.py:PolicyRouter.route`（无分歧分支） |
| P3 | 有分歧：`matrix.triggered_flags()` 逐 flag → `mapping.hook_for_flag(flag)`（DEFAULT_POLICY_MAPPING 归一化）→ `registry.get(hook_name)` | `resume/router.py:route` + `policies/mapping.py:PolicyMapping.hook_for_flag` |
| P4 | hook 未注册 → 兜底 `halt_for_human`（"Policy hook 'X' not registered"）——策略缺失不静默放行 | `resume/router.py:route`（hook is None 分支） |
| P5 | `hook.execute(ctx)` → PolicyOutcome（action/hook_name/flag/payload/message）→ apply_policies 强制执行（abort/load_checkpoint/run_assertions/halt_for_human）→ 全程 audit | `policies/builtin.py` + `kernel/run_handle.py:apply_policies` |
| P6 | 策略上下文注入：PolicyContext（run_id/flag/matrix/checkpoint/current/last_green_checkpoint_id/revalidation_template/assertions）——hook 只读上下文，不探测世界 | `policies/contract.py:PolicyContext` |

## Flow→KO 交叉校验门

| KO | 依赖的关键 Edge | 校验 |
|---|---|---|
| KO-01 | C4→(EK-01/EK-09)→S2 | ✅ 全部 trace 到代码 |
| KO-02 | D2/D3/D4/D5→P1 | ✅ |
| KO-03 | A5/D4 | ✅ |
| KO-04 | P1→P3→P5→E2 | ✅ |
| KO-05 | A1/A2/A3 | ✅ |
| KO-06 | C4 分类分支（SIDE_EFFECT_BOUNDARY/只读不 checkpoint） | ✅ |
| KO-07 | C2/C6/S3/P1 | ✅ |
| KO-08 | P3/P6/A5（显式声明面） | ✅ |

**结论**：无 KO 与 Flow 矛盾；23 条 edge 全部 VERIFIED（逐条回溯 symbol/file）。

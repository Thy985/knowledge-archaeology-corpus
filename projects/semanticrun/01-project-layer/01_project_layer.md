# 01 Project Layer — SemanticRun（ARCH-2026-11-05-001）

> 项目地图（L0 事实底座）。全部事实来自仓库实际内容，证据可回溯到文件。

## 1.1 项目定位

- **名称**：SemanticRun（PyPI `semanticrun`，包名 `semanticrun`；曾用名 semarun / SemaRun）
- **版本**：0.4.2（`semanticrun/__init__.py` `__version__ = "0.4.2"`；pyproject `[project] version`）
- **一句话**：artifact-aware durable agent environment——"freezes what your agent already committed — and resumes with proof of what drifted"（README）
- **分类**：Development Status 3-Alpha（pyproject classifier）；MIT License
- **作者**：ysharmcode（pyproject authors）；仓库 `vogue84/SemanticRun`
- **仓库元数据**：7★ / 0 fork / Python / created 2026-07-30 / last pushed 2026-08-04（~3 个月静止）/ default_branch master / 416KB

## 1.2 架构（模块依赖关系）

```
SemanticRun（runtime.py 门面）
├── storage/SQLiteStorage（sqlite.py：WAL + 5 表 + 迁移）
├── audit/AuditLog（log.py：append-only 事件）
├── kernel/SideEffectLedger（ledger.py：副作用账本 + ACRFence）
├── checkpoint/CheckpointEngine（engine.py：快照创建）
├── kernel/CheckpointWorker（kernel/checkpoint_worker.py：异步写入，drain 保证；默认 async_checkpoints=False 时 worker 为 None，走同步路径）
├── policies/PolicyRegistry + PolicyRouter（contract.py / resume/router.py）
├── policies/PolicyMapping（mapping.py：9 flag→hook）
├── policies/builtin（FailFast / RevalidateWithPrompt / StrictReset）
├── policies/behavioral（BehavioralDriftPolicy）
├── kernel/RunHandle（run_handle.py：生命周期句柄）
├── kernel/StepContext + PlanStepHandle（step_context.py）
├── kernel/divergence_matrix（divergence_matrix.py：机械 diff 矩阵）
├── kernel/artifact_diff（artifact_diff.py：哈希 diff）
├── kernel/skip_rules（skip_rules.py：只读动作分类）
└── models/*（state / checkpoint / artifacts / divergence：pydantic 模型）
```

**分层**：门面（runtime）→ 句柄（run_handle）→ 内核（step_context / ledger / divergence）→ 持久化（sqlite）→ 模型（pydantic）。policies 为横切层（registry 注册 + router 路由 + hook 执行）。

## 1.3 核心模块职责

| 模块 | 职责 | 关键符号 |
|---|---|---|
| `runtime.py` | 环境门面：open/start/create_run/resume/complete/abort | `SemanticRun` class |
| `kernel/run_handle.py` | run 生命周期句柄：steps 迭代 / checkpoint / pause / approval / divergence / policy 执行 | `RunHandle`（453 行，最大模块） |
| `kernel/step_context.py` | 单步上下文（with 进入/退出）+ plan cursor 迭代器 | `StepContext` / `PlanStepHandle` |
| `kernel/ledger.py` | 副作用账本：记录 + outbound replay gate（ACRFence） | `SideEffectLedger` / `ReplayVerdict` |
| `kernel/divergence_matrix.py` | 构建分歧矩阵（机械布尔） | `build_divergence_matrix` |
| `kernel/artifact_diff.py` | 纯机械 diff（schema/tool result/file tree/model） | `diff_*` 函数 |
| `kernel/skip_rules.py` | 只读动作分类（显式模式列表） | `classify_action` / `READ_ONLY_PATTERNS` |
| `storage/sqlite.py` | SQLite 持久化（WAL，5 表 4 索引，增量迁移） | `SQLiteStorage` |
| `checkpoint/engine.py` | 快照创建 + summary 文本 | `CheckpointEngine.create_checkpoint` |
| `checkpoint/triggers.py` | checkpoint 触发规则（5 类 trigger） | `should_checkpoint` |
| `checkpoint/hashing.py` | 规范化哈希（hash_exclude / sort_keys compact） | `hash_tool_result` |
| `resume/router.py` | 矩阵 flag → policy hook 路由 | `PolicyRouter.route` |
| `policies/*` | 策略契约 + 内置 hook + 映射 | `PolicyHook` / `FailFast` / `PolicyMapping` |
| `audit/log.py` | append-only 审计 + divergence 事件查询 | `AuditLog` |
| `models/*` | 全部 pydantic 状态模型 | `AgentState` / `Checkpoint` / `DivergenceMatrix` 等 |

## 1.4 核心数据结构（状态模型）

- **RunStatus**：running / paused / waiting_approval / completed / aborted（`models/state.py`）
- **StepType**：llm_call / tool_call / human_approval / retrieval / validation / reasoning
- **AgentState**：active_intent（intent+plan+version）/ working_memory（slots：content_hash+verified_at+source_step_id）/ tool_commitments（ToolResultCommitment：schema_hash+result_hash+hash_exclude）/ verified_claims / open_questions / pending_actions / approval_state / failure_history / green_checkpoint / plan_index / completed_steps
- **Checkpoint**：state 深拷贝 + model_context + model_id + tool_schemas + file_tree + policy_mapping_json + summary_text
- **ResumeArtifacts**（host 恢复时注入）：model_id / tool_schemas / tool_results / file_tree / intent_text / plan / approval_status / behavioral_drift_flagged(+reason) / outbound_payloads
- **DivergenceMatrix**：9 个机械布尔/dict flag + deltas + triggered_flags()
- **SideEffectRecord**：kind / target / payload_hash / schema_hash / outbound_request_hash / side_effect_class / replay_permitted
- **SideEffectClass**：read_only / filesystem / process / external

## 1.5 生命周期

1. **创建**：`SemanticRun.open(db_path)` → `env.start(intent, plan, policies)` → `create_run`（RunRecord RUNNING + AgentState.create + audit run_created）
2. **执行**：`run.steps()` 迭代 plan → `PlanStepHandle.tool()/llm()/remember()/require_approval()` → step 完成触发 checkpoint（SIDE_EFFECT_BOUNDARY / LLM / MANUAL / PERIODIC / APPROVAL_GATE）→ `advance_cursor`（completed_steps append + plan_index 前移 + checkpoint）
3. **审批门**：`request_approval` → WAITING_APPROVAL + save_approval(PENDING) + checkpoint；`approve`→RUNNING / `reject`→PAUSED
4. **恢复**：`env.resume(run_id, artifacts, enforce_policies=True)` → get_latest_checkpoint → state 深拷贝 → hydrate（tool_schemas/file_tree/model_context）→ mark_resumed → compute_divergence_matrix → apply_policies（abort / strict_reset / revalidate / halt）
5. **终止**：`complete`（COMPLETED + 最终 checkpoint）/ `abort`（ABORTED + 最终 checkpoint）

## 1.5b Public API 导出面

`semanticrun/__init__.py` 显式导出（`test_export.py` 锁定 `from semanticrun import *`）：`SemanticRun` / `PolicyAbort` / `PolicyMapping`（以及模块内部类）。公共入口仅 `runtime.SemanticRun`；策略对象通过构造参数注入。

## 1.6 配置 / 依赖 / 权限与治理

- **构建配置**：hatchling；`pip install -e ".[dev]"`；pytest testpaths=["tests"] pythonpath=["."]
- **运行时依赖**：仅 `pydantic>=2.0`；dev：`pytest>=7.0`
- **SQLite 配置**：`PRAGMA journal_mode=WAL` + `PRAGMA synchronous=NORMAL`（写性能/持久性折中）
- **权限与治理机制**（本项目的安全面）：
  1. 人工审批门（approval gate）——敏感动作必须人类审批
  2. FailFast → `require_human_reauth=True`（abort 需人类重新认证）
  3. BehavioralDriftPolicy → `halt_for_human`（行为漂移必须人工接管）
  4. StrictReset → 回滚 last green checkpoint
  5. ACRFence outbound replay gate——外部副作用重放前哈希对比，不一致拒绝（fail-closed）
  6. PolicyAbort 异常——环境强制中断，调用方必须捕获
- **CI**：`.github/workflows/test.yml`（ubuntu/windows × py3.11/3.12）；`publish.yml`（version tag → PyPI）
- **Benchmark**：`benchmarks/drift_recovery/`（5 场景 × 3 策略；OPENROUTER_API_KEY 时用真实模型，否则确定性 stub）

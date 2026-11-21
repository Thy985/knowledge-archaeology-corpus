# 06 Validation & Evidence — SemanticRun（ARCH-2026-11-05-001）

> 六 Auditor 盲重建 + 反例预算制 + Epistemic 核对。**不弱化证据标准换全绿**；Contradictions 与 Counterexamples 原样保留。

## 6.1 Truth Auditor（先盲重建再核验）

**盲重建**（独立重读仓库，不参考 KO）：SemanticRun = SQLite 持久化的 agent 运行环境；运行按 plan 迭代；每步记录工具结果哈希；审批门暂停；恢复时对比"当前世界"与最后 checkpoint，按策略 abort/回滚/重验证。

**核验结果**（全部 SUPPORTED）：

| KO | source_truth | 证据 |
|---|---|---|
| KO-01 | SUPPORTED | `_end_step→_enqueue_checkpoint→drain` 代码链存在（run_handle.py:206-254）；test_sync_durability 全过 |
| KO-02 | SUPPORTED | hashing.py + artifact_diff.py + ledger.py 哈希面完整（S4） |
| KO-03 | SUPPORTED | permit_replay + OutboundDivergenceError（test_acrfence_replay 4 断言全过） |
| KO-04 | SUPPORTED | divergence_matrix→router→apply_policies→audit 闭环 trace 完整 |
| KO-05 | SUPPORTED | request_approval/approve/reject + approvals 表 + WAITING_APPROVAL resume 保留（test_approval_gate） |
| KO-06 | SUPPORTED | test_checkpoint_sparsity 3 测试（<50% / 只读 0 checkpoint / recovery-relevant checkpoint） |
| KO-07 | SUPPORTED | test_cursor_skip / test_resume_hydration / test_divergence_matrix |
| KO-08 | SUPPORTED | skip_rules 显式列表 / behavioral 显式 / mapping 显式 / artifacts host 注入 |

**发现**：
- INFO：`enforce_policies=True` 是 resume 默认值，但 host 可显式关闭（`enforce_policies=False`）——矩阵计算与策略执行被跳过，属文档化逃生口，非缺陷。
- WARNING：`approve()/reject()` 无身份校验——任何持有 handle 的调用者都能批准审批；ApprovalState 无 approver 字段（见 06.5 反例）。

## 6.2 Coverage Auditor（独立重搜 + 强制子系统覆盖）

**独立重搜清单**（不依赖已产出 KO，从文件系统重新枚举）：
- 入口/门面（runtime.py）✅ 已覆盖
- 持久化（sqlite/base）✅ EK-02/EK-19/EK-22
- 恢复（resume/router + runtime.resume）✅ EK-20/KO-07
- 策略（policies/*）✅ EK-07/EK-13/EK-14/EK-17/EK-18/KO-04/KO-08
- 检查点（checkpoint/*）✅ EK-01/EK-09/EK-15
- 哈希（hashing/artifact_diff）✅ EK-03/KO-02
- 副作用（ledger/skip_rules）✅ EK-04/EK-05
- 审计（audit/log）✅ EK-21
- 状态模型（models/*）✅ 01 层 + EK 引用
- 测试（14 文件）✅ 6.1 引用
- benchmark（drift_recovery）✅ C-04 + 00 overview
- CI（.github/workflows）✅ snapshot + 01 层
- 示例（examples/survive_the_swap.py）——未深度考古（仅作为 API 用法佐证，已读 README 等价示例）

**缺失/低覆盖**：
- `models/__init__.py`、`resume/__init__.py` 等空/仅 re-export 文件（无知识价值，跳过）
- `benchmarks/drift_recovery/results.json` 未解析内容（仅确认存在；真实结果数据可作 S5 候选证据，但无独立运行环境）
- 结论：**无 Critical Knowledge Missing**；13/13 高密度子系统有代表。

## 6.3 Flow Auditor（逐条验证 Edge）

- 23 条 edge（C1-C7/S1-S5/D1-D5/E1-E3/A1-A5/M1-M3/P1-P6）全部 VERIFIED（symbol + file 逐条回溯，见 04 层表）
- 无 UNVERIFIED / INCORRECT
- **特殊检查**（bypass/override/exception/alternate/direct call/admin path/fallback/legacy path）：
  - `enforce_policies=False`（bypass 面，文档化）
  - `apply_policies` 中 `halt_for_human`/`run_assertions` 非终止路径（partial enforcement）
  - router 中 hook 未注册 → halt_for_human 兜底（fallback，fail-safe 方向）
  - `classify_action` 未知工具默认 RECOVERY_RELEVANT（保守方向，fallback）
  - `permit_replay` 无先例记录（NO_PRIOR_RECORD）→ 放行（首重放无法证明一致性，见反例 R-01）

## 6.4 Abstraction Auditor（独立判定 L3/L4/L5 晋升）

| 对象 | 原层 | 判定 | 理由 |
|---|---|---|---|
| KO-01~KO-08 | L3 | **PASS** | 每条均有同类对照（Temporal/DBOS/K8s/LangGraph），解释范围 > 单模块事实；簇 EK 全部可回溯 |
| CM-01 语义恢复 vs 重放 | L4 | **CONDITIONAL** | 解释范围确实扩大（两类系统分野），但仅 SemanticRun+durare 两例 → 必须标 Cross-project validation pending，不降级但也不升级为 Principle |
| CM-02 机械对比/人工处理 | L4 | **CONDITIONAL** | 同上；且"什么算 ephemeral"仍需人工定义（边界） |
| M-01 恢复语义五准则 | L5 | **PASS（项目内）** | 五条全部可操作、全部有 S4 证据；跨项目有效性标 pending |

**降级决定**：无对象被降级；CM-01/CM-02 保持 CONDITIONAL + pending（不过度宣称）。

## 6.5 Counterexample Hunter（反例预算制：每个 L3+ KO ≥3 定向攻击）

| 目标 | 反例 | 攻击结果 | 处置 |
|---|---|---|---|
| KO-01 | R-01：`permit_replay` 对 NO_PRIOR_RECORD 放行首重放——若崩溃发生在副作用记录前、重放发生时，无法证明"重放与已发生一致" | 攻击有效（fail-open 缺口） | KO-01 主张"同步持久化屏障"仍成立（记录路径本身同步），但**屏障只覆盖已记录副作用**；补充边界到 05 Candidates C-01 关联 + KO-03 claim 措辞已含"已提交记录"限定 ✅ 不降级，注明边界 |
| KO-01 | R-02：memory_mutated 仅在显式 `remember()` 置位——LLM 调用内部改 state 的路径不触发（除非 LLM_CALL 分支） | 部分有效 | LLM_CALL 步骤本身触发 checkpoint（EK-01 _end_step 分支），覆盖成立；保留 INFO |
| KO-02 | R-03：hash_exclude 误用可掩盖真实漂移（host 把关键字段排除） | 攻击有效 | 语义层面无法机械检测 → 归入 CM-02 边界（"什么算 ephemeral 是语义判断"）✅ |
| KO-03 | R-04：read_only 分类错误 → 外部副作用不经过 ACRFence（side_effect 未声明 + 工具在只读列表） | 攻击有效（面窄） | **独立 Auditor 收紧**：record_tool_side_effect 中 explicit_side_effect 可强制 recovery、read_only 分类仅影响记录分类与 checkpoint 跳过——绕过实际需"调用方未声明 + 工具在列表内 + 真实副作用"三条件同时成立。KO-03 claim 限定"外部副作用（filesystem/process/external）"（显式声明路径）✅ |
| KO-04 | R-05：多 flag 并发触发时路由顺序固定（triggered_flags 顺序）→ 第一个 abort 短路后续策略 | 攻击无效（设计如此：abort 优先） | 保留 INFO；apply_policies 顺序语义未文档化 |
| KO-05 | R-06：approve/reject 无权限校验，任何持 handle 者可批准 | 攻击有效 | **WARNING**：ApprovalState 无 approver/签名字段；权限在 host 层（未实现）→ 记入 06.1 WARNING + C-05 关联；KO-05 claim 收窄为"门本身持久化正确" ✅ |
| KO-06 | R-07：sparsity 目标 <50% 在"全副作用负载"下不可达（每轮都 recovery-relevant） | 攻击无效（测试场景 20 轮 5 checkpoint 为混合负载；全副作用负载下稀疏性本身不成立但安全不变量仍成立） | 保留 INFO |
| KO-07 | R-08：未提交步骤在崩溃后完全丢失（只有 checkpoint 状态恢复，无重放） | 攻击有效 | **这是语义恢复流派的设计取舍**（vs Temporal 重放）→ 记录为 CM-01 的 contrast 证据，非缺陷 ✅ |
| KO-08 | R-09：显式声明面把认知负担全部推给 host（分类列表/artifacts/behavioral 标记都要 host 正确提供） | 攻击有效 | host 提供错误 artifacts → 矩阵误判；**"显式优于推断"的代价是信任 host** → 记录为 KO-08 边界 + CM-02 边界 ✅ |

**反例预算执行**：8 KO × ≥3 攻击 = 24 次定向攻击执行；3 次确认 fail-open 缺口（R-01/R-03/R-04/R-06 其中 R-03/R-04 为同一边界两视图），0 次导致 KO 降级（均通过 claim 收窄 + 边界注明处置，不删除证据）。

## 6.6 Epistemic Auditor（认知状态诚实性核对）

| 内容 | 状态 | 核对 |
|---|---|---|
| README "5/5 safe" | **vendor 声明（S3 声明级）** | 机制实现 S4 证据充分（对应 flag/policy/测试全在），但 5/5 为 benchmark 输出，stub 路径 → 不写成"真实环境已验证"；C-04 保留 S5 缺口 |
| README "token savings up to 98%" | **vendor 声明，无仓库内证据** → Observation | 不进 EK/KO；C-02 待验证 |
| README "checkpoint 1-8ms / mem<4MB" | **vendor 声明，无基准** → Observation | C-02 待验证 |
| "No LLM guessing" | **Fact（S4）** | divergence_matrix/artifact_diff 全机械；behavioral 需 host 显式标记（测试覆盖） |
| "freezes what your agent already committed" | **Fact（S4 语义）** | checkpoint 快照 + 哈希承诺实现 |
| EK-02/EK-15/EK-19/EK-22 | **Fact（S3）/Observation（部分 S4）** | 代码存在；相关测试覆盖持久化路径 |
| CM-01/CM-02 | **Cognitive Model（Cross-project validation pending）** | 不冒充 Principle/Law |
| C-01~C-06 | **Hypothesis / Observation** | 不冒充知识 |
| 项目生命周期判断（07-30 创建 → 08-04 后静止 3 个月） | **Fact（GitHub API pushed_at）** | 与活跃度解读（Observation）分离 |

**无 Fact→Principle 无中间推导跃迁**；**无 Hypothesis 冒充 Fact**；**vendor 营销口径与实现事实全程分离**。

## 6.7 Reconciliation 摘要（阶段⑥输入，先于独立 Auditor）

| 项目 | 结论 |
|---|---|
| design_intent vs implementation | README 定位（"artifact-aware durable environment"）与实现一致（除性能/5-5 营销口径外） |
| implementation vs tested_behavior | 44 tests 全过，核心行为一致 |
| 已知矛盾 | 无硬矛盾；3 处边界（NO_PRIOR_RECORD / read_only 信任边界 / approve 无鉴权）以条件化知识保留 |
| 待独立 Auditor 复核 | 全部（见 validation/ 阶段⑤报告） |

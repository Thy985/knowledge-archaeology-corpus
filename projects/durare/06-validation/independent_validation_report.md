# Independent Validation Report — ARCH-2026-11-04-001（durare）

> 阶段⑤：独立 Auditor 盲重建。**不把阶段④ package 当事实来源**——独立重读仓库 `src/execution.rs`、`src/postgres.rs`、`src/serialize.rs`、`src/context.rs`、`docs/design/explicit-context.md`、`tests/{crash_consistency,adversity,interop,security}.rs`，建立 Independent Findings 后与 package 对比。
> 禁止修改原考古产物；修正意见经 Reconciliation（阶段⑥）写入 corpus 终稿。

## 1. Blind Reconstruction 范围

| 盲重建项 | 独立重读对象 | 攻击目标 |
|---|---|---|
| B1 | src/execution.rs L60-170 | control-error 通道 / latch / is_storage_failure 六类 |
| B2a | postgres.rs claim_for_recovery (L2472+) | CAS 谓词三分支 |
| B2b | postgres.rs record_step_result / run_transaction_step | CTE 单次往返 / 两层 retry / 重放先行 |
| B3a | serialize.rs 全文 | 格式常量 / 分发解码 / codec |
| B3b | context.rs claim_position 调用点 | 构造时 claim vs poll 时 |
| B4a | tests/crash_consistency.rs | 7 边界 + 精确计数断言真实性 |
| B4b | docs/design/explicit-context.md L90-106 | 官方 SDK 对照表 + upstream commit 引用 |
| B4c | tests/interop.rs + tests/security.rs | golden strings / EVIL sweep |
| B4d | tests/adversity.rs | exactly-once / dedup / 所有权 |
| B4e | context.rs patch/deprecate_patch | poll 时分配例外 |
| B4f | context.rs start_workflow_impl | 序列化失败不耗位置 |

## 2. Independent Findings（盲重建结论，独立于 package）

1. **F-1（execution.rs）**：`check()` 从 `OnceLock` 读失败→`RecoveryRequired`；`record()` 只分类 provider/storage 结果（语义拒绝保持 catchable）；`body_error()` 收养 RecoveryRequired|ObservationFailed 为 interrupt（"Ordinary body errors, including Db, remain business errors"）；`interrupt()` get_or_init + notify_one；`is_storage_failure` 恰六类（Db/Migrate/Serde/Serialization/RecoveryRequired/ObservationFailed）；另有 `observation_error()`（**package 未展开**）：观察者无本地 latch 时把 storage failure 转 ObservationFailed。
2. **F-2（claim_for_recovery）**：单条 UPDATE，CAS 谓词 = PENDING + 原 executor + 原 attempts；三分支 parked/requeued/claimed；rows_affected==0 → Lost；原文 "the predicate, not a transaction, is the arbiter" 属实。
3. **F-3（record_step_result）**：data-modifying CTE（ins INSERT..ON CONFLICT DO NOTHING RETURNING 1 + refresh UPDATE 门控 `EXISTS (SELECT 1 FROM ins)`）+ `SELECT EXISTS(SELECT 1 FROM ins)` 判赢；一次往返属实。
4. **F-4（run_transaction_step）**：重放先行（"a recorded failure is terminal, not a fresh error to be retried"）；OUTER user-attempt 循环 + INNER conflict-attempt 无限循环（冲突不记录）；read_only 独立分支。属实。
5. **F-5（serialize.rs）**：`DBOS_JSON`/`portable_json`/`__DBOS_NIL` 常量；"dispatches on the stored format name"；SerializerCodec trait（name 须异于内建）。属实。
6. **F-6（claim 时机）**：step/transaction/select/send 等在构造时 claim_position 并 `PendingStep::new(position,…)`；`patch`/`deprecate_patch` 走 `poll_in_execution`（poll 时）；`start_workflow_impl` 先序列化输入、成功后 claim（"input serialization failures do not consume a slot"）。与 package 的"大多数构造时 + patch 例外"一致。
7. **F-7（insert_child_workflow）**：**package 未提及**——用显式事务 + `SELECT ... FOR UPDATE` 锁父行 + 重放检查（name 不符→WorkflowConflict）+ status==CANCELLED 检查。这是"DB 仲裁"主题下的**事务+行锁路径**，与单语句 CAS 并存。
8. **F-8（测试断言）**：crash_consistency 7 边界（Entry/InStep(1..3)/AfterStep(1..3)）精确计数（checkpointed==1 / effect-but-no-checkpoint==2 / body 恰 2 次）属实；interop golden strings "pinned identically in the Python, TypeScript, Java, and Go SDK test suites" 属实；security EVIL=`'; DROP TABLE workflow_status; --` hostile_sweep 全 string-typed 输入属实；adversity 的 exactly-once/dedup-唯一赢家/launch-recovery 不偷活+显式 takeover 属实。
9. **F-9（explicit-context 对照表）**：对照表存在且引用 upstream commit `7937413`（dbos-transact-rust），StepBuiltElsewhere/NestedDurableCall/DurableCallCrossedBody/DurableCallOutsideExecution 四行与 package EK-39 一致。

## 3. 判定统计

| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 22 | EK-04/05/06/07/08/09/11/13/14/15/16/17/18/19/20/22/24/26/27/28/32/35/36/37/39/45/46（盲重建逐项命中） |
| PARTIALLY_CONFIRMED | 2 | KO-04（"无锁"表述，见 3.1）；KO-01（patch 例外已在 package 声明，确认存在） |
| DOWNGRADED | 1 | KO-04：L4"数据库即仲裁者（无锁）" → 表述修正为"仲裁下沉到数据库，形式因操作而异（单语句 CAS 或事务+行锁）" |
| OVER_GENERALIZED | 0 | L4/L5 均标注 durare-originated · Cross-project validation pending |
| MISSING | 2 | observation_error()（F-1）；insert_child_workflow 的 FOR UPDATE 路径（F-7） |
| CONTRADICTED | 0 | （6.4 已解决的两个 contradiction 盲重建复核通过） |
| NEEDS_HUMAN_REVIEW | 1 | C-02（AdminServer 无认证 by design） |

### 3.1 KO-04 修正细节（Independent vs Package）

- **Package 表述**：KO-04 claim "多进程 executor fleet 的所有权仲裁不靠分布式锁、不靠事务序列化，靠单条原子语句……"
- **盲重建发现**：`insert_child_workflow`（postgres.rs）用 `SELECT ... FOR UPDATE` 锁父行 + 事务，是**事务+行锁**仲裁路径；`dequeue_workflows` 用条件更新（ENQUEUED→PENDING）。
- **修正**：仲裁机制是"按操作选择的数据库原子原语族"——单语句 CAS（claim/checkpoint）与事务+行锁（child 插入）并存，共同点是**仲裁决定权在数据库而非应用锁**。KO-04 保留 L4 但重写 claim 前半句。

## 4. 3 成功 / 3 错误（对照说明）

**3 成功（盲重建独立确认的关键论断）**：
1. control-error 通道与 latch 幂等（F-1）——与 package EK-04/06/13 完全一致，包括"body catch 控制错误后下一个 durable call 仍被拒"的机制（check() 读同一 OnceLock）。
2. claim_for_recovery 的"predicate, not transaction, is the arbiter"（F-2）——原文存在，非转述失真。
3. crash_consistency 7 边界精确计数（F-8）——不是宽松 "at least once"，断言为 `assert_eq!(count==1/==2)` 精确值。

**3 错误/遗漏（package 需修正）**：
1. **KO-04 表述过度**：宣称"不靠事务序列化"但仓库存在事务+FOR UPDATE 路径（F-7）→ 降级为"原子原语族（含事务+行锁）"。
2. **observation_error() 遗漏**（F-1）：观察者无 latch 路径的 control-error 规范未进 EK/Flow。
3. **insert_child_workflow 遗漏**（F-7）：child 插入的"事务+锁+重放检查"模式未进 Flow Atlas 的 Authority/Control 流。

## 5. 遗漏扫描（Coverage Auditor 视角）

- bypass/override/exception：`patch`/`deprecate_patch` poll 时分配 ✓（已进 EK-01 例外）；`insert_child_workflow` 事务路径 ✗（补入）
- alternate/direct call：`Client` 直连 provider（无 engine）→ `observation_error` 路径 ✗（补入）
- admin path：AdminServer 无认证 ✓（C-02 已标记）；`start_on((LOCALHOST,…))` ✓
- fallback/legacy：`warn_on_legacy_empty_app_version`（`application_version=''`）✓；`application_version IS NULL` 兼容 ✓
- 失败路径：`MAX_RECOVERY_ATTEMPTS_EXCEEDED` park ✓；`WorkflowConflict` 第二写者 ✓；`NonexistentWorkflow`/`QueueDeduplicated`/`WorkflowTargetConflict` ✓

## 6. Benchmark case（质量锚）

盲重建对 KO-01 的复现声明（select! 随机 poll 顺序可调换 position）无法在本环境运行验证（需要真实运行 select! + crash-recover），标记为**文档声明（S2-S3 级）**——package EK-01 的 S4 标注应降级为 S3+S4（文档复现记录 + 测试存在但盲重建未运行），或标注"测试存在，未经本环境复现"（package 06 已统一声明证据上限 S4，此处确认合规）。

## 7. 结论

盲重建总体强确认：核心机制（seam/control-error/CAS/CTE/格式分发/精确测试）全部属实。3 处修正（KO-04 表述、observation_error、insert_child_workflow）不影响 10 个 KO 的成立性，属于**scope/表述收紧**而非推翻。禁止修改原考古产物；修正意见移交 Reconciliation。

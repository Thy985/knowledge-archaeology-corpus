# 02 Engineering Knowledge — durare（EK Graph，宽底座）

> 50 条 EK。每条含：层级（L1 工程事实 / L2 工程知识）、类别、证据（可溯源文件:符号）、links（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）、状态（epistemic status）。
> 证据上限 S3（已实现）/S4（测试断言存在，未在本环境运行）。D 级 = 项目局部知识（保留，不删除）。
> EK Graph 质量指标见文末。

---

## EK-01 position 构造时保留（claim at construction）
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: A | **strength**: S4
- **claim**: 0.5.0 起，大多数 durable call（step/transaction/send/recv/set_event 等）在**构造时** claim checkpoint position；`PendingStep` 返回后 `#[must_use]`，drop 已成功构造的 call 位置照常消耗。
- **evidence**: `src/context.rs` claim! 宏（L164-171）+ `PendingStep` 结构（L3245-3251）；`docs/design/explicit-context.md` "Most durable calls now return PendingStep and reserve a position during construction"；CHANGELOG [0.5.0] "A durable call built and never awaited now spends its position"
- **links**: [{to: EK-02, type: causal, note: "构造时 claim → task-local 作用域才能检查放置"}, {to: EK-08, type: dependency, note: "position 唯一性依赖 operation_outputs 唯一约束"}, {to: EK-03, type: mechanism, note: "同为 position 纪律机制"}]
- **status**: Validated Pattern（文档 + 测试 double-source）

## EK-02 task-local 执行作用域（CURRENT_EXECUTION）
- **layer**: L1 | **category**: ENGINEERING / IMPLEMENTATION | **value**: A | **strength**: S4
- **claim**: `Execution` 通过 `tokio::task_local!` 的 `CURRENT_EXECUTION` 建立 workflow 执行作用域；`scope()` 包装 invocation 和每次 poll，返回/panic 时恢复前一 execution；spawned task 不继承该作用域。
- **evidence**: `src/execution.rs` L31-58（task_local! + scope）；explicit-context.md "A spawned task does not inherit that scope"
- **links**: [{to: EK-01, type: causal, note: "claim 时检查作用域"}, {to: EK-03, type: mechanism, note: "同为放置检查机制"}]
- **status**: Validated Pattern

## EK-03 放置检查三重拒绝（NestedDurableCall / DurableCallCrossedBody / DurableCallOutsideExecution）
- **layer**: L1 | **category**: ENGINEERING / FAILURE | **value**: A | **strength**: S4
- **claim**: body 内构造 PendingStep → `NestedDurableCall`（不占位）；prebuilt call 在 body 内 poll → `DurableCallCrossedBody`（已占位保留）；spawned task/另一 execution 内使用 → `DurableCallOutsideExecution`（构造不占位，poll 保留已占位）。每次 poll 前检查放置。
- **evidence**: `src/execution.rs` check_placement（L60-79）；ErrorCode NestedDurableCall/DurableCallOutsideExecution（error.rs L50-53）；explicit-context.md "Constructing a PendingStep there returns DurableCallOutsideExecution without claiming a position"
- **links**: [{to: EK-02, type: dependency, note: "依赖 task-local 作用域实现"}, {to: EK-09, type: causal, note: "放置拒绝 → 记录式编程错误绕过 retry"}, {to: EK-39, type: contrast, note: "与官方 SDK 放置规则对照"}]
- **status**: Validated Pattern

## EK-04 control-error 二型通道（RecoveryRequired / ObservationFailed）
- **layer**: L1 | **category**: ENGINEERING / FAILURE | **value**: A | **strength**: S4
- **claim**: 跨越 seam（execution.rs）向上的两条控制错误：`RecoveryRequired`（本进程无法继续，无记录，行保持 PENDING，body 后续 durable call 被 execution latch 拒绝）与 `ObservationFailed`（操作结果无法观察，停止但不调度恢复——丢失响应≠操作未发生）。
- **evidence**: `src/execution.rs` L5-15 + docs/design/execution-layers.md "The control-error channel"
- **links**: [{to: EK-05, type: causal, note: "分类来自 provider 边界"}, {to: EK-06, type: mechanism, note: "同为 Execution 内部机制"}, {to: EK-13, type: constraint, note: "控制错误不可被 body 捕获分支"}]
- **status**: Validated Pattern

## EK-05 is_storage_failure 分类边界（provider boundary error contract）
- **layer**: L2 | **category**: ENGINEERING / DECISION | **value**: A | **strength**: S3
- **claim**: 只有 Db/Migrate/Serde/Serialization/RecoveryRequired/ObservationFailed 六类被认定为 storage failure（升为控制错误）；应用代码返回的 Db/Serde 是业务结果。分类是 provider 边界契约，不是全局分类。
- **evidence**: `src/execution.rs` is_storage_failure（L155-165）；provider.rs trait 文档 "the provider error contract separates storage faults from semantic rejections"
- **links**: [{to: EK-04, type: causal, note: "分类决定走哪个通道"}, {to: EK-11, type: constraint, note: "语义拒绝保持普通 catchable 错误"}]
- **status**: Fact（实现存在，注释阐明意图）

## EK-06 Execution latch 幂等中断（OnceLock + Notify）
- **layer**: L1 | **category**: ENGINEERING / IMPLEMENTATION | **value**: B | **strength**: S3
- **claim**: `Execution` 内部 `OnceLock<Arc<Error>>` 记录首个中断原因；`interrupt()` 幂等（首次记录 + notify）；`failed()` 循环等待 Notify 直到 check 失败。
- **evidence**: `src/execution.rs` Inner{failure: OnceLock, changed: Notify}（L16-22）；interrupt/failed（L95-120）
- **links**: [{to: EK-04, type: mechanism, note: "latch 实现"}, {to: EK-13, type: constraint, note: "body 后续 durable call 被 latch 拒绝"}]
- **status**: Fact

## EK-07 record_step_result 单次往返 CTE（checkpoint insert + executor refresh）
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: A | **strength**: S3
- **claim**: Postgres 下成功 checkpoint 用 data-modifying CTE：`WITH ins AS (INSERT ... ON CONFLICT DO NOTHING RETURNING 1), refresh AS (UPDATE workflow_status SET executor_id ... WHERE EXISTS ins)` 单条 SQL 完成 insert + executor refresh；赢家=ins 存在。
- **evidence**: `src/postgres.rs` record_step_result（L1262-1340）；注释 "One round trip covers the whole common case"
- **links**: [{to: EK-08, type: mechanism, note: "同为 ON CONFLICT 仲裁"}, {to: EK-22, type: causal, note: "赢家证明 executor 真在跑 → refresh"}]
- **status**: Fact（实现存在）

## EK-08 ON CONFLICT DO NOTHING 写仲裁（one checkpoint per position）
- **layer**: L2 | **category**: ENGINEERING / MECHANISM | **value**: A | **strength**: S4
- **claim**: operation_outputs 唯一约束 (workflow_uuid, function_id) + `ON CONFLICT DO NOTHING`：同一 position 第二个 writer 的 insert 是 no-op（rows_affected=0 → WorkflowConflict）。workflow_status insert 用 `ON CONFLICT DO NOTHING`（created flag 仲裁直接 start 去重）。
- **evidence**: `src/postgres.rs` record_step_result / insert_workflow_status；provider.rs dedup_or 注释 "the inserts use ON CONFLICT DO NOTHING"；execution-layers.md "One checkpoint per position"
- **links**: [{to: EK-07, type: mechanism, note: "同一机制"}, {to: EK-18, type: causal, note: "唯一约束 → WorkflowConflict 停止"}, {to: EK-20, type: dependency, note: "供 dequeue 幂等 claim 依赖"}]
- **status**: Validated Pattern

## EK-09 claim_for_recovery CAS（predicate, not transaction, is the arbiter）
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: A | **strength**: S3
- **claim**: 恢复认领用单条 UPDATE 的 CAS 谓词（status='PENDING' AND executor_id=expected AND recovery_attempts=expected）；rows_affected=0 → Lost。三分支：超 cap → park（MAX_RECOVERY_ATTEMPTS_EXCEEDED）；requeue → ENQUEUED（由 dispatcher 自己的原子 claim 收）；否则 direct claim 换 executor。
- **evidence**: `src/postgres.rs` claim_for_recovery（L2472-2540）"Every arm carries the same compare-and-set predicate ... the predicate, not a transaction, is the arbiter"
- **links**: [{to: EK-08, type: mechanism, note: "同为 DB 原子仲裁"}, {to: EK-15, type: causal, note: "恢复认领 → 换 executor 继续"}, {to: EK-23, type: constraint, note: "max_attempts 计数含恢复认领"}]
- **status**: Fact

## EK-10 事务步骤同事务提交（SQL 写 + checkpoint 原子）
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: A | **strength**: S4
- **claim**: `transaction` 步骤把用户 SQL 写与 operation_outputs checkpoint 放进**同一数据库事务**：要么一起 commit，要么一起 rollback——消除普通 step 的 at-least-once window（对写数据库的效果）。
- **evidence**: `src/transactions.rs` L1-30 + `src/tx.rs` L1-12；`src/postgres.rs` run_transaction_step 成功分支 "checkpoint the output in the same transaction"
- **links**: [{to: EK-11, type: causal, note: "同事务 → 冲突处理内层循环"}, {to: EK-12, type: contrast, note: "read-only 退化对照"}, {to: EK-30, type: contrast, note: "普通 step 是先执行后 checkpoint"}]
- **status**: Validated Pattern

## EK-11 事务两层 retry（内层冲突无限重试 + 外层用户重试）
- **layer**: L2 | **category**: ENGINEERING / MECHANISM | **value**: B | **strength**: S3
- **claim**: run_transaction_step 两层循环：内层（conflict_attempt）处理 serialization/deadlock/transient DB 错误——abort tx 换新 tx 无限重试（匹配 Go/Python，冲突不记录）；外层（user_attempt）执行用户 retry 策略（max_retries/retry_if），body 错误才计入预算。记录过的失败是终态，重放直接返回（"a recorded failure is terminal, not a fresh error to be retried"）。
- **evidence**: `src/postgres.rs` run_transaction_step（L1395-1445）
- **links**: [{to: EK-10, type: causal, note: "同事务前提下的 retry 结构"}, {to: EK-32, type: mechanism, note: "失败重放不重试与 EK-32 同纪律"}]
- **status**: Fact

## EK-12 read-only 事务退化为普通 step checkpoint
- **layer**: L1 | **category**: ENGINEERING / DECISION | **value**: C | **strength**: S3
- **claim**: read-only 事务先 commit 读（释放快照），再普通 step 风格 checkpoint——纯读 at-least-once 无害；也规避 Postgres 对 in-transaction insert 的 25006 拒绝。
- **evidence**: `src/postgres.rs` run_transaction_step read_only 分支（L1452-1465）
- **links**: [{to: EK-10, type: contrast, note: "同主题两种策略"}, {to: EK-07, type: dependency, note: "复用 record_step_result"}]
- **status**: Fact

## EK-13 控制错误不可被 body 捕获分支（latch 持续拒绝）
- **layer**: L2 | **category**: ENGINEERING / FAILURE | **value**: A | **strength**: S3
- **claim**: body 若 catch 控制错误并继续，其下一个 durable call 仍会被 execution latch 拒绝（check() 查 failure OnceLock）——"A body that catches a control error cannot branch on it: its next durable call is refused"。`body_error()` 收养来自 child handle 的另一 execution 的控制信号。
- **evidence**: `src/execution.rs` check/body_error（L82-110）；execution-layers.md
- **links**: [{to: EK-04, type: constraint, note: "对控制错误的不可消化性"}, {to: EK-06, type: dependency, note: "依赖 latch 实现"}, {to: EK-33, type: mechanism, note: "与错误分类纪律同主题"}]
- **status**: Fact

## EK-14 动态 SQL 不变量（bind parameter / 编译期常量插值）
- **layer**: L1 | **category**: ENGINEERING / SECURITY | **value**: A | **strength**: S4
- **claim**: 无 caller 字符串进入 SQL 文本。运行时值全走 bind parameter；插值片段仅编译期常量（列名/方言关键字/按 enum/flag 选的 `&'static str`）；唯一配置来源标识符（Postgres system schema）按 `[A-Za-z_][A-Za-z0-9_]*` 校验。外部强制：tests/security.rs hostile sweep。
- **evidence**: `src/security.rs` "The dynamic-SQL invariant"；`src/postgres.rs` claim_for_recovery 的 format! 只用编译期常量 + bind
- **links**: [{to: EK-34, type: causal, note: "不变量由外部测试强制"}, {to: EK-26, type: constraint, note: "schema 标识符校验是唯一例外"}]
- **status**: Validated Pattern

## EK-15 恢复 backoff + park（100ms doubling capped 5s / MAX_RECOVERY_ATTEMPTS_EXCEEDED）
- **layer**: L1 | **category**: ENGINEERING / FAILURE | **value**: B | **strength**: S3
- **claim**: 被 RecoveryRequired 中断的执行由 settlement 以 backoff（100ms 倍增封顶 5s）重认领，至 recovery-attempt cap 后 park 为 MAX_RECOVERY_ATTEMPTS_EXCEEDED（死信，需显式 resume）。
- **evidence**: docs/design/execution-layers.md；provider.rs STATUS_MAX_RECOVERY_ATTEMPTS_EXCEEDED；CHANGELOG 0.5.0 "settlement"
- **links**: [{to: EK-09, type: causal, note: "park 由 claim_for_recovery 执行"}, {to: EK-18, type: mechanism, note: "同 failed 停止语义"}]
- **status**: Fact

## EK-16 app_version = sha-256 of executable（默认版本门控）
- **layer**: L1 | **category**: ENGINEERING / CONFIG | **value**: B | **strength**: S3
- **claim**: 默认 application version = 运行可执行文件 sha-256：不同构建得不同版本，版本门控的 dispatch/recovery 据此工作；固定 app_version 的部署在 0.4→0.5 需先 drain。
- **evidence**: Cargo.toml sha2 依赖注释；CHANGELOG 0.5.0 "app_version hashes the executable"
- **links**: [{to: EK-17, type: causal, note: "版本值驱动 dequeue gate"}, {to: EK-21, type: mechanism, note: "同为版本安全机制"}]
- **status**: Fact

## EK-17 dequeue version gate（= $n OR IS NULL）+ legacy '' 警告
- **layer**: L1 | **category**: ENGINEERING / CONFIG | **value**: B | **strength**: S3
- **claim**: dequeue 匹配 `application_version = $n OR IS NULL`（与其他 SDK 一致）；legacy `''` 行无人认领会永久 ENQUEUED——启动时 best-effort 统计并警告（不致命）。queue 注册冲突策略：仅 latest 版本可 overwrite 已有 queue 行（防滚动部署中旧 executor 覆盖新配置）。
- **evidence**: `src/postgres.rs` warn_on_legacy_empty_app_version（L532-560）+ dequeue 谓词；engine.rs launch queue 冲突策略
- **links**: [{to: EK-16, type: causal, note: "版本来源"}, {to: EK-22, type: constraint, note: "版本门控约束跨版本 dispatch"}]
- **status**: Fact

## EK-18 INTERNAL_QUEUE 100ms polling（重执行/恢复直连 run 路由）
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: B | **strength**: S3
- **claim**: 引擎始终 dispatch `_dbos_internal_queue`（100ms poll）：Client 与引擎自身把无用户队列的 re-executed work（resume/fork of direct run）路由至此，任何 live engine claim 并运行——而非依赖重启的 recovery。
- **evidence**: `src/engine.rs` INTERNAL_QUEUE 常量 + internal_queue()（L46-55）
- **links**: [{to: EK-27, type: causal, note: "resume/fork 无队列路由"}, {to: EK-28, type: mechanism, note: "同为 runtime 编排机制"}]
- **status**: Fact

## EK-19 SCHEDULE_RECONCILE_INTERVAL 500ms（持久调度 reconciler）
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: B | **strength**: S3
- **claim**: 每 500ms reconciler 列出持久化 schedules 并安装/退役 per-schedule firing loop；schedule 状态存 `workflow_schedules` 表；运行中 loop 在 reconciliation 时拾取定义变更。
- **evidence**: `src/engine.rs` SCHEDULE_RECONCILE_INTERVAL（L44）；`src/schedule.rs`；CHANGELOG [Unreleased] ScheduleUpdate
- **links**: [{to: EK-25, type: causal, note: "reconciler 安装 firing loops"}, {to: EK-24, type: mechanism, note: "同为后台循环机制"}]
- **status**: Fact

## EK-20 队列 dedup（queue-scoped dedup index / dedup_or）
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: B | **strength**: S4
- **claim**: 入队唯一约束（queue_name+dedup_id）冲突 → `Error::queue_deduplicated`（unique violation 映射）；PK 冲突永不至此（inserts 用 ON CONFLICT DO NOTHING）。dedup 支持 Reject 策略（checkpointed 拒绝：replay 保留已 catch 的拒绝）。
- **evidence**: `src/provider.rs` dedup_or（L18-30）+ 注释；CHANGELOG 0.5.0 "Reject deduplication ... are checkpointed"
- **links**: [{to: EK-08, type: dependency, note: "依赖唯一约束仲裁"}, {to: EK-31, type: causal, note: "队列 dedup → 枚举候选"}]
- **status**: Validated Pattern

## EK-21 check_target / check_child_target（WorkflowTargetConflict）
- **layer**: L1 | **category**: ENGINEERING / FAILURE | **value**: C | **strength**: S3
- **claim**: dedup key/child 目标若指向已存在但 name/class_name/config_name 不同的 workflow → `WorkflowTargetConflict`（"a deduplication key must not silently adopt another workflow's output"）。
- **evidence**: `src/provider.rs` check_target/check_child_target（L33-60）
- **links**: [{to: EK-20, type: causal, note: "dedup 冲突后的目标校验"}, {to: EK-27, type: mechanism, note: "同为语义拒绝"}]
- **status**: Fact

## EK-22 恢复 dispatch 不 inline（task tracker spawn）
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: B | **strength**: S3
- **claim**: `recover()` 把每个恢复 run spawn 到 engine 的 task tracker 后立即返回——park 在 recv/durable timer 的 run 不阻塞 caller 或后续 run。recovers 只处理本 app version 的 PENDING（版本门控），超限 park。
- **evidence**: `src/engine.rs` recover（L2710-2720）+ recover_pending_workflows（L2750+）
- **links**: [{to: EK-23, type: causal, note: "恢复 → 重认领 → dispatch"}, {to: EK-24, type: dependency, note: "依赖 TaskTracker 计数"}]
- **status**: Fact

## EK-23 TaskTracker spawn 瞬间计数（无单独 guard）
- **layer**: L1 | **category**: ENGINEERING / IMPLEMENTATION | **value**: C | **strength**: S3
- **claim**: run 通过 tracker spawn 即被计数（从创建瞬间开始），无需单独 guard 保持同步；shutdown 靠 tracker drain。
- **evidence**: `src/engine.rs` DurableEngine.tasks 字段注释（L724-729）
- **links**: [{to: EK-22, type: dependency, note: "恢复 dispatch 依赖"}, {to: EK-29, type: mechanism, note: "同 shutdown 排水机制"}]
- **status**: Fact

## EK-24 七边界崩溃一致性测试（精确 effect 计数）
- **layer**: L1 | **category**: TESTING / METHODOLOGY | **value**: A | **strength**: S4
- **claim**: crash_consistency.rs 在 7 个崩溃边界（Entry / InStep(i) / AfterStep(i)）park workflow，drop engine，第二 engine 接管恢复；每边界断言**精确** effect 计数：checkpoint 已提交的 step 永不重跑（count=1）、effect 已跑但 checkpoint 未提交的 step 重跑（count=2）——at-least-once window 恰好一步宽；body 进入恰 2 次（崩溃执行 + 恢复重放）。
- **evidence**: `tests/crash_consistency.rs` L1-35（BOUNDARIES）+ 断言注释；run on SQLite unconditional + Postgres with DATABASE_URL
- **links**: [{to: EK-30, type: mechanism, note: "同为测试方法论"}, {to: EK-10, type: causal, note: "验证普通 step 的 window"}, {to: EK-32, type: causal, note: "失败重放同纪律"}]
- **status**: Validated Pattern（测试存在且断言精确）

## EK-25 contract(db) 三后端同跑模式
- **layer**: L1 | **category**: TESTING / METHODOLOGY | **value**: B | **strength**: S3
- **claim**: 触碰存储的行为写一个 `contract(db)` 并在 memory/SQLite/Postgres 三后端各跑一遍（AGENTS.md 强制；tests/child_atomic.rs 示范）。
- **evidence**: AGENTS.md "Write one contract(db) and run it on memory, SQLite and Postgres"; tests/child_atomic.rs
- **links**: [{to: EK-30, type: mechanism, note: "同测试方法论"}, {to: EK-38, type: constraint, note: "每后端行为一致性约束"}]
- **status**: Fact（规范 + 示范测试存在）

## EK-26 多 executor 对抗测试（exactly-once / dedup 唯一赢家 / 所有权）
- **layer**: L1 | **category**: TESTING / METHODOLOGY | **value**: A | **strength**: S4
- **claim**: adversity.rs 用多个独立 executor_id + 独立连接池的 engine 竞争同一 Postgres：断言队列 workflow 无论多少 dispatcher 竞争恰好跑一次；dedup 恰一个赢家、输家拿类型化错误；launch-recovery 不偷另一 live executor 的 pending work，而显式 takeover 可以。
- **evidence**: `tests/adversity.rs` L1-30（fleet_engine + 断言注释）
- **links**: [{to: EK-20, type: causal, note: "验证队列 dedup 保证"}, {to: EK-09, type: causal, note: "验证 CAS claim 所有权"}, {to: EK-30, type: mechanism, note: "同测试方法论"}]
- **status**: Validated Pattern

## EK-27 跨 SDK 互操作 golden strings（byte-identical）
- **layer**: L1 | **category**: TESTING / INTEROP | **value**: A | **strength**: S4
- **claim**: interop.rs 钉住 golden JSON strings（GOLDEN_OUTPUT_JSON 等）——"pinned identically in the Python, TypeScript, Java, and Go SDK test suites"，portable 模式序列化必须逐字节复现。
- **evidence**: `tests/interop.rs` L1-40（GOLDEN_* 常量 + 注释）
- **links**: [{to: EK-35, type: causal, note: "验证 portable_json 契约"}, {to: EK-36, type: mechanism, note: "同跨 SDK 一致性机制"}]
- **status**: Validated Pattern

## EK-28 security hostile sweep（EVIL 注入全部 string-typed input）
- **layer**: L1 | **category**: TESTING / SECURITY | **value**: A | **strength**: S4
- **claim**: tests/security.rs 把 `'; DROP TABLE workflow_status; --` 经所有字符串型公开输入（workflow id/name、queue、dedup/partition key、topic、event/stream key、step name、schedule name、list-filter 字段）推入 SQLite+Postgres，断言 verbatim round-trip 且系统表完整。
- **evidence**: `tests/security.rs` L1-35（EVIL 常量 + hostile_sweep）
- **links**: [{to: EK-14, type: causal, note: "外部强制动态 SQL 不变量"}, {to: EK-36, type: mechanism, note: "同外部强制纪律"}]
- **status**: Validated Pattern

## EK-29 shutdown 双信号（CancellationToken + TaskTracker drain）
- **layer**: L1 | **category**: ENGINEERING / IMPLEMENTATION | **value**: C | **strength**: S3
- **claim**: shutdown(timeout) 用 CancellationToken 停后台 loops + TaskTracker drain in-flight；launch 在 shutdown 后安装新 token（cancelled token 不可 reset）+ reopen tracker。
- **evidence**: `src/engine.rs` launch（L1520-1540）+ shutdown（L1903）+ DurableEngine 字段
- **links**: [{to: EK-23, type: mechanism, note: "同 tracker 机制"}, {to: EK-24, type: dependency, note: "drain 依赖计数"}]
- **status**: Fact

## EK-30 测试方法论主题簇（精确断言替代宽松断言）
- **layer**: L2 | **category**: TESTING / METHODOLOGY | **value**: A | **strength**: S4
- **claim**: 项目测试纪律：不用 "at least once" 断言，用精确计数（crash_consistency 的 count==1/==2、adversity 的 exactly-once、interop 的 byte-identical）；"a report with recorded: 0 passes in the same sense an empty test suite is green"（replay.rs 的 vacuous-pass 防伪）。
- **evidence**: tests/crash_consistency.rs 断言注释；tests/adversity.rs；`src/replay.rs` ReplayReport 文档
- **links**: [{to: EK-24, type: mechanism, note: "崩溃测试实例"}, {to: EK-26, type: mechanism, note: "对抗测试实例"}, {to: EK-27, type: mechanism, note: "互操作测试实例"}]
- **status**: Validated Pattern（跨文件一致）

## EK-31 队列候选枚举（queues/debounce 的 Queue 结构）
- **layer**: L1 | **category**: ENGINEERING / IMPLEMENTATION | **value**: C | **strength**: S3
- **claim**: WorkflowQueue 含 base_polling_interval/rate_limiter/partition key 等配置；enqueue 持久化 ENQUEUED 行由 dispatcher claim；队列注册在 launch 时持久化到共享 queues 表（fleet 注册表）。
- **evidence**: `src/queue.rs` + `src/queues.rs` + engine.rs register_queue/launch
- **links**: [{to: EK-20, type: subsystem, note: "同队列子系统"}, {to: EK-18, type: subsystem, note: "内部队列是特殊队列"}]
- **status**: Fact

## EK-32 失败记录重放为同一错误（不重试 flaky step）
- **layer**: L1 | **category**: ENGINEERING / FAILURE | **value**: A | **strength**: S4
- **claim**: 记录过的失败（ERROR 输出在 operation_outputs）在 replay 时原样返回同一错误——"a step that failed stays failed ... a replay returns the same error rather than giving a flaky step a second chance"。记录失败是终态，重放不重估 retry 策略。
- **evidence**: `src/durability.rs` "A step that failed stays failed"；postgres.rs run_transaction_step 重放分支
- **links**: [{to: EK-11, type: mechanism, note: "同失败终态纪律"}, {to: EK-13, type: mechanism, note: "同错误不可消化纪律"}]
- **status**: Validated Pattern

## EK-33 版本化错误信封（RecordedError / PREFIX / remote derive）
- **layer**: L1 | **category**: ENGINEERING / IMPLEMENTATION | **value**: B | **strength**: S3
- **claim**: recorded_error.rs 用 serde remote derive 把 Error 序列化为 tag/content（kind/data）持久化；新写应用消息转义 `__DURARE_ERROR__:` 前缀；旧 reader 无法恢复新类型（滚动升级规则：先升级 reader 再 writer）。
- **evidence**: `src/recorded_error.rs`（PREFIX/NAME/ErrorWire）
- **links**: [{to: EK-16, type: causal, note: "持久化错误格式随版本演化"}, {to: EK-36, type: mechanism, note: "同跨版本 wire 契约"}]
- **status**: Fact

## EK-34 外部强制纪律（security/interop 均"enforced from the outside"）
- **layer**: L2 | **category**: ENGINEERING / DECISION | **value**: B | **strength**: S3
- **claim**: 项目偏好"从外部强制不变量而非钉住源码"：security.rs "enforced from the outside rather than by pinning the source"；interop.rs golden strings 若源码 refactor 会静默失效（注释明说）。配套：每后端跑一遍。
- **evidence**: `tests/security.rs` L1-15；`tests/interop.rs` L10-15
- **links**: [{to: EK-14, type: causal, note: "动态 SQL 的外部强制"}, {to: EK-27, type: causal, note: "interop 的外部强制"}, {to: EK-35, type: dependency, note: "依赖 golden 常量为契约"}]
- **status**: Observation（两文件注释一致）

## EK-35 portable_json 跨 SDK wire 契约（serde_json preserve_order）
- **layer**: L1 | **category**: ENGINEERING / INTEROP | **value**: A | **strength**: S4
- **claim**: `Serializer::Portable` 写 plain JSON，`serialization` 列记 `portable_json`；serde_json `preserve_order` feature 保证 key 顺序 = struct 字段顺序 → 跨 SDK 字节一致。解码永远按存储的格式名分发（format-directed decoding），未知格式报错而非静默损坏。
- **evidence**: `src/serialize.rs` L1-30（PORTABLE/DBOS_JSON 常量 + 表）；Cargo.toml serde_json preserve_order 注释
- **links**: [{to: EK-27, type: causal, note: "golden strings 验证该契约"}, {to: EK-36, type: mechanism, note: "同跨 SDK 机制"}, {to: EK-37, type: dependency, note: "依赖 serialization 列"}]
- **status**: Validated Pattern

## EK-36 跨 SDK on-disk 共享标识（时间戳 BIGINT / sentinel / roles JSON）
- **layer**: L1 | **category**: ENGINEERING / INTEROP | **value**: B | **strength**: S3
- **claim**: 跨 SDK 字节一致的三项约定：①时间戳全存 BIGINT epoch ms（非 SQL timestamp）；②`STREAM_CLOSED_SENTINEL="__DBOS_STREAM_CLOSED__"` verbatim 存储（用户值经 serializer 编码不会碰撞）；③`authenticated_roles` 单列 JSON 数组（encode_roles/decode_roles）。
- **evidence**: docs/schema.md "every timestamp is stored as milliseconds since the epoch in a plain BIGINT"；provider.rs STREAM_CLOSED_SENTINEL + encode_roles/decode_roles
- **links**: [{to: EK-35, type: mechanism, note: "同跨 SDK 字节契约"}, {to: EK-37, type: dependency, note: "roles 存 serialization 同列族"}]
- **status**: Validated Pattern

## EK-37 serialization 列格式分发解码
- **layer**: L1 | **category**: ENGINEERING / INTEROP | **value**: B | **strength**: S3
- **claim**: 每个持久化值（input/output/step/message/event/stream）存 TEXT + `serialization` 格式名；解码按存储名分发（支持 DBOS_JSON / portable_json / custom codec），非按"当前配置"。
- **evidence**: `src/serialize.rs` L1-25；workflow_status/operation_outputs schema 均有 serialization 列
- **links**: [{to: EK-35, type: dependency, note: "portable 依赖此机制"}, {to: EK-33, type: mechanism, note: "同持久化格式路由"}]
- **status**: Fact

## EK-38 语义对齐纪律（match reference SDKs, cite the file）
- **layer**: L2 | **category**: ENGINEERING / GOVERNANCE | **value**: A | **strength**: S3
- **claim**: AGENTS.md 强制："Match the reference DBOS SDKs (Go, Python) on semantics, not only on names. Before stating 'the reference SDKs do X', read the code on the dbos-inc upstream main branch and cite the file." 无参考行为时先写下规则（condition → durable behavior）再实现。
- **evidence**: AGENTS.md "Conventions"
- **links**: [{to: EK-39, type: causal, note: "语义对齐 → 与官方 Rust SDK 对比表"}, {to: EK-27, type: dependency, note: "互操作测试支撑对齐"}, {to: EK-44, type: mechanism, note: "同治理纪律"}]
- **status**: Fact（规范存在）

## EK-39 explicit context vs task-local（官方 Rust SDK 对照）
- **layer**: L2 | **category**: ENGINEERING / DECISION | **value**: A | **strength**: S4
- **claim**: durare 显式 context 参数 vs 官方 dbos-transact-rust task-local state；step 放置规则差异表（spawned task 内构造：durare 拒绝不占位 vs 官方无 checkpoint 运行会重复；body 内构造：durare 拒绝 vs 官方 plain work；body 内 poll prebuilt：DurableCallCrossedBody vs StepBuiltElsewhere；另一 execution poll：DurableCallOutsideExecution vs StepBuiltElsewhere）——已对 upstream commit `7937413` 验证。
- **evidence**: docs/design/explicit-context.md "Comparison with DBOS's Rust SDK"（含对比表）
- **links**: [{to: EK-01, type: causal, note: "显式 context 使放置检查可行"}, {to: EK-03, type: contrast, note: "与官方放置规则对照"}, {to: EK-38, type: causal, note: "来自语义对齐纪律"}]
- **status**: Validated Pattern（文档 + 上游 commit 引用）

## EK-40 身份继承（AuthContext 贯穿 workflow/children/trace/row）
- **layer**: L1 | **category**: ENGINEERING / SECURITY | **value**: B | **strength**: S3
- **claim**: AuthContext（authenticated_user/assumed_role/authenticated_roles）在 start 时设置，流入 workflow、children（identity inheritance per field）、trace spans、持久化行（workflow_status 三列）。
- **evidence**: `src/security.rs` "Authorization"；`src/context.rs` enqueue_workflow 文档 "inherits identity per field"
- **links**: [{to: EK-41, type: causal, note: "身份供 require_roles 检查"}, {to: EK-36, type: dependency, note: "roles 持久化编码"}]
- **status**: Fact

## EK-41 require_roles 前置检查 + launch 拒未注册声明
- **layer**: L1 | **category**: ENGINEERING / SECURITY | **value**: B | **strength**: S3
- **claim**: `require_roles(name, roles)` 声明式角色门槛，engine 在 start 前检查；launch 时对未注册 workflow 的 required_roles 声明直接报错（"a typo that would silently enforce nothing"）。
- **evidence**: `src/engine.rs` launch（L1505-1515）；security.rs Authorization 段
- **links**: [{to: EK-40, type: causal, note: "消费 AuthContext"}, {to: EK-14, type: mechanism, note: "同安全不变量纪律"}]
- **status**: Fact

## EK-42 debounce 完全建在既有 durable 原语上
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: B | **strength**: S3
- **claim**: Debouncer 无新机制：每 key 一个内部 workflow（dedup id = key 保证唯一）collect pushed-back inputs over recv，最终 start 目标一次；producer 用 send + event ACK nudge。注册名 `_dbos_debouncer` 自动注册。
- **evidence**: `src/debounce.rs` L1-30（DEBOUNCER_WF + 原理注释）
- **links**: [{to: EK-20, type: dependency, note: "依赖队列 dedup"}, {to: EK-18, type: dependency, note: "走内部队列"}, {to: EK-43, type: mechanism, note: "同为高层原语复用底层"}]
- **status**: Fact

## EK-43 Client 无 registry 出站控制面（enqueue/observe 分离）
- **layer**: L1 | **category**: ENGINEERING / ARCHITECTURE | **value**: B | **strength**: S3
- **claim**: `Client` 无 workflow registry、不运行任何 workflow：enqueue/send/读 events/查询状态——生产者/观察者与执行者分离；API server 可只装 Client 无 workflow 代码。
- **evidence**: `src/client.rs` L1-25
- **links**: [{to: EK-18, type: causal, note: "Client enqueue 走内部队列当无队列"}, {to: EK-31, type: dependency, note: "依赖队列注册"}, {to: EK-42, type: mechanism, note: "debouncer 也用它"}]
- **status**: Fact

## EK-44 工程治理四门禁 + AGENTS.md 纪律
- **layer**: L2 | **category**: ENGINEERING / GOVERNANCE | **value**: A | **strength**: S3
- **claim**: CI 四门（fmt/clippy -D warnings/test+doc/RUSTDOCFLAGS doc）+ semver-checks 对 crates.io release + feature matrix；边界（禁 push main/force-push 共享分支/cargo publish/提交 secrets/local DB URL）；PR 描述六段模板；无 Co-Authored-By/工具署名；发布走 RELEASING.md。
- **evidence**: AGENTS.md 全文
- **links**: [{to: EK-38, type: mechanism, note: "同治理纪律"}, {to: EK-25, type: constraint, note: "每后端测试强制"}]
- **status**: Fact

## EK-45 Error::Cancelled 控制流不 checkpoint
- **layer**: L1 | **category**: ENGINEERING / FAILURE | **value**: B | **strength**: S3
- **claim**: step body 返回 `Error::Cancelled` 是控制流：不 checkpoint、不 retry（CHANGELOG 0.5.0）。`StepCtx::cancelled()` 提供取消观察。
- **evidence**: CHANGELOG 0.5.0 "Error::Cancelled returned from a step body is control flow: not checkpointed, not retried"
- **links**: [{to: EK-32, type: contrast, note: "业务错误重放 vs 控制流不记录"}, {to: EK-04, type: constraint, note: "与控制错误区分"}]
- **status**: Fact

## EK-46 两层编排归类决策表（新行为属于哪一层）
- **layer**: L2 | **category**: ENGINEERING / DECISION | **value**: A | **strength**: S3
- **claim**: 决策表（execution-layers.md）：结果依赖 call 自身参数 → workflow 层 claim 前拒绝；参数+已 charge position → claim 后拒绝无记录；数据库观察到的事实 → 记录为 step outcome、replay 服务；本进程配置/registry → runtime 层 RecoveryRequired 无记录；storage 是否应答 → ObservationFailed 无恢复。"如果正确配置的进程从相同 call+history 得到不同结果，结果不属于 workflow 记录。"
- **evidence**: docs/design/execution-layers.md "Which side does a new behavior belong to?"
- **links**: [{to: EK-04, type: causal, note: "归类驱动错误通道选择"}, {to: EK-13, type: constraint, note: "归类约束错误可捕获性"}, {to: EK-44, type: causal, note: "AGENTS.md 强制用此规则"}]
- **status**: Validated Pattern（文档 + AGENTS.md 引用）

## EK-47 无服务器嵌入库（无 sidecar，直连 DB）
- **layer**: L1 | **category**: ENGINEERING / ARCHITECTURE | **value**: B | **strength**: S3
- **claim**: "There is no separate server: the engine is a library that runs inside your worker and talks directly to the database"；网络面全 opt-in（admin/conductor feature）。
- **evidence**: README.md / lib.rs L3-9；security.rs "Network exposure" 表
- **links**: [{to: EK-43, type: causal, note: "无服务器 → 出站/入站面可选"}, {to: EK-48, type: constraint, note: "嵌入约束 admin 无认证设计"}]
- **status**: Fact

## EK-48 admin 无认证 by design + start_on(loopback)
- **layer**: L1 | **category**: ENGINEERING / SECURITY | **value**: B | **strength**: S3
- **claim**: AdminServer 无认证是设计决策（与其他 DBOS SDK 对齐，共享工具可用），默认绑全接口（orchestrator health probe 需要）；提供 `start_on((LOCALHOST, port))` 绑定 loopback；文档要求当数据库端口一样防护（私网/firewall）。
- **evidence**: `src/security.rs` "Network exposure" 表 + start_on 示例；`src/admin.rs` 端点列表（dbos-healthz/readyz/deactivate/conductor/queues-metadata/workflow-recovery/global-timeout/garbage-collect/workflows）
- **links**: [{to: EK-47, type: constraint, note: "嵌入库的安全边界交给网络层"}, {to: EK-14, type: mechanism, note: "同安全边界明确化"}]
- **status**: Fact

## EK-49 verification run 一体字段（Option<Arc<Verification>> 非 flag+cell）
- **layer**: L2 | **category**: ENGINEERING / DECISION | **value**: B | **strength**: S3
- **claim**: DurableContext.verify 是 `Option<Arc<Verification>>` 单字段而非 flag + cell："they are one fact: there is no such thing as a verification run without somewhere to report to"；clone 复制 Option（身份可读），只有报告 Arc 共享（clone 的 divergence 对 verifier 可见）。
- **evidence**: `src/context.rs` DurableContext.verify 字段注释（L570-583）
- **links**: [{to: EK-50, type: causal, note: "支撑 verify_replay 实现"}, {to: EK-30, type: mechanism, note: "同防伪报告纪律"}]
- **status**: Fact

## EK-50 verify_replay 盲重建报告（recorded/matched/divergence）
- **layer**: L1 | **category**: ENGINEERING / IMPLEMENTATION | **value**: B | **strength**: S3
- **claim**: `verify_replay(workflow_id)` 把记录过的 workflow 对当前代码重跑，报告第一处分歧；counts 防 vacuous pass（recorded:0 = 空测试绿；matched 远短于 recorded = 没验证到哪）。verification run 服务已记录 outcome、拒绝 live 执行。
- **evidence**: `src/replay.rs` ReplayReport 文档（L1-30）；engine.rs verify_replay（L2315）
- **links**: [{to: EK-49, type: causal, note: "依赖一体字段"}, {to: EK-30, type: mechanism, note: "同防伪报告纪律"}, {to: EK-13, type: constraint, note: "验证 run 拒绝 live 执行"}]
- **status**: Fact

---

# 补充（阶段⑤ Reconciliation 修正）
## EK-51 观察者控制错误通道（observation_error，无 latch 路径）
- **layer**: L1 | **category**: ENGINEERING / FAILURE | **value**: B | **strength**: S3
- **claim**: `observation_error()`（execution.rs）服务无本地 execution latch 的观察者（Client/observer 直连 provider）：把 storage failure 转 `ObservationFailed`（"Failure to observe an operation does not establish that its target stopped"），已 RecoveryRequired 的转 ObservationFailed，语义拒绝保持原样。与 body 内 `provider_error()` 形成"有 latch/无 latch"两条分类路径。
- **evidence**: `src/execution.rs` observation_error（L140-150）——独立盲重建补充（阶段⑤ MISSING 修正）
- **links**: [{to: EK-04, type: mechanism, note: "同为控制错误分类"}, {to: EK-05, type: causal, note: "复用 is_storage_failure 判定"}, {to: EK-43, type: dependency, note: "Client 观察路径依赖"}]
- **status**: Fact

## EK-52 子 workflow 插入的事务+行锁仲裁（insert_child_workflow）
- **layer**: L1 | **category**: ENGINEERING / MECHANISM | **value**: B | **strength**: S3
- **claim**: 子 workflow 注册用显式事务 + `SELECT ... FOR UPDATE` 锁父行 + 重放检查（已记录 child → name 比对不符 = WorkflowConflict；父 CANCELLED → Cancelled），再 insert child；与 claim/checkpoint 的单语句 CAS 并存——**仲裁形式因操作而异**（CAS 单语句或事务+行锁），共同点是仲裁决定权在数据库而非应用锁。
- **evidence**: `src/postgres.rs` insert_child_workflow（L2555-2610，FOR UPDATE + 重放检查）——独立盲重建补充（阶段⑤ MISSING 修正）
- **links**: [{to: EK-08, type: mechanism, note: "同为 DB 仲裁"}, {to: EK-09, type: contrast, note: "事务+锁 vs 单语句 CAS 对照"}, {to: EK-27, type: causal, note: "child 冲突 → WorkflowConflict"}]
- **status**: Fact

## EK Graph 质量指标

| 指标 | 值 | 合格线 |
|---|---|---|
| EK 总数 | 52 | 40~60 |
| 有 links 的 EK | 52/52（100%） | 100% |
| EK 平均出边数 | 52 条 EK 共 112 条 link 引用，≈2.15/条 | ≥1 |
| 游离 EK（无 link） | 0 | <20% |
| D 级 EK | 0（全部有边） | — |
| links 类型覆盖 | mechanism/subsystem/causal/dependency/constraint/contrast 六类全用 | 6 类 |
| 证据强度 | S3×27 / S4×23（无 S5+，静态考古上限） | 如实标注 |
| 聚合到 KO | 52/52 全部进入 ≥1 KO 簇 | 100% |

# 03 Knowledge Layer — durare（Generalized KO，窄尖顶）

> 10 个 Core KO。每个声明 `aggregation_rule`（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）、簇内 EK、边类型。全部可回溯到 02 层 EK 底座。
> Epistemic status：仅项目内证据 → 标注 `durare-originated · Strongly evidenced · Cross-project validation pending`（对照 dbos-transact-py 已考古，部分跨语言印证）。

---

## KO-01 位置即历史：position 在构造时分配，与执行顺序解耦

- **layer**: L3 Pattern | **aggregation_rule**: R2 因果链簇
- **簇内 EK**: EK-01 → EK-02 → EK-03（边：causal/mechanism）
- **claim**: 可重放历史的每个 checkpoint 位置（seq）必须在 durable call **构造时**分配，而不是首次执行/poll 时——否则并发 body（select!/join! 的随机唤醒顺序）会调换两个调用在历史中的位置，崩溃恢复后重放返回错的值。配套三机制：task-local 作用域（谁在跑）、三重放置拒绝（哪里构造/poll 合法）、`#[must_use]`（占位不被悄悄丢弃）。
- **解释范围扩大**: 从"durare 修了一个 select! 乱序 bug"升为"任何 checkpoint-per-call 系统要防并发唤醒序重排历史"。
- **evidence**: EK-01/02/03（context.rs claim!、execution.rs check_placement、explicit-context.md select! 复现）
- **counterexamples**: 官方 dbos-transact-rust 用 task-local + StepBuiltElsewhere 容忍（无 checkpoint 运行）→ 显示这是设计选择不是唯一答案
- **status**: durare-originated · Strongly evidenced · Cross-project validation pending

## KO-02 两层编排 + 单一 seam：workflow 决策与 runtime 所有权分离

- **layer**: L4 Cognitive Model | **aggregation_rule**: R4 主题簇
- **簇内 EK**: EK-04 / EK-05 / EK-13 / EK-15 / EK-18 / EK-22 / EK-23 / EK-46 / EK-47 / EK-48
- **claim**: durable 引擎把两件事分开：**workflow orchestration**（重放做原 run 的决策——"replay makes exactly the decisions the original run made"）与 **runtime orchestration**（所有权、恢复、队列、settlement——"at most one live execution per id"）。两者通过**单一 seam**（execution.rs）交接：向上是 durable-call 协议 + 二型控制错误，向下是 runtime 的所有权/恢复动作。新行为按决策表归层。
- **解释范围扩大**: 从 durare 的模块划分升为"编排引擎的通用分界"：重放正确性属于 workflow 层，并发安全属于 runtime 层，二者必须可独立推理。
- **evidence**: EK-46（execution-layers.md 决策表）、EK-04/05/13（seam 协议）、EK-15/18/22（runtime 动作）
- **counterexamples**: 单层引擎（如把恢复内联在 body 调用里）也能工作但会阻塞 caller——durare 明确选不 inline
- **status**: durare-originated · Strongly evidenced · Cross-project validation pending

## KO-03 控制错误不是业务失败：恢复信号必须类型级隔离且不可被消化

- **layer**: L4 Cognitive Model | **aggregation_rule**: R2 因果链簇
- **簇内 EK**: EK-04 → EK-05 → EK-06 → EK-13 → EK-15 → EK-32 → EK-51（causal/mechanism）
- **claim**: 基础设施失败（存储应答不了、解码失败、恢复中）必须与业务失败在错误类型层面隔离（RecoveryRequired/ObservationFailed vs Application），且**不允许业务代码捕获并继续**——一条被 catch 的控制错误会在下一个 durable call 再次拒绝（latch 幂等）。ObservationFailed 特别不能据此调度恢复（操作可能已发生，只是看不到结果）。无本地 latch 的观察者（Client）用 observation_error 转 ObservationFailed——同一分类纪律扩展到执行之外。
- **解释范围扩大**: 从 durare 的错误设计升为"恢复语义不能靠约定，要靠在类型/通道层面不可绕过"。
- **evidence**: EK-04/05/06/13/15/32/51（execution.rs、error.rs、execution-layers.md）
- **counterexamples**: 普通 retry 策略（用户可 catch 并重试业务错误）与 RecoveryRequired 并行存在——说明不是所有失败都要隔离，只有"需要恢复协调"的才要
- **status**: durare-originated · Strongly evidenced · Cross-project validation pending

## KO-04 数据库即仲裁者：fleet 所有权仲裁下沉到 DB 原子原语族

- **layer**: L4 Cognitive Model | **aggregation_rule**: R1 机制簇
- **簇内 EK**: EK-07 / EK-08 / EK-09 / EK-20 / EK-52（mechanism/contrast 边连通，跨 postgres.rs/provider.rs 独立文件）
- **claim**: 多进程 executor fleet 的所有权仲裁不靠分布式锁，仲裁决定权在**数据库而非应用锁**。实现形式因操作而异：单语句 CAS（claim_for_recovery："the predicate, not a transaction, is the arbiter"）、insert `ON CONFLICT DO NOTHING`（一个 position 一个 checkpoint；created flag 仲裁直接 start）、data-modifying CTE（checkpoint insert + executor refresh 一次往返）、以及**事务 + `SELECT ... FOR UPDATE` 行锁**（insert_child_workflow：锁父行 + 重放检查 + CANCELLED 检查）。赢家由 DB 决定，输家拿类型化错误安静退出。
- **解释范围扩大**: 从 durare 的并发控制升为"数据库原子操作可作为多进程所有权的通用仲裁机制"——形式不唯一（CAS/唯一约束/CTE/行锁），共同点是仲裁与状态变更同处数据库事务原子性内。
- **evidence**: EK-07/08/09/20/52（postgres.rs record_step_result/claim_for_recovery/insert_child_workflow、provider.rs dedup_or）
- **counterexamples**: 多 dispatcher 竞争 enqueue claim 仍需 ENQUEUED→PENDING 条件更新；行锁路径有死锁可能（Postgres 超时兜底）——原子语句要按状态机配对
- **status**: durare-originated · Strongly evidenced · Cross-project validation pending（dbos-transact-py 也依赖 DBOS schema 的同样约束——跨语言印证；表述经阶段⑤盲重建修正：原"不靠事务序列化"过度，现明确含事务+行锁路径）

## KO-05 事务步骤原子消除 at-least-once window

- **layer**: L3 Pattern | **aggregation_rule**: R2 因果链簇
- **簇内 EK**: EK-10 → EK-11 → EK-12 → EK-32（causal/contrast）
- **claim**: 当 step 的效果是数据库写时，把 body 的 SQL 写与 checkpoint 放进**同一事务**，使效果 exactly-once；否则普通 step 是"执行后 checkpoint"两写，崩溃窗口内效果至多一次重放（at-least-once window 恰好一步宽）。配套：事务冲突内层无限重试（冲突不记录）、用户错误外层按策略重试（记录失败是终态）、read-only 退化普通 checkpoint。
- **解释范围扩大**: 从 durare 的 transaction 步骤升为"at-least-once 是通用代价，消除它需要把效果与记录放进同一原子单位"。
- **evidence**: EK-10/11/12/32（transactions.rs、postgres.rs run_transaction_step、crash_consistency.rs）
- **counterexamples**: 普通 step 保留 at-least-once（幂等效果安全时不用事务）——消除是有代价的（必须 Fn 可重跑、SQL 必须方言无关）
- **status**: durare-originated · Strongly evidenced · Cross-project validation pending

## KO-06 跨 SDK 字节级兼容协议（DBOS 生态的共享 wire contract）

- **layer**: L4 Cognitive Model | **aggregation_rule**: R4 主题簇
- **簇内 EK**: EK-27 / EK-35 / EK-36 / EK-37 / EK-33（mechanism/subsystem 边，互补维度）
- **claim**: DBOS 多语言 SDK 在同一 Postgres 上互操作的可行方式 = 把**字节级契约**固化进 schema：portable_json 逐字节一致（serde_json preserve_order + golden strings pinned in every SDK test suite）、时间戳存 BIGINT epoch ms、`serialization` 列做格式分发解码、sentinel/roles 的 on-disk shape 固定。协议不是"语义对齐"而是"字节对齐"。
- **解释范围扩大**: 从 durare 的互操作测试升为"跨语言分布式系统用共享存储时的 wire contract 设计模式"。
- **evidence**: EK-27/35/36/37/33（interop.rs、serialize.rs、schema.md、recorded_error.rs）
- **counterexamples**: 默认格式 DBOS_JSON（base64）正是为了保留"非 portable"的自由——字节契约只在需要互操作的值上强制
- **status**: durare-originated · Strongly evidenced · **跨语言印证**（interop.rs 明示 golden strings 在其他 SDK 套件相同）

## KO-07 版本门控的部署安全：恢复与认领只属于同版本执行者

- **layer**: L3 Pattern | **aggregation_rule**: R3 不变量簇
- **簇内 EK**: EK-16 / EK-17 / EK-21 / EK-22（constraint 边汇聚）
- **claim**: 滚动部署安全不变量：**PENDING/ENQUEUED 行只能被同一 application version 的执行者认领**。实现：默认 version=可执行文件 sha-256（构建即不同版本）、dequeue 谓词 `version=$n OR IS NULL`、恢复版本门控、queue 注册仅 latest 可 overwrite（旧 executor 不能覆盖新配置）。语义不兼容的代码演进 = 版本演进，两个版本可安全并存。
- **解释范围扩大**: 从 durare 的版本机制升为"workflow 引擎滚动部署的通用安全模型"。
- **evidence**: EK-16/17/21/22（Cargo.toml、postgres.rs、engine.rs recover）
- **counterexamples**: 固定 app_version 部署（CI 可复现构建）0.4→0.5 需先 drain——版本门控不是免费
- **status**: durare-originated · Strongly evidenced · Cross-project validation pending

## KO-08 确定性执行纪律：replay 安全是编译期/运行时前置，不是事后检测

- **layer**: L4 Cognitive Model | **aggregation_rule**: R3 不变量簇
- **簇内 EK**: EK-01 / EK-03 / EK-38 / EK-39（constraint/contrast 边汇聚）
- **claim**: 不变量：**workflow body 的每次重放必须产生相同的历史**。执行方式：①构造时占位（poll 顺序无关）；②危险放置拒绝（嵌套/跨执行/spawned task——会丢 position 或跨 run 的调用在原型里 40 次运行 9 种顺序）；③determinism 规则书（时钟/UUID/rand/HashMap 迭代 → ctx.now()/ctx.uuid()/BTreeMap）；④UnexpectedStep 事后 catch 只是 backstop（"a backstop, not a substitute"）。显式 context 使这些检查在编译期/首个 poll 即触发，而非官方 Rust SDK 的容忍式运行。
- **解释范围扩大**: 从 durare 的确定性机制升为"可重放系统的安全前置"：确定性要靠 API 形状保证，不能靠程序员自律。
- **evidence**: EK-01/03/38/39（explicit-context.md、determinism.rs、execution.rs）
- **counterexamples**: 官方 SDK 允许 spawned task 内 step（无 checkpoint 运行）——说明严格性有取舍，durare 选择失败快
- **status**: durare-originated · Strongly evidenced · Cross-project validation pending（与 dbos-transact-rust 对照为 contrast 证据）

## KO-09 动态 SQL 不变量由外部强制：注入防护不依赖代码审查

- **layer**: L3 Pattern | **aggregation_rule**: R3 不变量簇
- **簇内 EK**: EK-14 / EK-26 / EK-28 / EK-34（constraint/mechanism 边汇聚）
- **claim**: 安全不变量：**无 caller 字符串进入 SQL 文本**。实现分层：①bind parameter 为唯一通道；②插值仅编译期常量；③唯一配置来源标识符按白名单正则校验。**强制方式是从外部推恶意串**（tests/security.rs 把 `'; DROP TABLE ...` 经全部 string-typed 输入推入两个后端，断言 verbatim round-trip + 表完好）——"enforced from the outside rather than by pinning the source"。
- **解释范围扩大**: 从 durare 的 SQL 安全升为"防注入不变量用外部攻击测试强制，比源码审查更防重构腐化"。
- **evidence**: EK-14/26/28/34（security.rs、tests/security.rs）
- **counterexamples**: SQLite+Postgres 都测（方言差异不改变不变量）；schema 标识符是唯一例外路径（白名单正则）
- **status**: durare-originated · Strongly evidenced · Cross-project validation pending

## KO-10 崩溃一致性测试方法论：边界枚举 + 精确计数

- **layer**: L5 Methodology | **aggregation_rule**: R4 主题簇
- **簇内 EK**: EK-24 / EK-25 / EK-26 / EK-30（mechanism/subsystem 边）
- **claim**: 可行动准则：①枚举每个 checkpoint 边界（Entry/InStep/AfterStep）并精确断言 effect 计数，不用 "at least once"；②同一行为在每后端各跑一遍（contract(db)）；③多 executor 对抗（独立连接池竞争同一 DB）验证 fleet 级不变量（exactly-once/所有权）；④防 vacuous pass（recorded:0 的空测试不算绿）。"A report with recorded: 0 passes in the same sense an empty test suite is green."
- **解释范围扩大**: 从 durare 的测试套件升为"对 checkpoint-replay 系统写回归测试的方法论"。
- **evidence**: EK-24/25/26/30（crash_consistency.rs、adversity.rs、child_atomic.rs、replay.rs）
- **counterexamples**: 普通 unit test 无法覆盖崩溃窗口——这正是 crash sweep 存在的理由；SQLite 无条件跑 + Postgres 条件跑（DATABASE_URL）平衡 CI 成本
- **status**: durare-originated · Strongly evidenced · Cross-project validation pending

---

## KO 聚合规则矩阵（v3.1 检查）

| KO | 规则 | 簇内 EK 数 | 主要边 | 解释范围扩大 | 可命名 | 可回溯 |
|---|---|---|---|---|---|---|
| KO-01 | R2 因果链 | 3 | causal/mechanism | ✓ | ✓ | ✓ |
| KO-02 | R4 主题 | 10 | mechanism/subsystem | ✓ | ✓ | ✓ |
| KO-03 | R2 因果链 | 7 | causal/mechanism | ✓ | ✓ | ✓ |
| KO-04 | R1 机制 | 5 | mechanism/contrast | ✓ | ✓ | ✓ |
| KO-05 | R2 因果链 | 4 | causal/contrast | ✓ | ✓ | ✓ |
| KO-06 | R4 主题 | 5 | mechanism/subsystem | ✓ | ✓ | ✓ |
| KO-07 | R3 不变量 | 4 | constraint | ✓ | ✓ | ✓ |
| KO-08 | R3 不变量 | 4 | constraint/contrast | ✓ | ✓ | ✓ |
| KO-09 | R3 不变量 | 4 | constraint/mechanism | ✓ | ✓ | ✓ |
| KO-10 | R4 主题 | 4 | mechanism/subsystem | ✓ | ✓ | ✓ |

- 聚合规则覆盖率：10/10 = 100%
- KO 平均簇规模：5.0（3~12 健康区间）
- 无"同子系统=聚合理由"的假聚合（每 KO 的边类型明确）

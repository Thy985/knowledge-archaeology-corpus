# Knowledge Layer — dbos-transact-py（ARCH-2026-10-28-001）

> 8 个 KO；每个声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证 + Epistemic 状态。

## KO-D01 数据库事务 = 执行权锁 + checkpoint 边界（L4 Cognitive Model）
- **aggregation_rule**: R1 机制簇（mechanism 边 D02↔D03 共享"数据库行即所有权"机制；跨 init/dequeue/datasource 独立子系统出现）
- **簇内 EK**: D01（数据库即状态）, D02（owner_xid 互斥）, D03（creator_xid）, D08（dequeue RETURNING 原子翻转）, D20（数据源共享事务）, D21（owner 条件删除）
- **知识**: 在一个以数据库为中心的 durable execution 系统中，**数据库事务同时承担两个正交职责：执行权互斥（WHERE owner_xid=token 的写条件）与持久化原子性（checkpoint 边界）**——二者合一消除分布式锁与持久化的两阶段问题
- **解释范围扩大**: 从"怎么防并发写同一 workflow"（D02）扩展到"整个系统为何不需要独立协调器"（D01/D20）——数据库 ACID 事务是唯一的原语
- **epistemic**: Pattern→Model（DBOS 内强证据；跨项目 validation pending——Restate 用专用日志段+epoch 锁，对照验证中）
- [code] dbos/_sys_db.py:1490-1590, dbos/_core.py:673-676, dbos/_sys_db.py:5371-5436, dbos/_datasource.py:113-128

## KO-D02 启动→认领→执行→checkpoint→完成/失败→恢复/死信 因果链（L4）
- **aggregation_rule**: R2 因果链簇（causal 边 D15→D08→D09→D05→D13 完整链）
- **簇内 EK**: D15（单事务持久化）, D08（dequeue 原子认领）, D09（attempt+1）, D05（步骤 checkpoint）, D13（状态机）, D11（死信）, D12（幂等恢复）
- **知识**: workflow 的完整生命周期是一条**数据库因果链**：初始行（单事务）→ 认领翻转（RETURNING）→ 步骤结果累积（operation_outputs）→ 终态写入（owner 认证）→ 失败后恢复（幂等谓词）或死信（预算耗尽）——每一步都是可见的数据库行，审计即查表
- **解释范围扩大**: 从单个机制（dequeue 怎么防乱序）到整个生命周期可观测性（每个状态转移都是 SQL）
- **epistemic**: Model（DBOS 强证据）
- [code] dbos/_core.py:651-770, dbos/_sys_db.py:5371-5436, 1444-1473, 5443-5475

## KO-D03 执行权单归属不变量（L4 Principle）
- **aggregation_rule**: R3 不变量簇（constraint 边汇聚到同一安全不变量"每行至多一个 owner"）
- **簇内 EK**: D02（owner_xid）, D03（creator_xid 不可变）, D04（park 轮询）, D10（claim_guard 重校验）, D21（owner 条件删除）
- **知识**: **任何状态写必须被当前 owner 认证（WHERE owner_xid=token），否则写入方必须放弃并等待 owner 结果**——这是贯穿所有执行路径（完成/失败/取消/恢复/数据源清理）的安全不变量；owner 认证失败即"我不再拥有执行权"的信号，绝不覆盖式写入
- **解释范围扩大**: 从"防并发写"（D02）扩展到所有写路径的认证纪律（D04/D10/D21）——统一了执行、恢复、清理三种场景的竞争语义
- **epistemic**: Principle（DBOS 内强证据；Cross-project validation pending——Restate 的 epoch/leader 模型是同类不变量，对照中）
- [code] dbos/_core.py:900-944, dbos/_sys_db.py:5387-5394, dbos/_datasource.py:320-330

## KO-D04 步骤 checkpoint 三态语义（L3 Pattern）
- **aggregation_rule**: R4 主题簇（D05/D06/D07/D24 覆盖"步骤持久化"互补维度）
- **簇内 EK**: D05（结果缓存）, D06（失败重抛）, D07（不可序列化 fail-open）, D24（per-row serialization）
- **知识**: 步骤 checkpoint 处理三种结局：**成功→缓存结果供重放；失败→记录错误供重抛（不重跑）；不可序列化→放弃记录（重跑）**——确定性恢复的代价是"重放必须复现相同结局"，序列化失败时宁可不 checkpoint 也要避免 wedge
- **解释范围扩大**: 从"怎么存步骤结果"（D05）到"每种结局如何保证重放确定性"（D05/D06/D07/D24）
- **epistemic**: Pattern（DBOS 强证据；同类对照：Restate journal Command/Completion 二分）
- [code] dbos/_sys_db.py:3410-3495, 3648-3660, dbos/_core.py:1285-1300

## KO-D05 durable 原语族：并发/信号/流全部可 checkpoint（L4）
- **aggregation_rule**: R1 机制簇（mechanism 边 D16↔D17↔D19 共享"状态持久化换取恢复确定性"机制）
- **簇内 EK**: D16（durable asyncio.wait）, D17（LISTEN/NOTIFY 通知）, D18（事件双表）, D19（streams offset）
- **知识**: durable execution 不止"步骤重放"——**并发等待（asyncio.wait）、跨 workflow 信号（events/notifications）、流式消费（streams）都被改造成可持久化原语**：等待结果/事件值/消费 offset 存数据库，恢复时确定性重建
- **解释范围扩大**: 从"workflow 步骤持久化"（D05）到"所有控制流语义都持久化"（asyncio.wait 的 done/pending 索引）
- **epistemic**: Model（DBOS 强证据）
- [code] dbos/_dbos.py:2100-2180, dbos/_sys_db.py:431-432, 3883-3971, dbos/_schemas/system_database.py:146-200

## KO-D06 队列即数据库行（L3 Pattern）
- **aggregation_rule**: R2 因果链簇（causal 边 D22→D08→D09→D23）
- **簇内 EK**: D08（dequeue 原子认领）, D09（attempt+1）, D10（claim_guard）, D22（分片限流）, D23（rate_limited/DequeueStopReason）
- **知识**: durable queue 的语义完全由数据库行表达：**enqueue=插入行（ENQUEUED/DELAYED），dequeue=原子翻转（RETURNING），限流=行级 rate_limited 标志 + 并发预算检查**；队列顺序纪律（禁 LIMIT+SKIP LOCKED）来自真实乱序 bug
- **解释范围扩大**: 从"dequeue 怎么原子化"（D08）到"队列全语义=行状态"（enqueue/限流/分区/优先级）
- **epistemic**: Pattern（DBOS 强证据；同类对照：Temporal task queue、Restate vqueues 都是"队列即状态"）
- [code] dbos/_sys_db.py:5371-5436, 156-171, dbos/_queue.py:356-505

## KO-D07 恢复预算治理（L4 Model）
- **aggregation_rule**: R4 主题簇（D09/D11/D12/D13 覆盖"失败后怎么办"）
- **簇内 EK**: D09（attempt 计 dispatch）, D11（死信）, D12（幂等恢复）, D13（状态机）
- **知识**: 恢复不是无限重试，而是**有预算的治理闭环**：每次 dispatch 消耗 recovery_attempts 预算 → 超限死信（MAX_RECOVERY_ATTEMPTS_EXCEEDED + 清理）→ 人工/程序 resume 给新预算；恢复请求必须幂等（executor_ids 谓词防重复入队）
- **解释范围扩大**: 从"失败怎么重试"到"失败治理的全生命周期（预算/死信/恢复/审计）"
- **epistemic**: Model（DBOS 强证据；对照 Restate OnMaxAttempts Pause|Kill 生命周期两态）
- [code] dbos/_sys_db.py:1444-1473, 1728-1747, 5405-5412, 5443-5475

## KO-D08 durable execution 存储哲学二分：数据库 vs 专用日志（L5 Principle）
- **aggregation_rule**: R4 主题簇（D01/D02/D05 汇聚"存储即 journal"主题）+ 跨项目对照（restate bifrost/dapr-agents/substrate）
- **簇内 EK**: D01（数据库即状态）, D02（事务锁）, D05（步骤 checkpoint）
- **知识**: durable execution 存在两条存储哲学路线：**"你已有的数据库"（DBOS：Postgres/SQLite 表即 journal，零额外基础设施，事务原子性内建）vs "专用日志段"（Restate：bifrost RocksDB/sequencer/memory provider，Temporal：自研历史存储）**——前者胜在运维面（借用已有 DB 可靠性），后者胜在能力面（专用分段/复制/截断）
- **解释范围扩大**: 从单项目机制到品类二分；周五 7-way 横向的分类轴
- **epistemic**: **Principle（Cross-project validation pending）**——restate（10-27）与 DBOS（本轮）两案例构成对照证据，但未达"普遍成立"；须在 10-30 品类横向中验证
- [code] DBOS: dbos/_schemas/system_database.py:23-274, dbos/_sys_db.py:1490-1590；Restate: crates/bifrost/src/bifrost.rs:67（bifrost 日志段抽象）——跨仓库对照，属 benchmark 验证项

## 聚合矩阵
| KO | 规则 | 簇内 EK | L 层 | Epistemic |
|---|---|---|---|---|
| D01 | R1 | D01,D02,D03,D08,D20,D21 | L4 | Model（pending 跨项目） |
| D02 | R2 | D15,D08,D09,D05,D13,D11,D12 | L4 | Model |
| D03 | R3 | D02,D03,D04,D10,D21 | L4 | Principle（pending） |
| D04 | R4 | D05,D06,D07,D24 | L3 | Pattern |
| D05 | R1 | D16,D17,D18,D19 | L4 | Model |
| D06 | R2 | D08,D09,D10,D22,D23 | L3 | Pattern |
| D07 | R4 | D09,D11,D12,D13 | L4 | Model |
| D08 | R4 | D01,D02,D05 | L5 | Principle（pending） |

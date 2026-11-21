# Knowledge Layer — Restate（Generalized KO，L3/L4/L5 窄尖顶）

> v3.1 规范：每个 KO 必须声明 aggregation_rule（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）+ 簇内 EK + 边类型。
> 跨项目结论一律标 `Cross-project validation pending`；Hypothesis 进 05_candidates。

---

## KO-01 步骤持久化（Durable Journal）是 durable execution 的核心抽象（L3 Pattern）
- **一句话**：把 handler 执行切成可持久化的步骤序列（journal），每步落盘后崩溃可从最后完成步骤恢复——这是"步骤持久化派"durable execution 的机制核心。
- **解释范围**：任何需要"长运行 + 崩溃恢复 + 精确一次"的运行时（Agent 编排、workflow 引擎、事务型 actor）。
- **aggregation_rule**：R2 因果链簇——EK-01(步骤) → EK-02(原子追加) → EK-03(pre-flight) → EK-31/41(恢复锚点)
- **对照**：与 agent-substrate（ARCH-2026-10-31-001，挂起恢复派）构成同一问题两条路线。
- **证据强度**：S3+S4（Restate）；跨项目 = `Cross-project validation pending`
- **links**：causal(01→02→03→31→41)

## KO-02 幂等是持久化执行的显式第一公民，而非事后补救（L3 Pattern）
- **一句话**：调用级别幂等（idempotency key→确定性 InvocationId→按状态分派）被实现为状态机的核心分支，而非 HTTP 层过滤。
- **解释范围**：需要精确一次语义的系统（消息队列消费、事件驱动、webhook 重试）。
- **aggregation_rule**：R1 机制簇——EK-04/05/08/09/12 共享幂等分派机制，跨 ingest/worker/types 三个子系统
- **证据强度**：S3+S4（idempotency.rs 测试）
- **links**：mechanism(04↔05↔08↔09↔12)

## KO-03 幂等分片：确定性分区是去重的前提（L3 Pattern）
- **一句话**：同幂等键的重复请求必须路由到同一分区，去重才成立——因此分区键设计是幂等系统的不可变契约。
- **解释范围**：分布式队列/分片存储设计；任何"先分区后去重"的系统。
- **aggregation_rule**：R3 不变量簇——EK-09/10/11/39 汇聚到"幂等键→同分区"不变量
- **证据强度**：S3 + 注释（S2 不变量声明："changing it would break deduplication"）
- **links**：constraint(09/10/11→39)

## KO-04 挂起/恢复：把"等待"变成可恢复的状态而非阻塞线程（L3 Pattern）
- **一句话**：suspend 优先尝试立即解析（notifications 已就绪→直接 resume），否则写 SuspendedEvent 转 Suspended 状态，等待通知到达后恢复——等待本身被持久化。
- **解释范围**：Agent 等待外部事件/人机交互/定时器；workflow 的 sleep/await。
- **aggregation_rule**：R2 因果链簇——EK-13(先解析) → EK-14(挂起) → EK-15(恢复) → EK-16(awaiting_on 树) → EK-34(Sleep)
- **证据强度**：S3
- **links**：causal(13→14→15→16→34)

## KO-05 组合 future 树（awaiting_on）是异步编排的通用语言（L3 Pattern）
- **一句话**：用不可变的组合 future（race/all/any/allSettled 语义）表示"恢复条件"，PENDING→SUCCEEDED/FAILED 单向不可变，通知解析即恢复。
- **解释范围**：workflow 引擎、Agent 编排的 fan-out/fan-in。
- **aggregation_rule**：R4 主题簇——EK-16（组合子）+ EK-34（Sleep）+ EK-13/15（解析/恢复）覆盖"恢复条件"主题
- **证据强度**：S3
- **links**：subsystem(16↔34↔13↔15)

## KO-06 分片内串行 + 分片间并行：keyed 调度的无锁保证（L3 Pattern）
- **一句话**：exclusive handler 独享队列（hash 到独立 VQueueId）、shared service 共享队列，队列内天然串行无需锁；配合显式锁（LockState）处理跨 key 场景。
- **解释范围**：actor 模型、单线程 per-key 处理、无锁数据结构设计。
- **aggregation_rule**：R1 机制簇——EK-17/18/19/28 共享 keyed 串行机制，跨 vqueues/partition-store 子系统
- **证据强度**：S3
- **links**：mechanism(17↔18↔19↔28)

## KO-07 日志与状态分离：durable log 作为单一事实源（L3 Pattern）
- **一句话**：Bifrost 分布式日志是 durable interconnect——append 返回 offset 即 durable committed；分区状态机消费日志推进 applied_lsn，快照+日志重放恢复。
- **解释范围**：事件溯源、WAL、数据库复制。
- **aggregation_rule**：R2 因果链簇——EK-20(日志抽象) → EK-21(链) → EK-22(恢复策略) → EK-32(快照) → EK-41(FSM 消费)
- **证据强度**：S3
- **links**：causal(20→21→22→32→41)

## KO-08 版本化元数据 + 乐观并发 = 分布式配置的安全更新（L3 Pattern）
- **一句话**：全局元数据（schema/logs/nodes config）通过版本化 kv + Precondition（MatchesVersion）更新，杜绝无意识覆盖。
- **解释范围**：分布式系统配置管理、schema 迁移协调。
- **aggregation_rule**：R4 主题簇——EK-24/25/26/27 覆盖元数据主题的读写/并发/同步/一致性
- **evidence**：metadata.rs:55 Precondition；MetadataManager 四路 GlobalMetadataUpdateTask
- **证据强度**：S3
- **links**：subsystem(24↔25↔26↔27)

## KO-09 迁移即一等公民：在线 schema 迁移用持久化 feature 标记驱动（L4 Cognitive Model，Cross-project validation pending）
- **一句话**：存储 schema 演化不该停机；把"迁移是否完成"持久化为 feature 标记（PersistedFeatures），通过日志命令（VersionBarrierCommand）向全分区广播，而非各节点自行判断。
- **解释范围**：任何需要滚动升级存储层的系统（数据库、消息队列、状态存储）。
- **aggregation_rule**：R1 机制簇——EK-29/30/31/42 共享迁移标记机制，跨 partition-store/worker
- **证据强度**：S3（实现存在，未验证大规模滚动迁移运行时行为——Observation）
- **links**：mechanism(29↔30↔31↔42)

---

## 认知状态与升维纪律
- KO-01~08 为 Restate 单项目内验证的 L3 Pattern；跨项目推广全部标 `Cross-project validation pending`（无第二项目证据）。
- KO-09 为 L4 候选，含运行时 Observation 成分，IVR 将攻击其升维合理性。
- 无 L5 方法论（单项目证据不足，禁止凭项目经验宣布通用 Principle）。

# Knowledge Layer — Restate Generalized KO（L3-L5 窄尖顶）

> 每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证。未跨项目验证的模式标 `Cross-project validation pending`。

## KO-R01 Journal 即真相（L4 Cognitive Model）
- **aggregation_rule**: R1 机制簇（EK-R01/EK-R02/EK-R03 共享 journal 机制边，跨 journal_v2/storage-api 两个独立文件面）
- **内容**: 可恢复执行系统应以"不可变副作用日志"为唯一事实源，状态与完成标记是其派生/物化视图；恢复 = 重放 + 跳过已完成
- **证据链**: EK-R01 → EK-R02 → EK-R03 → EK-R06 → EK-R19（journal → completion → 版本化 → 重放 → Lsn）
- **解释范围扩大**: 从 Restate journal 到任何 durable execution（Dapr 步骤日志/AWS Lambda Durable checkpoint 同构）——对照 dapr-agents 验证
- **状态**: L4 · Cross-project validation pending（dapr-agents 已考古可对照）

## KO-R02 步骤持久化 vs 整机快照（L4 Cognitive Model）
- **aggregation_rule**: R4 主题簇（EK-R01/EK-R04/EK-R19/EK-R16 围绕"恢复机制"互补维度）
- **内容**: "恢复"有两条互补哲学：步骤持久化（journal 重放，粒度=副作用）与整机快照（snapshot restore，粒度=内存态）；前者恢复成本与执行历史成正比，后者与状态大小成正比
- **证据链**: EK-R01（journal）+ EK-R04（派生状态）+ EK-R19（Lsn）vs substrate KO-S01（快照-恢复三件套）
- **解释范围扩大**: 直接解释 durable execution 品类（Dapr/Restate/MS/Lambda）vs 沙箱快照（Substrate）的分野；服务 TeamMind 方案选型
- **状态**: L4 · Cross-project validation pending（substrate 已考古对照——周五 7-way 素材）

## KO-R03 幂等三层组合（L3 Pattern）
- **aggregation_rule**: R2 因果链簇（EK-R05→EK-R18→EK-R10→EK-R11：幂等键→去重→保留期→purge）
- **内容**: 幂等 = 业务键 × 目标 × 调用方组合 + 完成状态保留期 + 生产者级去重；三层缺一不可（无保留期不去重、无 producer 去重会重复投递）
- **证据链**: EK-R05 → EK-R18 → EK-R10 → EK-R11
- **解释范围扩大**: 任何分布式调用框架（gRPC 幂等键/HTTP Idempotency-Key）的完整幂等设计模板
- **状态**: L3 · Cross-project validation pending

## KO-R04 对象级串行化（虚拟对象模型）（L4 Cognitive Model）
- **aggregation_rule**: R1 机制簇（EK-R08/EK-R09/EK-R15 共享 vqueues+lock 机制，跨 scheduler/partition-store 两个子系统）
- **内容**: 单线程执行幻觉 = 按 key 排队（vqueue）+ 持久锁（lock_table）+ 公平调度（RefillMode）；对象状态与执行队列都持久化，崩溃后队列不丢
- **证据链**: EK-R08 → EK-R09 → EK-R15 → EK-R06
- **解释范围扩大**: Actor 模型（Orleans/Dapr actors）的队列实现范式；虚拟对象与 actor 的并发语义对照
- **状态**: L4 · Cross-project validation pending（Dapr actors 对照）

## KO-R05 权威-状态-日志三分离（L5 Methodology）
- **aggregation_rule**: R3 不变量簇（EK-R07/EK-R22/EK-R06/EK-R21 汇聚到"分区状态一致性"不变量）
- **内容**: 分片系统应分离：日志（持久化事实）、派生状态（物化视图）、权威（leader 执行权）；leader 可迁移（epoch），但日志与状态不随 leader 消失；进度显式持久化（durable_point/last_applied_lsn）
- **证据链**: EK-R07 → EK-R22 → EK-R06 → EK-R21
- **解释范围扩大**: K8s controller/数据库分区/Dapr actor runtime 的通用一致性架构
- **状态**: L5 方法论（可操作：写日志→应用→记录进度→可迁移）· Cross-project validation pending

## KO-R06 日志后端可插拔（L3 Pattern）
- **aggregation_rule**: R4 主题簇（EK-R16/EK-R17/EK-R19 围绕"持久化后端"互补）
- **内容**: 日志抽象（Loglet Provider）把单机（RocksDB）与集群（replicated）统一为 append→Lsn 接口；部署形态（单节点 vs 多节点）不改变应用语义
- **证据链**: EK-R16 → EK-R17 → EK-R19
- **解释范围扩大**: 存储后端抽象的通用模板（类似 etcd raft vs 单机）
- **状态**: L3 · 本项目多 provider 已验证（local/replicated/memory 并存）

## KO-R07 挂起/终止/重启三态语义（L3 Pattern）
- **aggregation_rule**: R2 因果链簇（EK-R24→EK-R12→EK-R14：生命周期状态→attempt 机→挂起通知优先）
- **内容**: 长时程执行的显式生命周期：suspend（可恢复，journal 保留）≠ terminate（不可恢复）≠ restart_as_new（换 journal 保对象）；suspend 前检查 notification 可用性避免空挂
- **证据链**: EK-R24 → EK-R12 → EK-R14
- **解释范围扩大**: agent 长任务（等待人工/等待外部事件）的生命周期管理模板——服务 TeamMind 长时程 agent
- **状态**: L3 · Cross-project validation pending

## KO-R08 重试与背压分层（L4 Cognitive Model）
- **aggregation_rule**: R3 不变量簇（EK-R12/EK-R13/EK-R20/EK-R23 汇聚到"资源受控执行"不变量）
- **内容**: 执行失败与资源压力应分层治理：attempt 级重试（RetryPolicy，Pause|Kill）只管执行失败；分区级限流（RuleBook，日志一致）管资源压力；两层不互相替代
- **证据链**: EK-R12 → EK-R13 → EK-R20 → EK-R23
- **解释范围扩大**: agent 运行时（harness/编排器）的容错分层设计；对照 substrate 请求停车（背压）与 dapr 重试策略
- **状态**: L4 · Cross-project validation pending

## KO 聚合矩阵
| KO | 规则 | 簇内 EK | 状态 |
|---|---|---|---|
| KO-R01 | R1 | 01,02,03 | L4 pending |
| KO-R02 | R4 | 01,04,19,16 | L4 pending |
| KO-R03 | R2 | 05,18,10,11 | L3 pending |
| KO-R04 | R1 | 08,09,15 | L4 pending |
| KO-R05 | R3 | 07,22,06,21 | L5 pending |
| KO-R06 | R4 | 16,17,19 | L3 项目内验证 |
| KO-R07 | R2 | 24,12,14 | L3 pending |
| KO-R08 | R3 | 12,13,20,23 | L4 pending |

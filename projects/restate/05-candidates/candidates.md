# Candidates — Restate 未验证假说 / 跨项目模式

> 与 KO 严格区分：不能确认的内容留在此处，标 Hypothesis。

## A. 跨项目假说（Cross-project Hypothesis）

**C-R01 幂等三层组合是分布式调用框架通用模板**
- 假说：业务键×目标×保留期×producer 去重的四件套可迁移到任意 RPC 框架（gRPC/HTTP）
- 证据：EK-R05/R18/R10（本项目强证据）；无第二项目验证
- 缺失证据：非 Restate 框架的完整实现对照
- 验证路径：对照 dapr-agents 幂等语义 / HTTP Idempotency-Key 标准

**C-R02 虚拟对象 = 队列+锁+调度 是 actor 模型的实现范式**
- 假说：vqueues（按 key 排队+LockTable+RefillMode 公平调度）是 actor/virtual actor 的一种通用实现
- 证据：EK-R08/R09/R15；Dapr actors 理论对照
- 缺失证据：Dapr actor 实现源码级对照
- 验证路径：周五 7-way 横向 + Dapr quickstart 实测

**C-R03 journal 即真相 vs 快照即真相 是恢复机制的完整二分**
- 假说：所有 durable/可恢复执行系统落入"步骤持久化"或"整机快照"两端（或混合）
- 证据：Restate（journal）+ substrate（snapshot）双项目已考古；Vercel/Notion/Lambda 记录
- 缺失证据：MS Durable Extension/AWS Lambda Durable 实现细节（平台黑盒）
- 验证路径：7-way 对照表（周五）

## B. 暂定模式（Tentative，本项目内不确定）

**C-R04 journal v1→v2 迁移语义**
- 观察：storage-api 同时存在 journal_table 与 journal_table_v2；lifecycle/migrate_journal_table.rs 存在
- 不确定：v2 的激活条件与 v1 兼容边界（迁移触发时机未深读）
- 证据：migrate_journal_table.rs 存在性 [code]

**C-R05 RuleBook 限流粒度（key/用户/全局）**
- 观察：limiter 有 user_limits.rs + LimitKey 层次化；upsert 走日志
- 不确定：多级限流的实际交互语义（优先级/配额分配）未完整验证
- 证据：limiter/src/{rule_book,user_limits}.rs

**C-R06 query_engine（DataFusion）面**
- 观察：datafusion 55 依赖 + storage-query-datafusion crate + EntryLite "exposed in SQL"
- 不确定：SQL 查询面（journal/state 查询 API）的具体能力边界
- 证据：types/journal_v2/lite.rs:25-40（EntryLite SQL 用途注释）

## C. 已排除/低价值（Rejected）
**C-R07 单机模式（local_loglet）是主路径**
- 判定：非——replicated_loglet（sequencer+log_server）与 metadata-server 表明集群是主形态；local 是开发/单机部署形态
- 证据：providers/ 三 provider 并存 + metadata-store/server

## 跨项目连接候选
- C-R01 ↔ dapr-agents 幂等语义（10-25 考古）
- C-R02 ↔ Dapr actors（10-25）
- C-R03 ↔ substrate KO-S01（10-26 考古）+ Vercel/Notion/Lambda/MS（品类卡）
- 全部标 `Cross-project validation pending`，不升 Principle

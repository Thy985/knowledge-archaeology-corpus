# Restate 知识考古总览 — ARCH-2026-10-27-001

> 仓库：restatedev/restate（commit `db4b02e`，v1.8.2-dev，BUSL-1.1，Rust 35.3 万行）· run_id：ARCH-2026-10-27-001 · skill v3.2 · mode=initial

## 一句话定位
**分布式 durable execution 运行时**（Rust 实现）：把"有状态服务/虚拟对象/工作流/AI Agent"的执行拆成可持久化的 journal 原语，让任意基础设施故障下应用无需自写恢复逻辑即可恢复。与 Agent Substrate（整机快照-恢复）构成"恢复"两种哲学的另一半——**步骤持久化（journal）+ 崩溃后重放**。

## 六条认知主线
1. **Journal 即真相**：一切副作用（call/sleep/state/promise/signal）都写成 journal Command（20 种）+ Completion（结果回填）；已完成的 entry 重放时跳过（`is_resumable`）——执行确定性由 journal 次序保证，而非代码重跑
2. **分区-处理器权威模型**：数据按 PartitionKey 分片；每个分区有唯一 leader processor（epoch 标记），从 bifrost 日志 `last_applied_lsn` 起重放——权威可迁移（leadership 移交），状态不丢
3. **虚拟对象串行化（vqueues）**：Virtual Object = 按 key 串行的队列 + LockTable；同一 key 的并发请求排队，Scheduler 公平调度（RefillMode）——对象语义 = 单线程执行幻觉
4. **日志段抽象（bifrost）**：Loglet Provider 可换（local RocksDB / replicated sequencer+log server / memory）——单机到集群的持久化后端解耦；append 返回 Lsn 即持久化确认点
5. **重试与背压分层**：invoker 内 AttemptState（New/InFlight/WaitingRetry）+ RetryPolicy（OnMaxAttempts=Pause|Kill）；分区级 RuleBook 限流（UpsertRuleBook 走日志跨副本一致）；LimitKey 层次化并发控制
6. **BUSL-1.1 开放核心**：源码可用但"Public Restate Platform Service"（第三方注册自身服务经管理面调用）被禁止；4 年后转 Apache-2.0——商业护城河在托管层

## 关键数字
- 1242 个 .rs 文件 / 35.3 万行；196 个含 #[test] 文件；44 crates
- 核心 crate：worker 33944 行 / partition-store 22068 / bifrost 19342 / core 16091 / vqueues 11195
- partition-store 13 表族目录（invocation_status/journal(v2)/state/promise/outbox/timer/vqueue/lock/dedup/fsm/durable_lsn/snapshots…）
- 20 种 journal Command + 11 个 Completion struct（NotificationType 另含 Signal）
- GitHub：4609★/228 forks/created 2023-01/pushed 2026-10-26；v1.8.0(10-14)/v1.8.1(10-21)/v1.7.14(10-22)

## Corpus 跨项目连接
- **dapr-agents（10-25）**：Dapr DurableAgent 步骤持久化 vs Restate journal——同属"步骤持久化"恢复范式；Dapr 走 actor+state store 双存储，Restate 走单一日志+派生状态表
- **agent-substrate（10-26）**：整机快照 vs journal 重放——"恢复"两种哲学直接对照（周五 7-way 横向素材）
- **Vercel durable-harness / Notion Gen-6（10-25 记录）**：durable agent 品类主流化；Restate 是独立运行时侧代表
- **TeamMind（Java）**：Restate 提供 Java/Kotlin SDK——TeamMind durable state/恢复方案选型直接候选（KnowlegeMap 10-27 显式连接）

## 审计声明
- license 为 **BUSL-1.1 非 OSI 许可**（snapshot 已标注）——使用/部署须过 Additional Use Grant 条款
- 本考古以代码为准（journal_v2/processor/partition-store/bifrost/vqueues 均有文件:行锚）；docs/dev 为开发指南非产品文档，未作为实现事实

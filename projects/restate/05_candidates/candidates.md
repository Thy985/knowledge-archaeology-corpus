# Candidates — Restate（未验证假说 / 跨项目模式 / 待确认结论）

> 严格区分：Candidates 保持 Hypothesis 状态，不写成已验证知识。
> 每条标注：当前证据 / 缺失证据 / 验证路径。

---

## C-1（Fact，已由 IVR 澄清）InvocationTargetType 是两处独立枚举的名字撞车，非 types 缺变体
- **现象**：ingress-http/src/handler/invocation.rs:96-101 出现 `InvocationTargetType::IdempotencyId { ... }` 分支（URL 查询参数解析），而 types/src/invocation/mod.rs:148 的枚举只有 Service/VirtualObject/Workflow 三变体。
- **澄清（IVR F-3）**：ingress 侧 `crates/ingress-http/src/handler/path_parsing.rs:52-60` 定义了**独立的 `pub(crate) enum InvocationTargetType { InvocationId(String), IdempotencyId{name,target,handler,idempotency_id} }`**（URL 路径解析专用类型）；types 侧是执行语义类型。两类型同名、不同 crate、不同用途、不同变体集，无冲突。
- **状态**：Hypothesis → **Fact（CONFIRMED）**；不再需要审计。

## C-2（Hypothesis）journal_v2 双表并存期行为
- **现象**：老 journal（21 EntryType）与 journal_v2（20 Command+Notification）双表；do_drop_journal 按 pinned protocol version 分支删除。
- **当前证据**：`mod.rs do_drop_journal` 分支；journal_table / journal_table_v2 并存。
- **缺失证据**：同一次调用是否可能横跨两表（迁移中的调用）；v1.6 起默认 v2 的完整行为。
- **验证路径**：读 pre-flight / invoke 路径的 journal 选择逻辑（should_use_journal_table_v2）。

## C-3（Hypothesis）inbox 与 vqueue 的迁移完成度
- **现象**：suspend 挂起 vqueue entry 的 panic 分支提示 "Have you forgotten to migrate from the old inbox to vqueues?"；生命周期四阶段注释仍提及 inbox（仅 virtual-objects exclusive handler）。
- **当前证据**：`suspend.rs` panic；`mod.rs:767-772` 注释。
- **缺失证据**：inbox 在 v1.8.2 是否仍是活跃路径（或仅遗留兼容）；vqueue 是否完全接管。
- **验证路径**：grep inbox_table 读写调用点分布；读 vqueue 与 inbox 的调度分支。

## C-4（Hypothesis）LockState 锁的释放时机与死锁防护
- **现象**：锁模型存在（acquire/release_lock），但跨 key 锁（非本分区）的协调机制未在本次考古展开。
- **当前证据**：locks_table + CanonicalLockId + LockState。
- **缺失证据**：锁的 TTL/超时释放；跨分区锁如何协调（靠分区单线程？）。
- **验证路径**：读锁获取/释放的调用方（virtual object handler 执行前后）。

## C-5（Hypothesis）512 扇出 + 幂等分片的扩展瓶颈
- **现象**：UNSCOPED_SERVICE_PARTITION_KEY_FANOUT=512 硬编码；幂等服务侧不可变。
- **当前证据**：identifiers.rs:245-251（注释承认不可变约束）。
- **缺失证据**：512 是否成为单服务吞吐上限（每分区单线程 → 单服务最多 512 并发分区单元）；是否有动态扩桶计划。
- **验证路径**：查 issue/注释中关于 fanout 的讨论；sharding 分区分裂机制。
- **跨项目价值**：与 DBOS/Temporal 的分片扩展策略对照（Cross-project）。

## C-6（Hypothesis）ReplicatedLoglet 的读取一致性
- **现象**：ReplicatedLoglet 有 sequencer（本地/远程）+ log-server 复制；known_global_tail 有 caveat（启动默认 OLDEST，无本地 sequencer 时需 tail search）。
- **当前证据**：`replicated_loglet/loglet.rs` 头部注释 + known_global_tail 字段。
- **缺失证据**：副本间读一致性的精确保证（读已提交 vs 读尾）；故障切换窗口。
- **验证路径**：读 replicated_loglet/read_path + sequencer 状态机。
- **跨项目价值**：与 Raft/WAL 系统（etcd/BookKeeper）的复制读模型对照（Cross-project）。

---

## 认知状态汇总
| ID | 状态 | 是否跨项目候选 |
|---|---|---|
| C-1 | **Fact（IVR 澄清：两处独立枚举名字撞车，无冲突）** | 否 |
| C-2 | Hypothesis（迁移行为） | 否 |
| C-3 | Hypothesis（迁移完成度） | 否 |
| C-4 | Hypothesis（锁语义） | 否 |
| C-5 | Hypothesis（扩展瓶颈） | 是（跨分片设计） |
| C-6 | Hypothesis（复制一致性） | 是（跨日志系统） |

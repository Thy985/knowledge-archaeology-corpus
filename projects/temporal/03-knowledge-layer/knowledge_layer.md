# 03 · Knowledge Layer（Generalized KO）— Temporal（ARCH-2026-11-02-001）

> 窄尖顶：9 个 Generalized KO（L3/L4），每个声明 aggregation_rule（R1-R4）+ 簇内 EK + 边类型 + 解释范围扩大论证 + 反例攻击（详见 06）。
> 铁律：KO 必须可回溯到 EK 图；Hypothesis 不冒充 Fact；跨项目结论标 `Cross-project validation pending`。

---

## KO-01 · 确定性重放派：事件日志是状态真相，代码只是重放脚本（L4 认知模型）

- **聚合规则**：R2 因果链簇 — EK-01→EK-02→EK-03（Command→Event→Mutable State→重放恢复），边类型 causal。
- **内容**：在 Replay-based durable execution 中，**持久化的不是"执行"，而是"已发生事件的序列"**；程序逻辑在每次恢复时重新执行，但只对"日志中尚不存在的事件"产生副作用。状态 = 事件日志的函数。
- **解释范围扩大**：从 Temporal 单个系统扩展到所有 replay-based 引擎（Cadence、AWS SWF 同源）；解释"为什么 workflow 代码必须确定性"。
- **反例攻击**：见 06（Chasm 不重放整段代码而是持久化状态树——反例证明该模型是"一族"而非"唯一"）。
- **状态**：Validated Pattern（Temporal 代码直接证据 S3/S4；跨项目对照 Restate=journal 持久化派、agent-substrate=挂起恢复派）。

## KO-02 · 副作用隔离：确定性核心 + 不确定性出口（L3 Pattern）

- **聚合规则**：R1 机制簇 — EK-09（确定性约束）+ EK-10（Activity 状态机）+ EK-14（retry），边类型 mechanism/constraint。
- **内容**：Workflow 引擎把"必须确定性的代码"（workflow 逻辑）与"天然不确定的操作"（副作用）**分到两层**：workflow 层纯确定性，Activity 层承载 I/O 并有独立生命周期（retry/heartbeat/cancel）；任何不确定性在进入日志前被捕获为命令。
- **解释范围扩大**：适用于任何"可恢复计算"设计（actor 模型、event sourcing、CQRS 的写侧）；回答"为什么 workflow 不能直接调数据库"。
- **反例**：Chasm 的 skip-persistence 模式（EK-28）在特定场景允许不落盘——证明隔离有例外路径（性能换安全）。

## KO-03 · 两阶段提交式命令确认：worker 提议、服务端裁决（L4 认知模型）

- **聚合规则**：R2 因果链簇 — EK-01（Command→Event）→ EK-31（持久化层校验）→ EK-32（序列化+回滚），边类型 causal。
- **内容**：**执行者（worker）只提建议（command），权威者（history service）校验并裁决（event 追加）**。校验不通过（非法命令/状态冲突）则整个 batch 拒绝——worker 端没有自行落地状态的权力。
- **解释范围扩大**：这是"智能与权威分离"在分布式执行引擎的实例（对照 Tafcm/Codex 考古的 Intelligence≠Authority 原则）——执行方提议、控制方裁决、日志裁决结果。
- **反例**：Eager Activity（EK-17）允许 worker 响应中携带分派——部分是"执行者预授权"，但最终事件仍由 history 校验生成。

## KO-04 · 分片所有权 CAS：分布式状态写入的租约模型（L3 Pattern）

- **聚合规则**：R3 不变量簇 — EK-04（RangeID CAS）+ EK-31（状态校验）+ EK-05（VersionedTransition），边类型 constraint。
- **内容**：分片（shard）在同一时刻只有一个 owner；所有写操作携带 RangeID 条件（compare-and-swap），所有权变化后旧 owner 写入被拒（ShardOwnershipLostError）——**用"租约代际"防脑裂**，而非锁。
- **解释范围扩大**：分布式存储/队列/调度系统通用（Kafka partition leader、etcd lease、Postgres advisory lock 变体）；回答"两个节点同时写同一状态怎么办"。
- **反例**：standby 复制路径（EK-22/23）不经 CAS 直接应用复制事件——读路径/复制路径与写路径租约语义不同。

## KO-05 · 版本化代际（epoch/failover version）：跨故障语义的时钟（L4 认知模型）

- **聚合规则**：R3 不变量簇 — EK-05（VersionedTransition）+ EK-22（NDC 批处理）+ EK-23（failover）+ EK-26（Chasm requestID），边类型 mechanism/constraint。
- **内容**：分布式系统用**单调递增的代际号**（failover version / epoch / term）标记"由哪一代权威写入"，配合**幂等键**（request ID / transition count）让接收方区分"新事件"与"重放事件"——恢复与去重是同一种机制的两面。
- **解释范围扩大**：Raft term、Kafka epoch、etcd revision、Chubby sequencer 同构；回答"如何不重不漏地处理故障后重传"。
- **反例**：跨 namespace 复制（nsreplication）与 workflow 内部 transition 计数是两套代际——代际不是全局统一。

## KO-06 · 双引擎过渡：正交抽象层允许"旧系统即新系统的一个特例"（L3 Pattern）

- **聚合规则**：R4 主题簇 — EK-07/08/27/29/30（Chasm 核心）+ EK-28（开关矩阵）+ EK-30（workflow 作为 ASM），边类型 subsystem/mechanism。
- **内容**：Chasm 把"持久化状态机"抽象为通用框架（Tree/Node/Field/Transition），**Workflow 被重新实现为 Chasm 的一个 ASM**——新抽象通过"把旧系统重写为自己的实例"获得兼容性与迁移路径，而非推翻重建。
- **解释范围扩大**：适用于框架演进（LSP 的实例：子类型可替换父类型；Kubernetes 把 Docker 抽象为 runtime 后 Docker 成为其一个 runtime）。
- **反例**：Workflow 仍保留独立 MutableState/HSM 路径（EK-02/31），双路径并存（非纯单引擎）——迁移是渐进的。

## KO-07 · 运行时配置治理：特性开关是大型系统的安全网（L3 Pattern）

- **聚合规则**：R4 主题簇 — EK-25（dynamicconfig）+ EK-21（路由缓存）+ EK-28（Chasm 开关矩阵）+ EK-33（namespace 隔离），边类型 subsystem/mechanism。
- **内容**：大规模服务用**按约束（namespace/全局）分层的动态配置**承载"实验性特性"——新引擎（Chasm）的每个能力面都有开关（EnableChasm/ChasmSkipPersistenceEnabled/...），灰度粒度到 namespace，出问题可即时关闭。配置热更新 + 缓存 + 弱引用防泄漏是配套工程。
- **解释范围扩大**：任何"新旧并存、渐进迁移"的大型系统（feature flag 工程化：LaunchDarkly/Unleash 同构）。
- **反例**：动态配置本身引入新故障面（配置客户端轮询失败、约束匹配错误）——开关系统需要自己的监控。

## KO-08 · 任务队列 = 执行引擎的主动脉（L3 Pattern）

- **聚合规则**：R1 机制簇 — EK-18（队列处理器）+ EK-19（DLQ）+ EK-39（timer）+ EK-17（eager），边类型 mechanism/subsystem。
- **内容**：持久化执行引擎的"下一步做什么"不是进程内调度，而是**持久化任务队列**（transfer/timer/visibility）+ 队列处理器消费；失败任务进 DLQ 不丢；timer 是队列中的时间驱动条目——**调度本身被持久化**，因此崩溃后任务不丢。
- **解释范围扩大**：outbox pattern、事务消息、event sourcing 的写侧队列同构；回答"如何保证崩溃后定时任务仍触发"。
- **反例**：eager activity（EK-17）绕过队列（响应内联分派）——优化路径牺牲队列统一性换延迟。

## KO-09 · 可观测+可运维性内置：面向运维工程师的 API 设计（L3 Pattern）

- **聚合规则**：R1 机制簇 — EK-15（query）+ EK-16（update）+ EK-24（reset）+ EK-11（pause/reset activity）+ EK-12（CAN），边类型 mechanism。
- **内容**：Temporal 把"运维长时程执行"做成**一等 API**：运行中可查询（query）、可语义更新（update）、可重置（reset）、可暂停/恢复 activity、可收敛（continue-as-new）——不是事后日志分析，而是运行时操作面。
- **解释范围扩大**：长时程系统（agent 运行时、批处理平台、发布系统）应把运维动作设计为协议的一部分，而非运维脚本。
- **反例**：这些运维操作本身是高风险路径（reset 有 RequestID 幂等开关、update 有 accept/reject），复杂度转移到协议层。

---
## KO 聚合矩阵自检

| KO | 规则 | 簇内 EK | 边类型 | 可回溯 | 反例 |
|----|------|---------|--------|--------|------|
| KO-01 | R2 | EK-01/02/03 | causal | ✓ | ✓ |
| KO-02 | R1 | EK-09/10/14 | mechanism/constraint | ✓ | ✓ |
| KO-03 | R2 | EK-01/31/32 | causal | ✓ | ✓ |
| KO-04 | R3 | EK-04/31/05 | constraint | ✓ | ✓ |
| KO-05 | R3 | EK-05/22/23/26 | mechanism/constraint | ✓ | ✓ |
| KO-06 | R4 | EK-07/08/27/29/30/28 | subsystem/mechanism | ✓ | ✓ |
| KO-07 | R4 | EK-25/21/28/33 | subsystem/mechanism | ✓ | ✓ |
| KO-08 | R1 | EK-18/19/39/17 | mechanism/subsystem | ✓ | ✓ |
| KO-09 | R1 | EK-15/16/24/11/12 | mechanism | ✓ | ✓ |

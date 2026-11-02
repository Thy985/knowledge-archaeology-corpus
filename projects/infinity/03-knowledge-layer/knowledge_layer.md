# 03 · Knowledge Layer（Generalized Knowledge）— AWS Infinity

> 每个 KO 声明 `aggregation_rule`（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇），聚合自 02 的 EK Graph 簇。
> 升维铁律：单一项目证据 → 标 `Cross-project validation pending`，不冒充普遍定律。
> Epistemic Status：Validated Pattern（S4 测试/实现验证）→ Principle（跨项目验证缺口显式标注）。

## KO 清单（9 个）

### KO-01 · 挂起-恢复执行模型（stackless coroutine 语义）
**L4 Cognitive Model** · aggregation_rule: **R2 因果链簇**（EK-01→02→08→09→25）
**一句话**：把 agent 的异步工具调用建模为协程挂起点——等待中的 agent 是纯数据（历史），恢复 = 消息到达后重放 slice，因此"可完全关闭"是执行模型的推论而非事后优化。
- 簇内 EK: EK-01（slice 三阶段）→ EK-02（slice 永不阻塞外部）→ EK-08（消息全同质）→ EK-09（挂起=纯数据）→ EK-25（InputSender 即回环）
- 边类型: causal（01→02→09；08→02；25→02）
- 解释范围扩大: 从"Infinity 怎么跑"扩展到"任何 agent 系统如何实现 scale-to-zero"
- 反例攻击: Lambda 的 NoDeferral 会中断 pending call（EK-16）——"永不阻塞"是 per-slice 不变量，不是 per-turn；deferral 只是避免中断的优化
- Epistemic: Validated Pattern（S4：slice 结构有测试 + benchmark 实证；跨项目对照 temporal/restate 步骤重放派 = Cross-project validation pending）
- 对照: Restate/Temporal/DBOS = 步骤持久化重放（durable execution）；Infinity = 挂起-恢复（coroutine）。**两种资源经济学**：重放派吃"每次失败重跑历史"的成本，挂起派吃"常驻历史"的成本（154 KB/agent）。

### KO-02 · 幂等 + FIFO = 无锁并发（消息传递并发）
**L4 Cognitive Model** · aggregation_rule: **R3 不变量簇**（EK-06 + EK-07 + EK-05 + EK-41）
**一句话**：per-thread 有序投递 + 幂等消费让"每线程串行执行、跨线程自由并行"成为正确性推论——分布式并发正确性由队列排序保证，不需要锁。
- 簇内 EK: EK-05（turn buffer 时序）→ EK-06（processed IDs + history tail 幂等）→ EK-07（FIFO per group）→ EK-41（stopped 线程不响应）
- 边类型: constraint（05→06；06→07 因果）
- 解释范围扩大: 适用任何消息驱动的并发系统（actor 模型、事件溯源）
- 反例攻击: 工具结果无 durable ID，靠 history tail 去重——若结果先于其 tool call 落库（违反 EK-05 时序）会怎样？test `stale_result_does_not_flush_deferred_events` 证明运行时显式防止此序
- Epistemic: Validated Pattern（S4：dedup 语义在 HistoryManager 文档 + 测试）

### KO-03 · 状态转移 = 消息注入（不发明新状态机）
**L3 Pattern** · aggregation_rule: **R1 机制簇**（EK-13 + EK-14 + EK-15 + EK-16，共享 synthetic/事件注入机制，跨 ≥2 子系统）
**一句话**：让 LLM 理解"外部异步事件"的方式不是设计专门的提示框架，而是把它伪装成模型已训练过的工具调用-结果对（synthetic tool call）。
- 簇内 EK: EK-13（子报告）/ EK-14（receive_event__injected）/ EK-15（inline vs threaded）/ EK-16（deferral）
- 边类型: mechanism（13↔14↔15↔16 共享 synthetic 注入机制）
- 解释范围扩大: 任何"LLM 需要感知系统事件"的系统（agent runtime、事件驱动 agent）
- 反例攻击: `__injected` 命名是 hack——若模型学会主动调用它？测试/运行时拒绝路径（EK-14 明示 runtime rejects）
- Epistemic: Validated Pattern（S4：测试覆盖 synthetic 事件队列/去重）

### KO-04 · 上下文窗口是稀缺资源 → 治理即正确性
**L4 Cognitive Model** · aggregation_rule: **R4 主题簇**（EK-17 + EK-19 + EK-17b + EK-39）
**一句话**：在长会话 agent 中，context window 管理（compaction/overflow 恢复/大消息边界）不是性能优化，是正确性机制——处理不当会 wedge 线程（永卡）。
- 簇内 EK: EK-17（compaction 75% 触发）/ EK-17b（compaction×child）/ EK-19（ContextOverflow 恢复）/ EK-39（256MiB frame + 图像占位）
- 边类型: subsystem（全部属 context 治理子系统）+ constraint（19→17）
- 解释范围扩大: 长上下文 agent（RAG、编码 agent、客服）
- 反例攻击: 测试名 `oversized_tool_result_never_wedges_the_thread` 本身证明"可 wedge"曾是真实风险
- Epistemic: Validated Pattern（S4：4 个 compaction 测试 + overflow 测试）

### KO-05 · 平台可移植执行模型（4-trait 抽象）
**L3 Pattern** · aggregation_rule: **R1 机制簇**（EK-22 + EK-23 + EK-24 + EK-25 + EK-26）
**一句话**：把执行模型（turn/yield/threading/compaction/dedup）与平台 IO（store/queue/http/timer）分离为 4 个 trait，一套 core 跑通 serverless 与常驻进程。
- 簇内 EK: EK-22（4 trait）/ EK-23（Lambda 实现）/ EK-24（daemon 实现）/ EK-25（sender loopback）/ EK-26（session 持久化）
- 边类型: mechanism（22↔23↔24 同一抽象机制；25/26 是 22 的两个实例）
- 解释范围扩大: 任何"一次编写多处部署"的运行时（DBOS/step functions/actor 框架）
- 反例攻击: NoDeferral 证明平台能力差异必须显式建模（trait 方法返回"无法 defer"），不是隐藏
- Epistemic: Validated Pattern（S4：lambda+daemon 双实现 + 文档）

### KO-06 · 睡眠即一等公民（被动可中断）
**L3 Pattern** · aggregation_rule: **R1 机制簇**（EK-20 + EK-27 + EK-31）
**一句话**：agent 的"等待"作为显式工具暴露（sleep/sleep_until_event_or_input），且设计为 passive——事件到达立即处理，pending 结果稍后追加，不阻塞其他工作。
- 簇内 EK: EK-20（中断语义）/ EK-27（durable timers 双机制）/ EK-31（订阅生命周期）
- 边类型: mechanism（三者共享"被动等待+事件唤醒"机制）
- 解释范围扩大: 事件驱动 agent（监控、告警、市场监控）
- 反例攻击: `sleep_until_event_or_input` 是 no-op——真正等待的是"没有任何待办时 driver 空闲"，睡眠本身不占资源
- Epistemic: Validated Pattern（S4：sleep 工具测试 + 中断测试）

### KO-07 · 错误分类优先（provider-owned errors）
**L3 Pattern** · aggregation_rule: **R2 因果链簇**（EK-18 → EK-19 → EK-21）
**一句话**：让错误分类由产生错误的 provider 声明（Transient/Throttled/ContextOverflow/Fatal），运行时按类别选择恢复策略——运行时永不解析错误字符串。
- 簇内 EK: EK-18（ErrorClass）/ EK-19（ContextOverflow 恢复）/ EK-21（重启恢复）
- 边类型: causal（18→19→21）
- 解释范围扩大: 任何多 provider 的 LLM 网关 / 模型代理
- 反例攻击: 若 provider 分类错误（如把 ContextOverflow 标 Transient）→ 无限重试；边界是 provider 契约质量
- Epistemic: Validated Pattern（S4：ErrorClass 枚举 + overflow 测试）

### KO-08 · 并发 agent 的测试 = 竞态纪律
**L4 Cognitive Model** · aggregation_rule: **R4 主题簇**（EK-35 + EK-21 + EK-36 + EK-17b）
**一句话**：并发 agent 系统的正确性主要由"什么与什么能/不能同时发生"的测试揭示——测试名本身就是系统契约（compaction×tool call、stale result×deferred events、fatal error×sleeping child）。
- 簇内 EK: EK-35（测试纪律）/ EK-21（重启）/ EK-36（内存经济）/ EK-17b（compaction×child）
- 边类型: subsystem（都是"运行时行为契约"）
- 解释范围扩大: 并发/分布式系统测试设计
- 反例攻击: 28 个测试无法覆盖所有交错；snapshots 150 个证明 TUI 回归面广
- Epistemic: Validated Pattern（S4：tests.rs 2,199 行实测）

### KO-09 · 工具调用即并发边界（tool call = concurrency primitive）
**L4 Cognitive Model** · aggregation_rule: **R2 因果链簇**（EK-04 → EK-05 → EK-10 → EK-07）
**一句话**：agent 的并发性完全来自工具调用的异步化——spawn_thread 是工具、sleep 是工具、订阅是工具；并行 = 并行工具调用 + per-thread FIFO。
- 簇内 EK: EK-04（异步 RAP dispatch）/ EK-05（turn 时序）/ EK-10（spawn=同步工具）/ EK-07（FIFO）
- 边类型: causal（04→05→10→07）
- 解释范围扩大: "agent 如何并行"这一普遍问题
- 反例攻击: 同步工具 loop-back 证明"某些工具调用不能 yield"（微秒级操作不应产生异步往返）
- Epistemic: Validated Pattern（S4：spawn/sleep 工具测试）

## KO 聚合规则矩阵

| KO | aggregation_rule | 簇内 EK | 边类型 | 簇成立检查 |
|---|---|---|---|---|
| KO-01 | R2 因果链簇 | 01,02,08,09,25 | causal | 完整链（slice→yield→serverless）✓ 可命名 ✓ 可回溯 ✓ |
| KO-02 | R3 不变量簇 | 05,06,07,41 | constraint | 汇聚到"无锁并发"不变量 ✓ |
| KO-03 | R1 机制簇 | 13,14,15,16 | mechanism | 同一 synthetic 注入机制，跨 ≥2 子系统 ✓ |
| KO-04 | R4 主题簇 | 17,17b,19,39 | subsystem+constraint | context 治理主题互补维度 ✓ |
| KO-05 | R1 机制簇 | 22,23,24,25,26 | mechanism | 4-trait 抽象跨 2 生产实现 ✓ |
| KO-06 | R1 机制簇 | 20,27,31 | mechanism | 被动等待机制 ✓ |
| KO-07 | R2 因果链簇 | 18,19,21 | causal | 分类→恢复→重启链 ✓ |
| KO-08 | R4 主题簇 | 35,21,36,17b | subsystem | 测试揭示行为主题 ✓ |
| KO-09 | R2 因果链簇 | 04,05,10,07 | causal | 工具调用→并发结构链 ✓ |

## 升维纪律

- **全部 KO 标 `Infinity-originated · Strongly evidenced（S4）· Cross-project validation pending`**
- 无 S5（本环境无 Docker，无法运行验证）→ 禁止升 Principle/Law
- KO-01/02/04 为 L4 候选，其余 L3；无 L5（方法论需跨项目）

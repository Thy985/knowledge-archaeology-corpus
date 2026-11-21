# 03 · Knowledge Layer — Generalized Knowledge

> **升维规则**：只有解释范围超出 Graphiti 单项目、且能由 EK 簇支撑的内容才进入本层。所有 L3/L4/L5 均标注 `Cross-project validation pending`，不写成 Law。
> **KO 聚合规则**：每个 KO 声明 R1–R4、簇内 EK、边类型与“解释范围扩大”的论证。

## 0. Pattern Inventory（L3）

| ID | L3 Pattern | 来源 EK | 可迁移问题 |
|---|---|---|---|
| P-01 | Temporal validity interval | EK-01/02/04/09 | 如何表达“事实曾经为真、现在不为真” |
| P-02 | Event-time vs record-time separation | EK-01/03/05 | 如何同时支持回溯审计与当前视图 |
| P-03 | Model proposes, deterministic code disposes | EK-10/11/12 | 如何限制 LLM 对持久状态的直接权威 |
| P-04 | Soft invalidation / append-only facts | EK-07/09 | 如何保留历史而不物理删除 |
| P-05 | Read-time validity projection | EK-36/37/38 | 存储保留全部，读取按用途投影 |
| P-06 | Partition key is not an authorization boundary | EK-42/44/45 | 多租户记忆系统的隔离设计 |
| P-07 | Durable outbox for async acknowledgement | EK-24/25/26/27 | 异步接收如何不丢任务 |
| P-08 | Storage capability matrix | EK-17/18/19 | 统一抽象下的语义差异 |
| P-09 | Evidence anchor invariant | EK-15/29/30/31/35 | 派生事实如何保持可追溯 |
| P-10 | Request-scoped dependency bundle | EK-43/55 | 共享客户端并发安全 |
| P-11 | Bounded context window | EK-50/51 | 长记忆系统的上下文选择 |
| P-12 | Per-key serialization | EK-25 | 同实体/同分区写顺序 |
| P-13 | Candidate-set-bounded LLM dedupe | EK-14/15 | 大规模图上的去重成本控制 |
| P-14 | Provenance merge instead of edge duplication | EK-15/31 | 重复事实的合并语义 |
| P-15 | Tombstone vs cascade ambiguity | EK-33/34 | 删除源证据后的派生状态回滚 |

## 1. KO-01 · Bi-temporal Fact Ledger

- **Aggregation rule**: **R2 因果链簇**
- **Cluster**: EK-01 → EK-02 → EK-03 → EK-04 → EK-05 → EK-09 → EK-07
- **Edge types**: mechanism + causal
- **Explanation expansion**: 从“Graphiti 有四个时间字段”扩大到“任何需要表达事实有效期的系统，都应把事件时间与记录时间分开建模”。

### L1 — 工程事实
Graphiti 的 `EntityEdge` 携带 `created_at`、`valid_at`、`invalid_at`、`expired_at`；旧事实被取代时写 `invalid_at`/`expired_at`，不删除（EV-029–032, EV-036–038）。

### L2 — 工程知识
因为事实的“现实有效期”和“系统获知/废弃时间”不是同一个时间轴。若只保留 `created_at`，系统无法回答“这条事实在 2025 年是否成立”；若用删除表达失效，系统又失去审计轨迹。四时间轴让两种查询可以共存，代价是写入与读取都必须理解区间语义。

### L3 — 工程模式（P-01 / P-02）
**Temporal validity interval + event-time/record-time separation**。同类解法：Datomic 的 bitemporal facts、数据仓库的 SCD Type 2、事件溯源中的 `valid_time`/`transaction_time`。复用条件：事实会变化、需要回溯、且不能物理删除历史。

### L4 — 认知模型
> **事实的有效性是一个时间区间，不是一行记录的存在与否。**

### L5 — 方法论
1. 为每条事实记录至少两类时间：事件时间（何时为真）与记录时间（系统何时知道/废弃）。
2. 失效用区间结束表达，不用删除表达；删除只用于合规擦除，并单独建模。
3. 写入时明确时间锚（`reference_time`），不要让“抽取发生的时间”冒充“事实发生的时间”。
4. 查询时提供“当前视图 / 历史视图 / 审计视图”三种读模式，而不是一个默认语义。

### 反例与边界
- **反例 1**：`valid_at` 相同的新旧事实不会互相失效（EK-08），说明区间模型不能自动解决同一时刻的冲突。
- **反例 2**：未由 LLM 标记的矛盾不会进入时间算术（EK-10），区间模型本身不保证完整性。
- **反例 3**：`invalid_at` 不是数据库约束，任何直接写边路径都可以绕过（EK-16/EK-35）。
- **边界**：时区/精度未在代码中统一强制；跨时区部署需外部约定。

### Epistemic
`Pattern (Cross-project validation pending)`；L4 为 `Model (Cross-project validation pending)`。

---

## 2. KO-02 · Model Proposes, Time Arithmetic Disposes

- **Aggregation rule**: **R2 因果链簇**
- **Cluster**: EK-10 → EK-09 → EK-07，约束边 EK-11/EK-12
- **Edge types**: causal + constraint
- **Explanation expansion**: 从“Graphiti 用 LLM 判矛盾”扩大到“如何给 LLM 一个有限、可审计的状态变更角色”。

### L1 — 工程事实
去重 prompt 输出 `duplicate_facts` 与 `contradicted_facts`；只有后者进入 `_apply_temporal_invalidation`。prompt 明确禁止仅凭 recency、episode 日期、weekly snapshot 判矛盾（EV-042–045, EV-036–037）。

### L2 — 工程知识
因为“语义上是否矛盾”需要语言理解，而“把哪条记录在哪个时刻标记失效”必须是确定性状态变更。把 LLM 限制为“提名候选”，把算术交给代码，可以缩小模型错误的爆炸半径：模型最多选错候选，不能直接删除/覆盖数据。

### L3 — 工程模式（P-03 / P-13）
**Model proposes, deterministic code disposes**：LLM 输出受限的候选 ID/标签，状态机执行唯一合法转移。同类：human-in-the-loop 审批、分类器 + 规则引擎、推荐候选 + 排序器。复用条件：模型输出会影响持久状态，但状态转移可以被形式化。

### L4 — 认知模型
> **语义判断与状态变更必须分离：让模型提名，让代码裁决。**

### L5 — 方法论
1. LLM 的输出 schema 只允许候选 ID、枚举标签、置信度；禁止自然语言直接生成数据库操作。
2. 状态转移写成纯函数，并用时间区间/前置条件做二次校验。
3. 为“模型漏判”设计兜底扫描；为“模型误判”设计回滚/人工复核。
4. 记录模型判断的原始输入与输出，作为审计证据。

### 反例与边界
- **反例 1**：若模型漏判矛盾，代码不会失效旧事实，旧事实继续为“当前”（EK-10）。
- **反例 2**：若新事实比旧事实更旧，代码会把新事实结束在旧事实起点，而不是让模型决定（EK-06）。
- **反例 3**：同一 `valid_at` 的冲突双方都不失效（EK-08），说明“代码裁决”仍有未定义区间。
- **边界**：prompt 是软约束，模型可违反；代码只校验索引范围，不校验语义。

### Epistemic
`Pattern (Cross-project validation pending)`；L4 `Model (Cross-project validation pending)`。

---

## 3. KO-03 · Retention and Validity Are Independent Axes

- **Aggregation rule**: **R3 不变量簇**
- **Cluster**: EK-07 + EK-36 + EK-37 + EK-38 + EK-33
- **Edge types**: causal + constraint
- **Explanation expansion**: 从“Graphiti 不删除旧边”扩大到“可审计记忆系统的存储保留与读取有效性必须解耦”。

### L1 — 工程事实
失效路径不删除旧边（EV-038）。检索默认没有时间过滤（EV-066）。`search_results_to_context_string` 仍把带 `invalid_at` 的事实交给 LLM（EV-067–069）。用户可显式传 `valid_at`/`invalid_at` 过滤，但不是默认（EV-075）。

### L2 — 工程知识
因为“保留历史”与“当前为真”是两个不同需求：审计需要完整历史；对话需要当前事实。若把二者混为一个默认读语义，要么丢失审计，要么把过期事实当成当前事实。Graphiti 选择了“存储保留 + 读取时由调用方/LLM 解释”的折中。

### L3 — 工程模式（P-04 / P-05）
**Soft invalidation + read-time validity projection**：写入只追加/标记，读取按视图投影。同类：事件溯源 + 物化视图、SCD2 + 当前视图、账本 + 余额快照。复用条件：需要审计，但查询可以显式选择时间点/视图。

### L4 — 认知模型
> **保留（retention）与有效（validity）是两条独立轴；可审计系统必须同时表达两者。**

### L5 — 方法论
1. 提供明确的读模式：`current`（仅未失效）、`as_of(t)`（时间点）、`audit`（全部含失效）。
2. 默认读模式必须与产品语义一致；若默认是 `audit`，UI/Agent 必须显式处理 `invalid_at`。
3. 在 top-k 检索中考虑失效事实的挤占效应；必要时先过滤再排序。
4. 删除源证据时，明确派生事实的级联/墓碑策略，不要假设“未删除=未失效”。

### 反例与边界
- **反例 1**：默认检索不排除失效边（EK-37），旧事实可能进入上下文。
- **反例 2**：上下文只以文本标注 `invalid_at`，LLM 可能忽略；标注不是过滤。
- **反例 3**：`remove_episode` 不恢复它触发的失效（EK-33），删除不等于回滚。
- **边界**：本结论只描述代码默认值；部署方可自行加过滤层。

### Epistemic
`Pattern (Cross-project validation pending)`；L4 `Model (Cross-project validation pending)`。

---

## 4. KO-04 · A Partition Key Is Not an Authorization Boundary

- **Aggregation rule**: **R3 不变量簇**
- **Cluster**: EK-42 + EK-43 + EK-44 + EK-45 + EK-46 + EK-47 + EK-48
- **Edge types**: causal + constraint + contrast
- **Explanation expansion**: 从“Graphiti 用 group_id 分区”扩大到“多租户/多 Agent 记忆系统的命名空间与安全边界是两件事”。

### L1 — 工程事实
`group_id` 贯穿四类检索（EV-061/073）；非默认数据库时作为 database 名路由（EV-006）；只校验 ASCII（EV-078）；reference server 无认证/授权/调用方隔离；同一 group 同信任（EV-095–099）。

### L2 — 工程知识
因为 `group_id` 是数据组织维度，不是身份凭证。它可以让查询“只看见某个分区”，但不能阻止调用方指定另一个分区。把二者混同，会让“租户隔离”退化为“调用方自觉”。Graphiti 的 reference server 明确把认证/授权/网络控制外推给部署方，这是有意的边界，不是隐藏漏洞。

### L3 — 工程模式（P-06）
**Partition key is not an authorization boundary**：命名空间用于路由与聚合，授权必须在入口处执行。同类：S3 bucket 不是 IAM、数据库 schema 不是行级安全、Kafka topic 不是 ACL。复用条件：多租户、多 Agent、共享存储。

### L4 — 认知模型
> **隔离需要执行权威，而不是一个命名约定。**

### L5 — 方法论
1. 在 API 网关/服务入口做认证，在每次读写上做授权（subject → allowed group_ids）。
2. 不信任调用方传入的 `group_id`；从身份上下文派生或校验。
3. 对跨 group 的聚合/检索默认拒绝；显式管理员路径单独审计。
4. 把 `group_id` 当作数据标签，不要当作租户 ID 的唯一证据。

### 反例与边界
- **反例 1**：部署方在反向代理后加认证，可部分弥补；但那不是 Graphiti 内部保证。
- **反例 2**：单机单用户部署中，`group_id` 分区确实足够；风险随多租户出现。
- **反例 3**：`group_id` ASCII 校验可防注入，但不能防“合法地指定别人的 group”。
- **边界**：本文只评估开源 reference server；Zep 托管服务不在范围内。

### Epistemic
`Pattern (Cross-project validation pending)`；L4 `Model (Cross-project validation pending)`；与 `mem0`/`memmachine` 的对照待跨项目验证。

---

## 5. KO-05 · Durable Graph, Ephemeral Acknowledgement

- **Aggregation rule**: **R1 机制簇**
- **Cluster**: EK-24 + EK-25 + EK-26 + EK-27
- **Edge types**: causal + mechanism
- **Explanation expansion**: 从“MCP add_memory 立即返回”扩大到“异步接收语义的持久性必须与后端一致”。

### L1 — 工程事实
MCP `add_memory` 立即返回，后台处理；同 group 串行；队列是 `asyncio.Queue`；失败只 log + `task_done`；进程退出队列消失（EV-100–104）。

### L2 — 工程知识
因为 LLM 抽取/嵌入/图写入是慢操作，同步等待会让 MCP 调用超时；用内存队列把延迟移出请求路径。但代价是“接收”不再等于“持久化”：调用方收到成功，任务却只存在于进程内存。对记忆系统，这会造成“用户以为记住了，实际丢了”的静默失败。

### L3 — 工程模式（P-07 / P-12）
**Durable outbox + per-key serialization**：接收时先持久化任务（outbox/DB/队列），再返回；worker 按 key 串行消费，失败重试/死信。同类：Transactional Outbox、Kafka + consumer group、任务队列。复用条件：异步接收 + 至少一次处理 + 可观测状态。

### L4 — 认知模型
> **确认的持久性，不会超过它背后队列的持久性。**

### L5 — 方法论
1. 若返回“已接收”，先把 job 写入持久存储；内存队列只作为加速层。
2. 为每个 job 提供 idempotency key 与状态查询（queued/running/failed/done）。
3. 失败要有重试上限、死信队列与告警；不能只 log。
4. 按 `group_id`/实体键串行，避免同键并发破坏前序上下文。

### 反例与边界
- **反例 1**：直接调用库 `await add_episode` 的路径是同步的，不存在该问题；问题只在 MCP 异步路径。
- **反例 2**：若调用方不依赖 ack，或可重放，内存队列的丢失风险可被业务容忍。
- **反例 3**：MCP 客户端可能因断线重连重试，需服务端幂等才安全。
- **边界**：未运行队列，无法确认实际重启行为；结论来自结构与代码路径。

### Epistemic
`Pattern (Cross-project validation pending)`；`Hypothesis`：生产部署若直接使用该 MCP server，重启会丢未处理任务（需运行验证）。

---

## 6. KO-06 · Transaction Boundary Is a Storage Property

- **Aggregation rule**: **R1 机制簇**
- **Cluster**: EK-17 + EK-18 + EK-19 + EK-20 + EK-21 + EK-22
- **Edge types**: mechanism + contrast + causal
- **Explanation expansion**: 从“Graphiti 用 execute_write”扩大到“统一存储抽象不能统一事务语义”。

### L1 — 工程事实
`add_nodes_and_edges_bulk` 调 `execute_write`；Neo4j 有真实 `begin/commit/rollback`；FalkorDB/Kuzu/Neptune 的 `execute_write` 直接执行；bulk 分两阶段；saga/community 后置（EV-080–093）。

### L2 — 工程知识
因为 `GraphDriver` 抽象统一了 API，但没统一后端能力。Neo4j 的多语句写入可回滚；其他后端的 `execute_write` 只是“把函数跑一遍”，多语句中途失败会留下部分写入。把“execute_write”读成“事务”是跨后端最常见的语义误读。

### L3 — 工程模式（P-08）
**Storage capability matrix**：抽象层必须发布每个后端的原子性、隔离级别、批量写、索引、全文检索能力矩阵；调用方按能力选择，而不是按接口名假设。同类：SQLAlchemy 方言、缓存客户端抽象、文件系统抽象。复用条件：多后端可替换。

### L4 — 认知模型
> **事务边界是存储引擎的属性，不是接口名的属性。**

### L5 — 方法论
1. 在驱动接口上显式声明 `supports_transactions`、`atomic_multi_statement` 等能力。
2. 对不支持事务的后端，用幂等键 + 补偿/对账，而不是宣称原子。
3. 跨阶段写入要可恢复：记录 phase 状态，失败后重试/清理孤儿。
4. 集成测试必须按后端分别验证“中途失败后的状态”。

### 反例与边界
- **反例 1**：Neo4j 上主写入是原子的，所以“Graphiti 写入原子”在 Neo4j 部署中成立。
- **反例 2**：单条 Cypher 在部分后端可能自身原子；本文否定的是多语句 `execute_write` 的整体原子性。
- **反例 3**：上层可用 outbox/补偿实现最终一致，但不能称“事务”。
- **边界**：未运行任何后端；结论来自 `execute_write` 实现与 docstring。

### Epistemic
`Pattern (Cross-project validation pending)`；L4 `Model (Cross-project validation pending)`。

---

## 7. KO-07 · The Evidence Chain Is a Write-Path Invariant

- **Aggregation rule**: **R2 因果链簇**
- **Cluster**: EK-29 → EK-30 → EK-31 → EK-33 → EK-35 → EK-57，约束 EK-34
- **Edge types**: causal + constraint
- **Explanation expansion**: 从“Graphiti 有 episodes provenance”扩大到“派生事实的可追溯性必须由写入路径强制，而不是由字段存在暗示”。

### L1 — 工程事实
默认保存原始内容；关闭后只清空 `content`；`entity_edges`/`MENTIONS` 保存溯源；`add_triplet` 无真实 episode 时会写入一个**悬空 synthetic episode UUID**（不是空列表）；`remove_episode` 不恢复失效；summary 重建失败吞异常；LLM 上下文不含 edge/episode UUID（EV-016, 024, 027, 028, 051–053, 113–114; IF-13/14）。

### L2 — 工程知识
因为“边有一个 `episodes` 字段”不等于“每条边都有可回读的证据”。证据链依赖三个不变量：每条派生边必须有 episode；episode 原文必须可回读（或至少有内容哈希）；删除源证据时必须级联处理派生事实。Graphiti 允许配置/直接写/删除破坏其中任一环；更严重的是 `add_triplet` 会写入一个指向不存在 episode 的 UUID，使“字段非空”变成**伪溯源**。因此可追溯性是**可选强度**，不是默认保证。

### L3 — 工程模式（P-09 / P-14 / P-15）
**Evidence-anchored write path**：派生事实只能通过“源 → 抽取 → 边”的受控路径写入；源可有 tombstone；删除源时按策略级联或保留并标记证据缺失。同类：数据血缘、W3C PROV、事件溯源。复用条件：需要解释“这条结论从哪来”。

### L4 — 认知模型
> **派生边不是证据，除非有一条不可变/可回读的源 episode 锚定它；悬空 UUID 比空字段更危险。**

### L5 — 方法论
1. 写入 API 强制要求 `episode_uuid`，或显式标记 `derived_without_source` 并降级信任；**禁止追加未持久化对象的 UUID**。
2. 原始内容可配置保留，但至少保存内容哈希、来源 URI、抓取时间与长度。
3. 删除源时选择：级联删除派生、保留但标记 `evidence_missing`、或重建。
4. 摘要/聚合失败要 surface，不要静默保留旧摘要。

### 反例与边界
- **反例 1**：`source_description` 仍在，可能足以定位外部原文；但 Graphiti 不保证外部原文可回读。
- **反例 2**：`add_triplet` 的调用方可能自带 episode；传入后证据链成立。
- **反例 3**：`store_raw_episode_content=False` 是合规/隐私选择，不等于系统没有溯源元数据。
- **反例 4（Independent Validation）**：`add_triplet` 的 `episodes` 字段非空，但 UUID 无法解析；不能因为“非空”就判定有证据。
- **边界**：未验证 `remove_episode` 的实际图状态；结论来自代码路径。

### Epistemic
`Pattern (Cross-project validation pending)`；`Hypothesis`：关闭 raw content 后，跨会话无法重建 original evidence（需运行验证）。

---

## 8. KO-08 · Request-Scoped State, Bounded Fan-Out

- **Aggregation rule**: **R4 主题簇**
- **Cluster**: EK-06/43/49/50/51/55
- **Edge types**: subsystem + constraint + causal
- **Explanation expansion**: 从“Graphiti 修了 group_id 并发 bug”扩大到“有共享客户端的长生命周期对象，如何做请求级隔离与并发上限”。

### L1 — 工程事实
`_resolve_request_scope` 返回请求级 driver/clients，避免并发不同 group_id 覆写 `self.driver`；但该 scope **只用于 `add_episode` / `add_episode_bulk`**；`add_triplet`、`remove_episode`、`get_nodes_and_edges_by_episode` 与默认 `search` 仍使用 `self.driver`/`self.clients`；`max_coroutines` 是每个 `semaphore_gather` 的上限；前序窗口 10 条，可被 `previous_episode_uuids` 覆盖（EV-006, 007, 008, 076, 077; IF-16–19）。

### L2 — 工程知识
因为 `Graphiti` 是长生命周期对象，内部持有 driver/clients；多个 `await` 点之间，另一个请求可能改写共享状态（issue #1676 的真实故障模式）。把 driver/clients 作为请求级参数传递，消除了**两个 ingestion API** 中“共享可变状态 + 交错 await”的竞态。但这不是公开 API 的统一属性：跨库部署下，`add_triplet`/`remove_episode`/`get_nodes_and_edges_by_episode`/默认 `search` 仍可能落在默认库。并发上限按调用点设置，则允许不同阶段各自 fan-out，总并发是叠加的。

### L3 — 工程模式（P-10 / P-11）
**Request-scoped dependency bundle + bounded fan-out per stage**：不可变上下文随请求传递；每个阶段有独立并发预算；全局预算需另设。同类：请求上下文（Go context）、依赖注入 scope、Actor 每消息状态。复用条件：长生命周期客户端 + 并发请求；**但只有覆盖全部公开 API 才成立**。

### L4 — 认知模型
> **并发安全始于状态所有权，而不是始于加锁；状态所有权必须覆盖全部读写删路径，否则只是局部补丁。**

### L5 — 方法论
1. 禁止在请求处理中改写长生命周期对象的可变字段；改为返回局部副本/依赖束。
2. 每个阶段显式传入 `context`，不依赖隐式全局。
3. 区分“每调用点并发上限”和“全局并发上限”；分别配置与观测。
4. 上下文窗口大小、覆盖机制、排序规则都要成为显式配置。
5. **审计所有公开 API 的 driver 来源**；只要有一个 API 仍用默认 driver，就不能宣称“请求级隔离”。

### 反例与边界
- **反例 1**：默认数据库路径复用共享 driver 是安全的，因为没有改写；不能据此推断所有共享状态都安全。
- **反例 2**：`max_coroutines` 不限制跨调用点总并发；高 QPS 下仍可能过载。
- **反例 3**：`previous_episode_uuids` 绕过时间过滤，错误使用会引入未来/无关上下文。
- **反例 4（Independent Validation）**：`add_triplet`/`remove_episode`/`get_nodes_and_edges_by_episode`/默认 `search` 未接请求级 driver；非默认 group 的多库部署可能出现读空、写错库、删错库。
- **边界**：未做并发压测；结论来自代码与 issue 注释。

### Epistemic
`Pattern (Cross-project validation pending)`；L4 `Model (Cross-project validation pending)`。

---

## 9. KO Aggregation Matrix（v3.1 检查）

| KO | Rule | Cluster size | Edge types | Explanation expansion | Backtrace complete |
|---|---|---|---|---|---|
| KO-01 | R2 causal chain | 7 | mechanism/causal | 有 | 是 |
| KO-02 | R2 causal chain | 5 | causal/constraint | 有 | 是 |
| KO-03 | R3 invariant | 5 | causal/constraint | 有 | 是 |
| KO-04 | R3 invariant | 7 | causal/constraint/contrast | 有 | 是 |
| KO-05 | R1 mechanism | 4 | causal/mechanism | 有 | 是 |
| KO-06 | R1 mechanism | 6 | mechanism/contrast/causal | 有 | 是 |
| KO-07 | R2 causal chain | 7 | causal/constraint | 有 | 是 |
| KO-08 | R4 theme | 6 | subsystem/constraint/causal | 有 | 是 |

**Checks**
- KO average cluster size: **5.88**（在 3–12 内）
- Aggregation rule coverage: **8/8 = 100%**
- False aggregation (“same subsystem”)：**0**
- KO without EK backtrace：**0**
- Unverified hypotheses written as KO：**0**（KO-05/07 的运行时部分明确下放 `05_candidates.md`）

## 10. L5 Methodology Summary（可操作准则）

1. **先定义失效语义，再写存储**：区分现实有效期、系统记录有效期、删除/擦除。
2. **把 LLM 限制为候选提名者**：状态变更由确定性代码执行。
3. **把“保留”和“当前有效”分开读**：提供 current/as-of/audit 三种视图。
4. **不要用命名空间当安全边界**：授权在入口执行，group_id 只作数据标签。
5. **异步 ack 必须配持久 outbox**：否则“已接收”只是进程承诺。
6. **存储抽象必须发布能力矩阵**：原子性、批量、索引、全文检索按后端分别声明。
7. **证据链是写入路径不变量**：每条派生事实都要有可回读或可定位的源锚。
8. **并发安全先做状态所有权**：请求级依赖束优先于全局锁。

## 11. Cross-Project Status

- Pattern 级：`Strongly evidenced in Graphiti`，`Cross-project validation pending`。
- 与 `mem0`、`memmachine`、`letta-code` 的对照尚未执行；不得声称“Agent Memory 系统普遍如此”。
- 任何 L5 准则都是设计启发，不是行业定律。

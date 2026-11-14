# 00 · Overview — Graphiti Archaeology

- run_id: `ARCH-2026-11-15-001`
- project: **Graphiti** (`getzep/graphiti`)
- commit: `ed23196506eba8806fae6da831d73fcd2ad714d8` (`main`, 2026-11-14)
- version: `graphiti-core 0.31.2`
- license: `Apache-2.0`
- skill: `knowledge-archaeology v3.2`
- mode: `initial`
- date: 2026-11-15

## 1. Central Question

> **Graphiti 如何用“双时间事实图”保证记忆的权威性、可追溯性与作用域？为此付出了什么代价？**

拆成四个可证伪的子问题：
1. “双时间”到底由哪些字段、哪些写入路径、哪些时间锚构成？
2. 旧事实被新事实取代时，系统是删除、覆盖，还是保留并标记？谁来判定“矛盾”？
3. 检索是否默认只返回当前有效事实？如果不是，无效事实会以什么形式进入 LLM 上下文？
4. `group_id` 是不是安全边界？reference server 的信任模型是什么？

## 2. Answer in One Page

Graphiti 的核心不是一个向量库，而是**一个带证据边和时间有效区间的事实图**：

- 每个事实边（`EntityEdge`）同时携带 `created_at`（入库时间）、`valid_at`（事实在现实世界开始为真的时间）、`invalid_at`（事实在现实世界不再为真的时间）、`expired_at`（系统记录该事实被取代的时间）。这不是通常意义上的“双时间”两列，而是**四条时间轴**：知识时间、事件开始、事件结束、记录失效。
- 新事实进入时，LLM 先判断它是旧事实的重复、矛盾还是无关；若判为矛盾，代码再把旧边的 `invalid_at` 设为新边的 `valid_at`，把旧边的 `expired_at` 设为当前时间。**旧边不删除**，所以历史仍可审计，但默认检索仍可能返回它。
- 因此 Graphiti 的“权威性”不是数据库约束保证的，而是三层协作：**LLM 的语义判断 + 时间窗口算术 + 调用方对上下文的解释**。任何一层缺失，旧事实都可能继续作为“当前事实”出现。
- `group_id` 是分区键，且在非默认数据库时会触发 driver/database 路由；但**这种路由只覆盖 `add_episode` 与 `add_episode_bulk`**。`add_triplet`、`remove_episode`、`get_nodes_and_edges_by_episode` 与默认 `search` 仍使用默认 driver；跨库部署下可能读空、写错库、删错库。它更不是认证或授权——reference REST/MCP server 明确声明无认证、无授权、无调用方隔离、无 prompt-injection 防护。
- **证据锚有一个可复现的破口**：`add_triplet` 不传 episode 时会构造一个未持久化的 synthetic `EpisodicNode`，并把它的 UUID 追加到 `edge.episodes`。字段看起来有溯源，实际指向不存在的 episode（悬空 UUID），比空列表更危险。
- 持久化层存在明确的**驱动差异**：Neo4j 的 `transaction()` 有真实 commit/rollback；FalkorDB、Kuzu、Neptune 的 `execute_write` 只是立即执行函数，没有原子事务。所谓“主图原子写”只在 Neo4j 上成立。
- MCP server 的“立即返回”依赖一个**进程内 `asyncio.Queue`**：同 `group_id` 顺序执行，但队列不持久化、失败只记日志、进程重启即丢。图是持久的，接收确认不是。

一句话结论：

> **Graphiti 把“事实何时为真”与“系统何时知道/何时废弃”拆开保存，用软失效保留审计轨迹；但它把“哪条事实现在为真”的最终裁决权，留给了 LLM 判断、检索调用方和外部认证层。它的请求级隔离与证据锚都是“局部实现”，不是系统级保证。**

## 3. What This Project Teaches (preview)

| 认知层 | 预览 |
|---|---|
| L3 Pattern | 事实图应把“有效性”做成可查询的时间区间，而不是布尔删除位。 |
| L3 Pattern | 语义矛盾判断与时间簿记应分离：LLM 提议矛盾，确定性代码执行失效算术。 |
| L3 Pattern | 接收确认的持久性必须与图写入的持久性对齐；否则“已接收”只是进程内承诺。 |
| L4 Model | **保留（retention）与有效（validity）是两条独立轴**；可审计系统必须同时表达两者。 |
| L4 Model | **命名空间不是安全边界**；隔离需要独立的执行权威，而不是一个分区键。 |
| L5 Methodology | 设计记忆系统时，先定义“事实失效”的语义与时序，再决定检索默认过滤与审计保留策略。 |

## 4. Package Index

| File | Content |
|---|---|
| `00_overview.md` | 本文件：问题、结论、认知预览 |
| `01_project_layer.md` | 项目地图：架构、模块、生命周期、数据结构、配置、依赖、测试 |
| `02_engineering_knowledge.md` | Evidence Index（EV-001..EV-114）+ EK-01..EK-57（含 EK-41b，共 58 条）+ EK Graph |
| `03_knowledge_layer.md` | KO-01..KO-08，逐条 L1→L5 + 聚合规则 + 反例 |
| `04_flow_atlas.md` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy），**89 条 Edge** |
| `05_candidates.md` | 未验证假说 C-01..C-20、跨项目候选 XP-01..XP-05、需人工复核 HR-01..HR-07 |
| `06_validation/self-validation.md` | 自验证：Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic |
| `06_validation/independent-validation.md` | 独立 Auditor 盲重建：48 项判定、3 成功、3 错误、5 遗漏、Benchmark case |
| `06_validation/coverage-gaps.md` | 覆盖缺口与最小复现建议 |
| `06_validation/reconciliation.md` | 独立验证修正如何合并进 Package |
| `run_metadata.yaml` | 运行元数据与质量指标 |

## 5. Validation Summary（阶段⑤/⑥后）

| Dimension | Result |
|---|---|
| Self-validation | PASS WITH LIMITATIONS |
| Independent validation | PARTIALLY_CONFIRMED（32 confirmed / 6 partial / 1 downgraded / 1 over-generalized / 5 missing / 3 contradicted） |
| Core correction 1 | `add_triplet` 的 `episodes` 是悬空 UUID，不是空列表 |
| Core correction 2 | 请求级 driver 只覆盖两个 ingestion API，不是所有公开 API |
| Core correction 3 | `remove_episode` / `get_nodes_and_edges_by_episode` / 默认 `search` 未做 group→database 路由 |
| Runtime evidence | S4=0, S5=0 |


## 5. Epistemic Declaration

- **Fact**：直接来自 `ed23196` 的代码、配置、文档或本轮静态计数。
- **Observation**：由多条 Fact 组合出的行为描述，未运行验证。
- **Hypothesis**：代码路径暗示但未运行验证的结论，全部进入 `05_candidates.md`。
- **Pattern / Model / Principle**：跨出本项目解释范围的抽象，均在 `03_knowledge_layer.md` 标注 `Cross-project validation pending`，不冒充 Law。
- **未执行**：未启动服务、未连接图数据库、未调用 LLM、未运行测试。因此本 Package 的 S4/S5 证据数为 **0**。

## 6. Source Boundary

- 只读仓库快照：`/home/user/.doubao/agent_mode/workspace/archaeology-jobs/ARCH-2026-11-15-001/repo`
- 所有代码引用均指向该快照的 `ed23196`。
- 未使用 README 宣传语作为实现事实；README 只在需要区分“reference server”与“生产部署”时作为设计意图证据。

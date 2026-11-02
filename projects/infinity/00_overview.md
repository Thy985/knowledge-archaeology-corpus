# 00 · Overview — AWS Infinity（hydro-project/infinity）

## 一句话定位

Infinity 是一个 **Rust 实现的"大规模并发 agent 运行时"**：把编程语言的 **stackless coroutine（无栈协程）** 语义搬到 agent 执行模型上——agent 调用工具时 **yield（挂起）**，会话历史作为 continuation 保存，工具结果到达时 **resume（恢复）**；等待中的 agent 是"纯数据"（只有历史），无 task、无 stack、无连接，因此 **scale-to-zero 是默认行为**，75k 个 agent 可常驻在一台 Raspberry Pi 的内存里。

> "Infinity does for agents what async did for threads: instead of blocking on slow tools, Infinity agents run them concurrently, yield while they wait, and cost nothing until the next event arrives."（README.md）

## 核心命题（考古要回答的）

> **"挂起-恢复"是否可能成为 durable execution 的第三派？** 已考古的 Restate / Temporal / DBOS / Dapr Agents 都是"步骤持久化恢复"（失败后重放已持久化的步骤），Infinity 走的是另一条路：**每个 tool call = 挂起点，slice 有界执行后完全退出，等待零资源占用**。这不是"更好的重放"，而是完全不同的资源经济学——它把"hibernation"内建为执行模型的推论，而非事后优化。

## 三层速览

| 层 | 内容 | 状态 |
|---|---|---|
| **Project Layer** | 21 crates / 168 Rust 文件 / 57,187 行；agent-core（slice 三阶段）+ RAP 协议 + daemon/Lambda 双生产应用 | 完整（见 01） |
| **Engineering Knowledge** | 45 条 EK（EK Graph：slice 生命周期 / 幂等 / FIFO / synthetic events / deferral / compaction / 错误分类 / 平台 trait） | 完整（见 02） |
| **Generalized Knowledge** | 9 个 KO（挂起-恢复模型 / actor 式无锁并发 / 幂等优先 / synthetic tool calls / 平台可移植执行模型 / 上下文治理 / 失败恢复层次 / 内存=状态经济 / 事件驱动唤醒） | 完整（见 03） |

## 关键数字（全部来自仓库实际内容）

- 21 crates，168 个 .rs 文件，57,187 行 Rust（python3 逐文件统计）
- agent-core 15,372 行 / cli 10,294 / daemon 9,215 / sandbox 8,565（合计）
- docs 一手文档 6,507 行（.md/.mdx）
- 测试：2 个 `*_test.rs` + 29 个 `tests/` 目录 + 150 个 .snap 快照；agent-core/system/tests.rs 2,199 行
- benchmark（CHANGELOG）：50,000 agents × 20 turns = 2,000,000 completions / 255 s / 7.9 GB RSS / **≈154 KB per idle agent**
- 仓库规模：468★ / 48 forks / 54 open issues / 5,003 KB（GitHub API，2026-11-03）

## 核心机制（详细见 02 EK / 04 Flow Atlas）

1. **Slice 三阶段**：`prepare_input`（吸收/去重/路由）→ `run_completion`（流式补全）→ `execute_action`（派发工具调用，fire-and-forget）
2. **Synchronous tools loop-back**：`Tool::execute_synchronous` 内联执行 + 同 slice 回环（spawn_thread 即此例，避免 store 往返与竞态）
3. **幂等优先**：StateStore 按消息 ID 去重 + 工具结果按 history tail 去重 → at-least-once 投递安全
4. **FIFO per thread**：SQS FIFO MessageGroupId / 本地 mpsc channel → 无需分布式锁
5. **Synthetic tool calls**：`receive_event__injected` 把异步事件塞进模型可理解的历史结构（模型不可主动调用，runtime 拒绝）
6. **Deferral**：等待非被动工具调用时订阅事件/子报告暂存（NoDeferral / InMemoryDeferQueue / 自定义 durable）
7. **Compaction**：~3/4 context window 触发，摘要按 history index 定位，子线程按位置重建
8. **平台可移植**：4 个 trait（ConversationStore / StateStore / InputSender / HttpClient）→ Lambda（DSQL+DynamoDB+SQS）与 daemon（in-memory+JSON）同一套 core 代码

## 证据上限声明

- 本环境无 Docker/容器运行时 → **静态考古证据上限 S4**（代码+测试验证），不升 S5（真实运行验证）
- 全部事实来自仓库实际内容（源码/文档/测试/CHANGELOG）+ GitHub API，无第三方转述
- benchmark 数字为仓库 CHANGELOG 自述（S3-S4 级，非本环境复现），标注为项目自测

## 交付物清单

```
00-overview/overview.md
01-project-layer/project_layer.md
02-engineering-knowledge/ek_graph.md        # 45 EK + 六类边 + links
03-knowledge-layer/knowledge_layer.md      # 9 KO + aggregation_rule R1-R4
04-flow-atlas/flow_atlas.md               # 七类流（Control/State/Data/Evidence/Authority/Memory/Policy）
05-candidates/candidates.md               # 未验证假设 + 跨项目候选
06-validation/validation_report.md        # Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic
run_metadata.yaml                         # run_id/commit/skill_version/metrics/validation
```

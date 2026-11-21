# Job Manifest — ARCH-2026-11-02-001

## 1. 候选排序（按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 考古状态 / Benchmark 学习价值）

| 排名 | 项目 | 仓库 | 状态 | 体量 | 排序理由（非 star 排名） |
|---|---|---|---|---|---|
| **1** | **Temporal** | temporalio/temporal | **NEW（未考古）** | 174MB / Go / 24k★ | durable execution **品类鼻祖**；确定性重放（Replay-based）执行模型，与已考古 Restate（journal 步骤持久化）/agent-substrate（挂起恢复）构成**第三种执行模型**的 Benchmark 对照；10-15 出 agent 适配页（llm-info），Agent-AI 相关性高；1.32.1（10-09）近期发布；Corpus 连接极强（restate/dapr-agents/dbos/agent-substrate 均已考古，Temporal 是 durable 品类最后一块核心拼图）；Benchmark 学习价值最高 |
| 2 | DBOS Transact (TS) | dbos-inc/dbos-transact | NEW（未考古 TS 本体） | 6.7MB / TS | Postgres 原生 durable runtime；但 dbos-transact-py 已考古（10-28），TS 本体增量有限 |
| 3 | Vercel durable-harness | vercel-labs/durable-harness | NEW | 2.9MB / TS | harness 包装层（durable log 包装既有 agent）；价值 B 级，体量小但知识增量有限 |
| 4 | Obelisk | obeli-sk/obelisk | NEW | 27MB / Rust / AGPL-3.0 | durable agents + layered sandboxes；766★ 信号弱；AGPL 许可 |
| 5 | Mastra | mastra-ai/mastra | NEW | **1.75GB** / TS / 29k★ | Agent 框架 durable 化信号强，但 1.75GB monorepo 体量**超出单日深度考古容量**（无 Docker 环境，静态考古不可行） |
| — | AWS Infinity | aws/aws-infinity（Hydro） | **已考古（ARCH-2026-10-29-001）** | — | 分支 arch/infinity-2026-10-29 已含完整产物 → 排除 |
| — | Restate | restatedev/restate | 已考古（ARCH-2026-10-27-001 + 11-01 轮） | — | 11-01 轮刚考古 → 排除（本轮不重复） |
| — | Dapr Agents | dapr/dapr-agents | 已考古（ARCH-2026-10-25/30） | — | 已考古 + 10-30 validated → 排除 |
| — | DBOS (py) | dbos-inc/dbos-transact-py | 已考古（ARCH-2026-10-28） | — | 已考古 → 排除 |

## 2. 选中项目

- **job_id**：ARCH-2026-11-02-001
- **project**：Temporal
- **repository**：https://github.com/temporalio/temporal.git
- **mode**：initial
- **priority**：P0

## 3. Reason（详细）

Temporal 是 durable execution 的**开山鼻祖与工业标准**（Go 实现 Temporal Server，24k★，Temporal Server 1.32.1 于 2026-10-09 发布）。其核心执行模型是 **Replay-based（确定性重放）**：workflow 代码通过 workflow-determinism 约束（时间/Sleep 用 Workflow.Sleep、Timer 用 timer、外部 I/O 走 activity/signal/query 门），执行历史（Event History）持久化，worker 崩溃后从事件流**重放** workflow 恢复状态，重放不重新执行副作用。这与已考古的：
- **Restate（ARCH-2026-11-01-001）**：journal_v2 步骤持久化（每次 handler 执行切 journal 条目，崩溃从最后完成 entry 恢复）——「步骤持久化派」
- **agent-substrate（ARCH-2026-10-31-001）**：suspend/resume 挂起恢复派

构成 durable execution 的**三大执行模型对照**：Replay-based（Temporal）/ Journal-append（Restate）/ Suspend-Resume（agent-substrate/Infinity）。Temporal 是其中**唯一在运行前就已完全确定 workflow 代码路径**、用 Event History 重放恢复的系统——这对理解「如何在不重放副作用的前提下重放状态」是最高价值的学习对象。

Agent-AI 相关性：Temporal 于 2026-10-15 上线 llm-info agent 适配页（面向 AI crawlers/assistants），并出现 agent 化采用信号；其 SDK 层（Go/TS/Python/Java）是 Agent 长时程任务（durable workflows、human-in-the-loop、signal-driven agent）的成熟载体，与用户 TeamMind（Java 多 Agent 运行时）直接相关（Temporal 有 Java SDK）。

## 4. source_entry

- KnowlegeMap `03_expansion_queue/candidates/[cand]durable-execution-category-mainstreaming-2026-10.md`（2026-10-27 建卡，Runtime 周 Day2 增量）：
  - FACT（Temporal agent 适配页，2026-10-15）：Temporal llm-info 页面向 AI crawlers/assistants（https://temporal.io/llm-info）
  - FACT（Temporal Server 1.32.1，2026-10-09）
- KnowlegeMap `07_radar_watch/current_focus.md`：Runtime 周（2026-10-26 起）领域树 A 层「Durable Execution 与失败恢复」；durable 品类 9 向横向（10-30）已覆盖 Temporal 行（Replay 语义未详→本轮深度考古补齐）
- KnowlegeMap HEAD：f2d6443（2026-10-30 radar watch，Runtime Day5 验证完成）

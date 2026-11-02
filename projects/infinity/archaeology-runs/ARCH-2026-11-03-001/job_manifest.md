# Job Manifest — ARCH-2026-11-03-001

| 字段 | 值 |
|---|---|
| **job_id** | ARCH-2026-11-03-001 |
| **project** | AWS Infinity（hydro-project/infinity） |
| **repository** | https://github.com/hydro-project/infinity.git |
| **mode** | initial |
| **priority** | A（Runtime 周第二周，高 Benchmark 学习价值） |
| **run_id** | ARCH-2026-11-03-001 |

## 候选排序（按 SOP 排序标准，不按 star）

| 排名 | 项目 | 状态 | 理由 |
|---|---|---|---|
| **1** | **AWS Infinity（hydro-project/infinity）** | **NEW** | **durable execution 品类第三派恢复语义的仓库级代表（stackless coroutine 挂起/恢复）**：与已考古 Restate（步骤持久化）/ Temporal（replay）/ DBOS（Postgres 事务）/ Dapr Agents（步骤持久化）构成完整对照系；Novelty 高（research early release，slice 语义 + synthetic tool calls + serverless step 模式）；增长信号强（2026-10-23 Apache 2.0 开源，AWS×UC Berkeley，10-29 活跃推送，468★）；Rust 语言栈补齐 corpus；Benchmark 学习价值极高（75k 并发/8GB 基准）；未考古 |
| 2 | durare（crates.io） | QUEUED | DBOS 兼容 Rust SDK（品类第 10 向）；但 DBOS 语义已由 dbos-transact-py 考古覆盖，小 SDK 增量价值有限，暂缓 |
| 3 | semarun（PyPI） | QUEUED | Python semantic-checkpoint 第三派候选；生态未明（PyPI 自述），无明确成熟仓库，观察位 |
| 4 | Mastra | QUEUED（排除） | 1.75GB TS 超考古容量，维持排除 |

## 选中理由（A 级）

1. **执行模型对照价值（Benchmark 学习价值）**：Infinity 是"挂起恢复"派（slice 有界执行 + serverless 卸载，等待零计算，挂起 agent 只占 history 内存）的仓库级代表，与 corpus 已考古的"步骤持久化恢复"派（Restate / Temporal / DBOS / Dapr Agents）构成**两派互补对照**——步骤持久化保证失败后不重复已完成调用，挂起模型保证等待期间零资源占用。这是 Knowledge Value 最高的单一增量。
2. **Agent-AI Engineering 相关性（最高）**：actor-style 线程模型（每会话独立 input queue、线程间仅消息、FIFO 串行化免锁）+ synthetic tool calls（`receive_event__injected`）+ spawn_thread/report_to_parent 父子通信，直接对应 TeamMind（用户 Java 多 Agent 运行时）并发/事件模型设计问题。
3. **Novelty 高**：research early release，把编程语言 stackless coroutine 机制搬到 agent 执行，10 月刚开源，corpus 无同类。
4. **增长信号 / 近期发布**：2026-04 创建，10-23 开源，10-29 活跃推送（5MB 小规模，考古容量安全）。
5. **与已有 Corpus 连接**：restate（ARCH-2026-11-01）/ temporal（ARCH-2026-11-02）/ dbos-transact-py / dapr-agents / substrate 全部在库，Infinity 可与其形成"恢复语义两派 + 三路线"横向闭环。

## source_entry

- **卡片**：`03_expansion_queue/candidates/[cand]aws-infinity-agent-runtime-2026-10.md`（状态 `[cand]`，B 级，2026-10-29 建卡，2026-10-30 验证日回填三路线对照，决策=维持观察）
- **当前领域**：Runtime（Agent 执行运行时），第二周（11-02~11-08），`07_radar_watch/current_focus.md` active
- **本周计划**：周五 11-07 Agent Substrate 复现验证（头号任务，缺口②）；缺口④⑤（TeamMind 权限对照 / Agent Mesh 身份层）
- **KnowlegeMap HEAD**：60e852c（2026-11-03 radar watch 增量：durare 第 10 向 + semarun 第三派观察 + AgentCore 14 天会话）

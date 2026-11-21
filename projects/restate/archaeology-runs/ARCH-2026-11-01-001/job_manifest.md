# Job Manifest — ARCH-2026-11-01-001

## 候选排序（按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 考古状态 / Benchmark 学习价值；不按 star）

| 排序 | 项目 | 状态 | 理由 |
|---|---|---|---|
| 1 | **Restate**（restatedev/restate） | **NEW → 选中** | durable 品类「独立运行时侧」代表（每步 virtual-object 持久化、从最后完成步骤恢复）；agent 原生定位 + Replit 全量采用（10 月上旬）；Java/Kotlin SDK 与 TeamMind 最强连接；与上轮考古 agent-substrate（挂起恢复派）构成「步骤持久化 vs 挂起恢复」路线对照；未考古 |
| 2 | AWS Infinity（aws/…Hydro） | NEW | 第三条 runtime 设计线（slice/serverless 挂起恢复）；但 research early release、GitHub robots 限制 + 无容器环境，仓库级复现未做 → 维持观察，不适合本轮深度考古 |
| 3 | DBOS（dbos-inc） | NEW | PostgreSQL 原生 durable execution（第 8 向候选）；三强格局中 Restate 的 agent 定位 + Java SDK 连接更强，故次之 |
| 4 | Temporal（temporalio/temporal） | NEW | durable execution 鼻祖（agent 适配页 10-15）；仓库体量过大（C++/Go 混合 + 长期项目），Benchmark 学习价值低于 Restate 的单体机制对照 |
| 5 | Dapr 本体 | REFRESH_REQUIRED（间接） | Dapr Agents（PR #59 已考古）底层平台 v1.19 增量；非独立目标，并入 durable 品类认知即可 |

## 选中项目
- **job_id**: ARCH-2026-11-01-001
- **project**: restate
- **repository**: https://github.com/restatedev/restate.git
- **mode**: initial
- **priority**: P0
- **reason**:
  1. **Runtime 周闭环补全**：上轮（10-31）考古 agent-substrate（checkpoint 快照 + BootID 幂等 = 挂起恢复派）；本轮考古 Restate（virtual objects 每步持久化 = 步骤持久化派）——两条恢复语义路线在 corpus 形成直接对照，补齐 Runtime 能力树 A 层（Durable Execution 与失败恢复）的关键旁支。
  2. **TeamMind 连接**（用户 Java 多 Agent 运行时项目）：Restate 提供 Java/Kotlin SDK 且被 Replit 全量采用——是 TeamMind 生产化 durable state/恢复方案选型的直接输入（10-30 验证日结论："Java 首选 Restate"）。
  3. **Benchmark 学习价值**：Restate 的 virtual objects（无锁状态）、durable execution 日志/快照机制、keyed 调度 vs agent-substrate 的 worker 池调度——机制级对照素材。
  4. **增长信号**：10 月上旬 Replit 采用为证 + agent 原生定位强化；未考古、未克隆。
- **source_entry**: KnowlegeMap `03_expansion_queue/candidates/[cand]durable-execution-category-mainstreaming-2026-10.md`（§3 TeamMind 连接 + §5 9 向表 Restate 行）+ `07_radar_watch/current_focus.md`（Runtime 周锚点）

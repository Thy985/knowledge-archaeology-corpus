# durare 知识考古索引

> 一句话定位：**DBOS 兼容 durable-execution Rust 库**（社区维护，SamuelXing/durare）——普通 async 函数每步 checkpoint 到 Postgres/SQLite/内存后端，崩溃/重启/重新部署后从精确断点恢复；无服务器、无 sidecar，引擎是进程内库直连数据库，与 DBOS Transact SDK 跨语言在同一 Postgres 上字节级互操作。

## 考古信息

- 仓库：https://github.com/SamuelXing/durare.git（commit `87ed2db`，v0.5.0 2026-11-01，MIT OR Apache-2.0，Rust MSRV 1.88，~62k 行 / 61 集成测试文件）
- 考古日期：2026-11-04 · run_id：`ARCH-2026-11-04-001` · skill：knowledge-archaeology v3.2
- 来源：KnowlegeMap Runtime 第二周 Day3 雷达（与 dbos-transact-py 同生态跨语言对照；与官方 dbos-transact-rust 显式 context vs task-local 设计对照）

## 产物清单

| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 考古元数据（commit/run/summary/判定统计） |
| `00_overview.md` | 总览：一句话定位 + 10 条认知主线 + 三层配比 + 证据上限声明 + 最争议设计 |
| `01-project-layer/project_layer.md` | 项目地图：两层架构/九状态机/模块/生命周期/配置/测试/治理 |
| `02-engineering-knowledge/ek_graph.md` | 52 条 EK（EK Graph 六类边，平均出边 2.15，游离 0，含 Reconciliation 补充 EK-51/52） |
| `03-knowledge-layer/knowledge_layer.md` | 10 个 KO（R1×2/R2×4/R3×3/R4×2 + 聚合规则矩阵，KO-04 经盲重建修正） |
| `04-flow-atlas/flow_atlas.md` | 七类流 23 条 Edge（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05-candidates/candidates.md` | 6 候选（4 Hypothesis + 1 安全敏感 NEEDS_HUMAN_REVIEW + 1 Cross-project） |
| `06-validation/validation_report.md` | 阶段④ Validation 设计 + 反例预算（14 攻击：3 修正 + 2 降级）+ Epistemic 自检 |
| `06-validation/independent_validation_report.md` | 阶段⑤ 独立盲重建报告（判定统计 + 3 成功 + 3 错误 + 遗漏 + Benchmark case） |
| `archaeology-runs/ARCH-2026-11-04-001/` | 本轮 run 原始产物（job_manifest + snapshot_artifact + package + run_metadata） |

## 核心认知（10 KO 速览）

1. **KO-01 位置即历史**：checkpoint 位置在 durable call 构造时分配，与执行顺序解耦——并发 body（select!/join! 随机唤醒）不会调换历史中两个调用的位置。
2. **KO-02 两层编排 + 单一 seam**：workflow orchestration（重放决策）与 runtime orchestration（所有权/恢复）经 execution.rs 单一 seam 交接，新行为按决策表归层。
3. **KO-03 控制错误不是业务失败**：RecoveryRequired/ObservationFailed 类型级隔离、不可被 body 捕获消化；ObservationFailed 不调度恢复。
4. **KO-04 数据库即仲裁者**：CAS/ON CONFLICT/CTE/事务+行锁的原子原语族——仲裁决定权在 DB 不在应用锁。
5. **KO-05 事务步骤原子消除 at-least-once window**：SQL 写与 checkpoint 同一 DB 事务；普通 step 保留一步宽 window。
6. **KO-06 跨 SDK 字节级兼容协议**：portable_json + golden strings pinned in every SDK suite + 时间戳 BIGINT + serialization 列格式分发。
7. **KO-07 版本门控部署安全**：app_version=可执行 sha-256；dequeue/recovery 只认领同版本行。
8. **KO-08 确定性执行纪律**：replay 安全靠 API 形状前置（拒绝嵌套/spawned task），UnexpectedStep 只是 backstop。
9. **KO-09 动态 SQL 不变量外部强制**：bind parameter 唯一通道 + EVIL sweep 外部测试，防重构腐化。
10. **KO-10 崩溃一致性测试方法论**：边界枚举 + 精确 effect 计数；vacuous pass 防伪。

## 跨项目连接

- **同生态对照**：dbos-transact-py（corpus 已含）——同一 DBOS 语义两套实现，schema column-for-column、wire byte-identical。
- **设计对照**：官方 dbos-transact-rust（upstream commit `7937413`）——显式 context vs task-local；放置规则差异表（EK-39）。
- **相关性**：Restate/Temporal（open PR 待合入）——同为 durable execution 生态，position/重放语义可做 cross-project 对照（C-06 候选）。

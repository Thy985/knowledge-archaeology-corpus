# Restate 知识考古索引

> 一句话定位：**分布式 durable execution 运行时**——把有状态服务/虚拟对象/工作流/AI Agent 的执行拆成可持久化 journal 原语，基础设施故障下自动恢复；"步骤持久化"恢复范式代表（与 agent-substrate 整机快照构成两种哲学互补）。

## 考古信息

- 仓库：https://github.com/restatedev/restate.git（commit `db4b02e`，v1.8.2-dev，2026-10-26，BUSL-1.1 非 OSI）
- 考古日期：2026-10-27 · run_id：`ARCH-2026-10-27-001` · skill：knowledge-archaeology v3.2
- 品类卡：durable-execution-category-mainstreaming-2026-10（KnowlegeMap 10-27 radar watch B 卡）

## 产物清单

| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 考古元数据（commit/run/summary/判定统计） |
| `00_overview.md` | 总览：六条认知主线、关键数字、Corpus 连接、审计声明 |
| `01-project-layer/project_layer.md` | 项目地图：架构/模块/数据结构/生命周期/状态机/测试/配置/依赖 |
| `02-engineering-knowledge/ek_graph.md` | 24 条 EK（EK Graph 六类边 + 证据锚） |
| `03-knowledge-layer/knowledge_layer.md` | 8 个 KO（R1×2/R2×2/R3×2/R4×2 聚合规则矩阵） |
| `04-flow-atlas/flow_atlas.md` | 七类流 22 Edge（Control/State/Data/Evidence/Authority/Memory/Policy）+ Flow→KO 交叉校验 |
| `05-candidates/candidates.md` | 7 候选（跨项目假说 C-R01..03 / 暂定 C-R04..06 / 已排除 C-R07） |
| `06-validation/validation_report.md` | 六审计 + 质量指标 + Reconciliation |
| `06-validation/independent_validation_report.md` | 独立 Auditor 盲重建报告（13 CONFIRMED/2 CONTRADICTED/1 DOWNGRADED/4 MISSING/1 NEEDS_HUMAN_REVIEW） |
| `archaeology-runs/ARCH-2026-10-27-001/` | 本轮原始产物（job_manifest / snapshot_artifact / run_metadata） |

## 关键发现（Top 6）

1. **Journal 即真相**：20 种 journal Command + 11 个 Completion struct；完成标记 is_resumable 使恢复重放跳过已完成 entry——执行确定性由 journal 次序保证，非代码重跑
2. **分区-领导者权威模型**：PartitionKey 分片 + 唯一 leader（epoch）+ last_applied_lsn 重放；权威可迁移、状态不丢
3. **虚拟对象串行化**：vqueues 按 key 排队 + lock_table 持久锁 + Scheduler RefillMode 公平调度——单线程执行幻觉
4. **bifrost 日志段抽象**：local RocksDB / replicated sequencer / memory 三 provider 可换；append→Lsn 即持久化确认点
5. **重试与背压分层**：attempt 级 RetryPolicy（OnMaxAttempts Pause|Kill）+ 分区级 RuleBook 限流（日志一致）+ LimitKey 层次化并发
6. **BUSL-1.1 开放核心**：禁 Public Restate Platform Service；4 年后转 Apache-2.0——商业护城河在托管层

## 审计声明
- license=BUSL-1.1 **非 OSI**；使用/部署须过 Additional Use Grant 条款
- 以代码为准（file:line 锚）；docs/dev 为开发指南非实现事实
- 独立 Auditor 盲重建发现 2 处计数错误（Command 17→20、Completion 12→11）已 Reconciliation 修正

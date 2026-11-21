# DBOS Transact — Knowledge Archaeology Index

**一句话定位**："数据库即日志"的轻量 durable execution（PostgreSQL 原生路线）——不搭编排服务、不用专用日志段，把 workflow 状态/步骤结果全部存数据库表，用数据库事务原子性做 checkpoint，故障后从最后完成步骤重放。

| 字段 | 值 |
|---|---|
| Repository | https://github.com/dbos-inc/dbos-transact-py |
| 考古 run | ARCH-2026-10-28-001（initial） |
| Commit | `47379ca717ec93ef7cd853ad7b43169c4e5cf530`（main，2026-10-27 "Serialize concurrent SQLite migrations (#923)"） |
| Version | 3.3.0aN（pdm SCM dynamic，main preview） |
| 考古日期 | 2026-10-28 |
| Skill | knowledge-archaeology v3.2 |
| License | MIT（SDK）；DBOS 核心 runtime 闭源 |

## 产物清单
- `00_overview.md` — 六条认知主线 + 关键数字 + Corpus 连接（restate 存储哲学二分）+ 审计声明
- `01_project_layer.md` — 项目地图（架构/核心模块/数据结构/生命周期/状态机/测试/配置/依赖）
- `02_engineering_knowledge.md` — EK Graph（24 条，links 六类边，平均出边 2.38）
- `03_knowledge_layer.md` — Generalized KO（8 个，aggregation_rule R1-R4 全覆盖，L3×2/L4×5/L5×1）
- `04_flow_atlas.md` — 七类流（34 Edge 可回溯）+ Flow→KO 交叉校验 8/8
- `05_candidates.md` — 跨项目假说 3（幂等三层组合/虚拟对象范式/存储介质运维面）+ 暂定 3 + 已排除 1
- `06_validation.md` — 六审计结果 + 质量指标 + 待 IVR 攻击面
- `archaeology-runs/ARCH-2026-10-28-001/` — run 原始产物（job_manifest/snapshot_artifact/run_metadata/IVR）

## 核心发现（Top 3）
1. **owner_xid 事务级执行权**：数据库事务即互斥锁——所有状态写 `WHERE owner_xid=token`，失权→park 轮询（KO-D01/D03）
2. **dequeue 原子认领 + 有序性纪律**：固定候选集 + SKIP LOCKED + RETURNING 翻转，注释明确拒绝 LIMIT 查询（真实乱序 bug 教训）
3. **durable execution 存储哲学二分**：DBOS（数据库即 journal，零额外基础设施）vs Restate（专用 bifrost 日志段）——L5 Principle，Cross-project validation pending（KO-D08）

## 验证结论
IVR（盲重建）：10 CONFIRMED / 2 CONTRADICTED（系统表 10→11、Flow 22→34）/ 1 DOWNGRADED（EK 平均出边 2.38）/ 4 MISSING（CLI/Kafka/scheduler/fork-rewind）；Reconciliation 已合入；无 NEEDS_HUMAN_REVIEW

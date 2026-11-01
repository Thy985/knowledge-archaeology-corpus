# Job Manifest — ARCH-2026-10-28-001

## 候选排序（2026-10-28，Runtime 周 Day3）

| 排名 | 项目 | 模式 | 优先级 | 排序依据 | 状态 |
|---|---|---|---|---|---|
| 1 | **dbos-inc/dbos-transact-py** | initial | P0 | Knowledge Value 极高（PostgreSQL 事务即日志 = durable execution 存储哲学第三极，与 Restate bifrost 专用日志段最强对照）；Agent-AI 相关性高（Runtime 周主线，Python 栈与 dapr-agents 对照）；Novelty 高（未考古）；增长信号（1652★/112 forks，pushed 2026-10-27，v3.3.0 10-15）；近期发布（品类卡 10-12 三强对比文确认）；与已有 Corpus 连接强（restate 10-27 专用日志段 vs Postgres 事务、dapr-agents 10-25）；Benchmark 价值（Postgres 事务+幂等键+恢复语义） | **NEW → 选中** |
| 2 | dbos-inc/dbosify-py | initial | P1 | Drop-in Temporal Replacement（209★），API 兼容层知识密度低于 transact-py 本体 | QUEUED |
| 3 | dapr/dapr | initial | P1 | 锚点平台（v1.19 actor placement 增量），但 Go 大仓库数万行 + dapr-agents 已覆盖 agent 面向 | QUEUED |
| 4 | dbos-inc/dbos-transact-ts | initial | P2 | 1416★ 同哲学第二实现，与 Python 版重复度高 | QUEUED |

## 选中项目
- **project**: dbos-transact-py
- **repository**: https://github.com/dbos-inc/dbos-transact-py.git
- **mode**: initial（corpus 无 dbos* 记录）
- **priority**: P0
- **GitHub 元数据**: Python / MIT / 1652★ / 112 forks / created 2024-07-19 / pushed 2026-10-27T18:11:18Z；release v3.3.0（2026-10-15）/ v3.2.0（09-29）/ v3.1.0（09-24）
- **reason**: DBOS 是"PostgreSQL 原生 durable execution"开源代表（品类卡第 8 向候选、10-12 三强对比文确认 Temporal/Restate/DBOS 格局）；核心 runtime 闭源（DBOS Cloud），开源侧 dbos-transact-py 是最大语言 SDK——workflow 状态/步骤结果/journal 全存 Postgres 表（SQL 即日志），与 Restate（bifrost 专用日志段：RocksDB/sequencer/memory 三 provider）构成 durable execution **存储哲学第三极**；为周五（10-30）durable 品类 7-way 横向提供最强"零额外基础设施"对照；Python 栈与 dapr-agents 直接可比
- **source_entry**: KnowlegeMap `03_expansion_queue/candidates/[cand]durable-execution-category-mainstreaming-2026-10.md`（§2 Day3 增量：DBOS 第 8 向候选，["https://revisit.tech/blog/durable-execution-in-2026-revisit-temporal-restate-and-dbos-before-you-hand-roll-retries-sagas-and-cron-jobs/"]）+ `05_scan_logs/2026-10/2026-10-28_radar-watch_scan.md`

## 否决理由
- Temporal：品类卡已覆盖、数十万行超 48h 预算
- Mastra：框架层 API 封装、知识密度中（10-27 已否决，维持）
- Diagrid agent-integrations：drop-in 包装库、无独立代码库考古价值
- MS/AWS/Obelisk/Vercel/Notion：平台文档/SDK 型、无独立代码库考古价值
- dbos-transact-rust：43★ 过小；dbos-transact-java：248★ 且 Java 栈本轮非主线

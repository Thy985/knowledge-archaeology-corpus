# Independent Validation Report — ARCH-2026-10-28-001

> Auditor 盲重建：独立重读仓库关键文件（不把主考古当事实源），对比后判定。禁止修改原考古产物；修正经 Reconciliation 写入 corpus。

## 判定统计
| 判定 | 数量 | 项 |
|---|---|---|
| CONFIRMED | 10 | IV-2,3,4,5,8,9,10 + 补充（单语句 outcome/owner_xid 认证/SQLite RETURNING 防护） |
| CONTRADICTED | 2 | IV-1（系统表 10→11）、IV-6（Flow Edge 22→34） |
| DOWNGRADED | 1 | IV-7（EK 平均出边 2.17→2.38 精确重算） |
| MISSING | 4 | CLI 子命令全集 / Kafka 消费实现 / scheduler cron 内部 / fork+rewind 全路径（主考古已声明 PARTIAL，确认遗漏） |
| NEEDS_HUMAN_REVIEW | 0 | 本轮无 fail-open 类争议 |

## 3 成功（重点 CONFIRMED）
1. **owner_xid 认证写**：update_workflow_outcome 只对 "PENDING row still owned by owner_xid" 生效（_sys_db.py:1494-1500）；返回值 False = 失去所有权 → park 轮询——所有权不变量属实
2. **SQLite 非生产声明**：_sys_db.py:774 "for development and testing. PostgreSQL is recommended for production use"——双后端定位属实
3. **dequeue 原子认领纪律**：固定候选集 + `with_for_update(skip_locked=True)` + `returning(workflow_uuid)` 翻转（_sys_db.py:5371-5419）；注释明确拒绝 LIMIT 查询防乱序；SQLite 无 SKIP LOCKED 时以 RETURNING 翻转做防护（5371 注释）——真实 bug 教训属实

## 3 错误（CONTRADICTED / DOWNGRADED）
1. **[CONTRADICTED] 系统表 10→11 张**：SystemSchema 实际 11 个 Table（workflow_status/workflow_input/workflow_output/operation_outputs/notifications/workflow_events/workflow_events_history/streams/workflow_schedules/application_versions/queues）——主考古 00/01/02/06 均写"10 张"，漏数 workflow_events_history
2. **[CONTRADICTED] Flow Edge 22→34**：04_flow_atlas 实际 34 Edge（Control 5/State 5/Data 5/Evidence 4/Authority 5/Memory 4/Policy 6）——06/run_metadata 写 22 错误
3. **[DOWNGRADED] EK 平均出边 2.17→2.38**：精确重算（24 EK 共 57 边 → 平均 2.38；游离 0 确认）——02 末尾统计凭估

## 遗漏（MISSING，确认主考古 Coverage=PARTIAL 声明）
- CLI 子命令全集（dbos/cli/cli.py：init/start/migrate 等）
- Kafka 消费实现（_kafka.py：KafkaConsumer 生命周期/offset 管理）
- scheduler cron 内部（_scheduler.py：croniter 解析/automatic_backfill 语义）
- fork/rewind 全路径（_workflow_commands.py：fork_workflow/rewind_workflow 边界）

## 附加发现（Observation，非主考古错误）
- **IV-2' 补充**：update_workflow_outcome 单语句 autocommit 优化——"One statement, run in autocommit: a single round trip"（_sys_db.py:1511-1518）；UPDATE 返回行驱动 upsert，状态写与输出写原子且无孤儿 payload——owner 认证与原子写一体
- **IV-5' 补充**：SQLite dequeue 走 unlocked re-read + RETURNING flip 防护（无 SKIP LOCKED）——双后端语义差异显式管理

## Benchmark case（供 skill CI）
**Case**: "库级 durable execution（数据库即 journal）在恢复时如何保证不重复执行已完成步骤？"
- 期望：查表跳过（checkpoint 结果缓存）而非重跑
- DBOS 证据：check_operation_execution（_sys_db.py:3648）+ record_operation_result（:3410）+ operation_outputs 表（workflow_uuid+function_id PK）
- 反例面：错误不可序列化→重跑（D07）；owner 丢失→park 轮询（D04）；SQLite 生产不可用（C-D07）
- 对照：Restate is_resumable（journal_table/mod.rs:41-46）——两种实现同答"跳过"，机制不同（DB 查表 vs header 完成标记）

# DBOS Transact 考古总览 — ARCH-2026-10-28-001

## 一句话定位
**"数据库即日志"的轻量 durable execution**——不搭编排服务、不用专用日志段，把 workflow 状态/步骤结果/journal 全存 Postgres（或 SQLite）表，用数据库事务原子性做 checkpoint，故障后从最后完成步骤重放。

## 六条认知主线
1. **数据库即协调器（无额外基础设施）**：系统表 11 张（workflow_status/operation_outputs/notifications/workflow_events/workflow_events_history/streams/schedules/queues/application_versions/input/output）承载全部编排状态；SQLAlchemy + psycopg3 双后端（Postgres 生产/SQLite 开发测试）；README 明言 "there's no additional infrastructure for you to configure or manage"
2. **owner_xid 事务级执行权**：workflow_status 每行带 owner_xid（当前执行者 token）+ creator_xid（不可变创建者）；所有状态写都带 `WHERE owner_xid = token` 条件——数据库事务即互斥锁，失去所有权（update 返回 False）→ 执行者 "park" 并轮询 owner 结果（`_get_wf_invoke_func` persist 的 adopt_recorded_outcome）
3. **步骤级 checkpoint = operation_outputs 表**：每个 step/子 workflow 完成写 operation_outputs（workflow_uuid+function_id PK + output/error/child_workflow_id + serialization + retention_timestamp）；重放时 check_operation_execution 查表——已完成步骤直接取结果/重抛错误，不重跑；步骤失败也 checkpoint（replay 重抛）；**错误不可序列化则不记录**（"leave it unrecorded so replay re-runs"）
4. **dequeue 原子认领 + 有序性纪律**：固定候选集 + `FOR UPDATE SKIP LOCKED` + UPDATE...RETURNING 原子翻转 ENQUEUED→PENDING；注释明确拒绝 LIMIT 查询（"SKIP LOCKED could slide past a locked head and admit out of order"）——从真实 bug 学到的有序性教训；claim 时 recovery_attempts+1 计入 DLQ 限额
5. **7 态工作流状态机 + 死信/恢复**：PENDING/SUCCESS/ERROR/MAX_RECOVERY_ATTEMPTS_EXCEEDED/CANCELLED/ENQUEUED/DELAYED；recovery_attempts 超限 → dead_letter_workflows（MAX_RECOVERY_ATTEMPTS_EXCEEDED + 清 owner/dedup/queue）；resume 给新预算（recovery_attempts=0）；reenqueue_for_recovery 用 executor_ids 谓词幂等
6. **并发原语也 durable**：DBOS.asyncio.wait 持久化 done/pending 索引（恢复时确定性重建）；set_event/get_event 用 notifications 表 + LISTEN/NOTIFY（数据库触发器保证 "recv is never woken before its row commits"）；streams 流式 offset

## 关键数字
- HEAD `47379ca`（main，2026-10-27）；最近 release v3.3.0（10-15）
- 规模：dbos/ 52 py 30551 行；tests/ 54 文件 47486 行（测试/代码 1.55x）；chaos-tests 2 文件
- 系统表 11 张；状态机 7 态；核心依赖 6 个（pyyaml/dateutil/croniter/psycopg3/sqlalchemy/click）
- Python ≥3.10；MIT；双后端（Postgres/SQLite）+ 企业版 sharding（_enterprise.load 动态注入，核心闭源）

## Corpus 连接
- **restate（10-27）**：专用 bifrost 日志段（RocksDB/sequencer/memory 三 provider）vs DBOS 直接用数据库表当 journal——durable execution 存储哲学两极（"你已有数据库吗？" vs "需要专用日志基础设施"）
- **dapr-agents（10-25）**：Python 步骤持久化（actor/workflow building block）vs DBOS 轻量库级
- **substrate（10-26）**：整机快照 vs 数据库记录恢复
- **品类卡**：Temporal/Restate/DBOS 三强格局确认（10-12 对比文）——DBOS = PostgreSQL 原生极简路线

## 审计声明
- DBOS 核心 runtime 闭源（DBOS Cloud）；本考古只覆盖开源 SDK（dbos-transact-py）
- 企业版（sharding）走 `_enterprise.load` 动态注入闭源包——不可考古面，已标注
- 以代码为准（file:line 锚）；README 定位为意图证据非实现事实

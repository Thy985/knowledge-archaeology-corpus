# Project Map — dbos-transact-py（ARCH-2026-10-28-001）

## 定位与历史
- DBOS Transact：轻量 durable workflows（"Lightweight Durable Workflows"），Postgres 之上（README）
- 仓库 dbos-inc/dbos-transact-py：Python SDK（MIT），created 2024-07-19，pushed 2026-10-27，1652★/112 forks
- 版本 v3.3.0（10-15）；pdm SCM 动态版本（main=preview next minor aN；release 分支=tag；feature 分支=aN+git hash）
- 组织生态：dbos-transact-{py,ts,go,java,rust} + dbosify-py（drop-in Temporal replacement）+ durable-swarm 等

## License
- MIT（dbos-transact-py）；DBOS 核心 runtime 闭源（DBOS Cloud），企业版功能走 `_enterprise.load(description).ShardedSystemDatabase` 动态注入闭源 dbos-enterprise 包

## 架构总览
```
应用代码（@DBOS.workflow / @DBOS.step / @DBOS.queue / @DBOS.schedule / @DBOS.kafka_*）
        │
        ▼
   DBOS 类（_dbos.py：装饰器注册/配置/launch/生命周期）
        │
        ▼
  _core.py（workflow/step 执行：start_workflow / execute_dequeued_workflow / enqueue）
        │                              │
        ▼                              ▼
  DBOSContext（_context.py：workflow/step 边界、owner_xid、roles、Otel）   queue_thread（_queue.py：轮询 dequeue）
        │                              │
        ▼                              ▼
  SystemDatabase（_sys_db.py ABC + Postgres/SQLite 实现）
        │
        ├── workflow_status（7 态 + owner_xid/creator_xid/recovery_attempts）
        ├── operation_outputs（步骤 checkpoint：output/error/child_workflow_id）
        ├── notifications / workflow_events / workflow_events_history / streams（信号/事件/流）
        ├── workflow_schedules / queues / application_versions（调度/队列/版本）
        └── workflow_input / workflow_output（参数/结果 + retention）
```

## 核心模块（行数/职责）
| 模块 | 行数 | 职责 |
|---|---|---|
| `_sys_db.py` | 6966 | SystemDatabase ABC + 全部状态读写（init/claim/outcome/dead-letter/recovery/schedules/events/notifications/streams） |
| `_dbos.py` | 4410 | DBOS 类：workflow/step/queue/schedule/kafka 装饰器 + launch + 配置 |
| `_core.py` | 3508 | 执行内核：start_workflow/enqueue/execute_dequeued_workflow/重放/persist |
| `_context.py` | 1221 | DBOSContext：上下文边界/owner_xid/roles/Otel/SetWorkflowID 等 |
| `_queue.py` | 1006 | Queue 类 + queue_worker_thread/queue_thread（轮询调度） |
| `_workflow_commands.py` | — | workflow 内命令：get/fork/rewind/delete/datasource checkpoint 清理 |
| `_datasource*.py` | — | 用户数据源双后端 + 步骤级事务 checkpoint |
| `_sys_db_postgres/sqlite.py` | — | 系统数据库双后端实现 |
| `_schemas/system_database.py` | 274 | 11 张系统表 schema |
| `_recovery.py` | 76 | 恢复辅助（薄层） |
| `_serialization/_shard_keys/_roles/_tracer/_validation` | — | 序列化/分片/角色/Otel/Pydantic 校验 |

## 核心数据结构
- `WorkflowStatusInternal`（TypedDict）：workflow_uuid/status/name/inputs/output/error/executor_id/recovery_attempts/queue_name/deduplication_id/priority/queue_partition_key/owner_xid/creator_xid/delay_until_epoch_ms/rate_limited/shard_key 等
- `operation_outputs` 行：workflow_uuid+function_id PK + output/error/child_workflow_id + serialization + retention_timestamp
- `Queue`：name/concurrency/worker_concurrency/rate_limit_max/rate_limit_period_sec + partition_* 分片限流 + polling_interval_sec
- `RecordedResult`/`OperationResultInternal`（TypedDict）：步骤记录载体

## 生命周期
1. 注册：@DBOS.workflow/@DBOS.step 装饰器 → DBOSRegistry（get_func_info）
2. 启动（start_workflow）：_init_workflow 单事务持久化 status+inputs（+父步骤记录）→ should_execute 判定（去重/复用）
3. 执行（_execute_workflow_wthread/_async）：roles 检查 → _dropping_datasource_checkpoints 包裹 → persist（执行+序列化+update_workflow_outcome）
4. 步骤（step 装饰器 → EnterDBOSStepRetry）：check_operation_execution → 未记录则执行 → record_operation_result
5. 完成/失败：update_workflow_outcome（SUCCESS/ERROR，带 owner_xid 条件）→ 失败 park 轮询
6. 队列（enqueue）：dequeue 原子认领 → execute_dequeued_workflow
7. 恢复：重启 → recovery_workflows 重入队（executor_ids 幂等）；超限 → dead_letter

## 状态机（7 态）
PENDING / SUCCESS / ERROR / MAX_RECOVERY_ATTEMPTS_EXCEEDED / CANCELLED / ENQUEUED / DELAYED
- 活跃：ENQUEUED/PENDING/DELAYED（workflow_is_active）
- 终态写 owner_xid=None + completed_at

## 测试体系
- pytest 54 文件 47486 行（> 主代码）；SQLite+Postgres 双跑（conftest 参数化）
- 关键测试文件：test_singleexec/test_singleton（单次执行）、test_duplication_policy（去重）、test_rewind（回滚）、test_datasource（数据源 checkpoint）、test_schema_migration、test_queue、test_async_workflow_management、test_concurrency_primitives
- chaos-tests/test_workflows.py：进程级故障注入
- CI：unit-test.yml / chaos-test.yml / publish.yml

## 配置与治理
- DBOSConfig（_dbos_config.py）：system_database_url/engine_kwargs/schema/app_name/telemetry
- 环境变量 DBOS__ 前缀 + config.yaml
- CONTRIBUTING/DEVELOPING；pre-commit；pyrightconfig.test.json
- 企业面：_enterprise.load 动态闭源注入（sharding）

## 外部依赖（核心 6）
pyyaml / python-dateutil / croniter / psycopg[binary]≥3.1 / sqlalchemy[asyncio]≥2.0.43 / click；可选 otel/pydantic/aiosqlite

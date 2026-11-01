# Snapshot Artifact — ARCH-2026-10-28-001（dbos-transact-py）

## 快照
- repository: https://github.com/dbos-inc/dbos-transact-py.git
- HEAD: `47379ca717ec93ef7cd853ad7b43169c4e5cf530`（main，2026-10-27 09:47:58 -0700，"Serialize concurrent SQLite migrations (#923)"）
- 版本: 动态（pdm SCM 版本钩子）；main 分支=preview（next minor aN）；最近 release tag v3.3.0（2026-10-15）/ v3.2.0（09-29）/ v3.1.0（09-24）
- License: **MIT**（dbos-transact-py 本身；注意：DBOS 核心 runtime 闭源，仅 SDK 开源）
- 语言: Python ≥3.10（classifier 至 3.14）

## 规模与结构
- dbos/ 主包：52 个 .py，**30551 行**
- tests/：54 个测试文件，**47486 行**（测试 > 主代码 1.55x）；chaos-tests/：conftest + test_workflows
- 单包布局（无 src/ 层级），模块全为 `_*.py` 私有前缀

## 入口
- CLI：`dbos = dbos.cli.cli:app`（click）
- 库入口：`dbos/__init__.py` 导出 DBOS/DBOSClient/DBOSConfig/Queue/Debouncer/StepOptions/WorkflowHandle 等
- 启动：`_dbos.py`（4410 行）DBOS 类（workflow/step/queue/schedule/kafka 装饰器 + launch）

## 核心模块（行数/职责）
| 模块 | 行数 | 职责 |
|---|---|---|
| `_sys_db.py` | 6966 | 系统数据库 ABC + 全部状态读写（claim/outcome/dead-letter/recovery） |
| `_dbos.py` | 4410 | DBOS 类：装饰器注册/配置/生命周期/launch |
| `_core.py` | 3508 | workflow/step 执行与重放（start_workflow/enqueue/execute_dequeued） |
| `_context.py` | 1221 | DBOSContext 上下文（workflow/step 边界） |
| `_queue.py` | 1006 | durable queue（dequeue/分区/限流） |
| `_workflow_commands.py` | — | workflow 内部命令（step 记录/事件/数据源 checkpoint） |
| `_recovery.py` | 76 | 恢复辅助（薄层，主逻辑在 _sys_db/_core） |
| `_datasource*.py` | — | 数据源双后端（Postgres/SQLite）+ checkpoint |
| `_sys_db_postgres.py` / `_sys_db_sqlite.py` | — | 系统数据库双后端实现 |
| `_schemas/system_database.py` | 274 | 系统表 schema（10 表） |
| `_serialization.py` / `_shard_keys.py` / `_roles.py` / `_tracer.py` | — | 序列化/分片/角色/OTel |

## 系统表（"数据库即日志"实体证据）
- `workflow_status`（workflow_uuid PK/status/recovery_attempts/**owner_xid**/**creator_xid**/deduplication_id/priority/queue_partition_key/delay_until_epoch_ms/rate_limited/application_name）
- `workflow_input` / `workflow_output`（+retention_timestamp 保留期）
- `operation_outputs`（**步骤结果缓存**：workflow_uuid+function_id PK/output/error/child_workflow_id/retention_timestamp）
- `notifications`（topic/message/consumed 标志/message_uuid gen_random_uuid）
- `workflow_events` + `workflow_events_history`（可变+不可变历史双表）
- `streams`（key+offset+function_id）
- `workflow_schedules`（schedule/cron_timezone/automatic_backfill）
- `application_versions` / `queues`（concurrency/worker_concurrency/rate_limit/partition_*）

## 状态机（WorkflowStatusString，7 态）
PENDING / SUCCESS / ERROR / MAX_RECOVERY_ATTEMPTS_EXCEEDED / CANCELLED / ENQUEUED / DELAYED
- 活跃判定：ENQUEUED/PENDING/DELAYED（workflow_is_active）

## 测试体系
- pytest + SQLite/Postgres 双跑；test_* 覆盖：async/serialization/queue/recovery/rewind/singleton/singleexec/datasource/schema_migration/duplication_policy/kafka/telemetry/otel
- chaos-tests：workflow 混沌测试（进程级故障注入）
- CI：unit-test.yml / chaos-test.yml / publish.yml；pre-commit；pyrightconfig.test.json

## 配置
- `_dbos_config.py`：DBOSConfig（system_database_url/engine_kwargs/schema/app_name/telemetry）
- 环境变量 + config.yaml；DBOS__ 前缀

## 治理
- CONTRIBUTING.md / DEVELOPING.md；AGENTS 无独立文件（对比 Restate 的 AGENTS.md）
- 企业版：`_enterprise.py` 动态 load（dbos-enterprise 包，sharding 等）——核心闭源面

## 外部依赖（核心 6）
pyyaml / python-dateutil / croniter（cron 调度）/ psycopg[binary]≥3.1（Postgres 驱动）/ sqlalchemy[asyncio]≥2.0.43 / click；可选：otel（opentelemetry）、validation（pydantic）、aiosqlite

## 与品类对照（Corpus 连接）
- **Restate（10-27）**：专用 bifrost 日志段（RocksDB/sequencer/memory）vs DBOS 直接用 Postgres 表当 journal——存储哲学对照
- **dapr-agents（10-25）**：Python 步骤持久化对照（actor/workflow 模型）
- **substrate（10-26）**：整机快照 vs 数据库记录恢复

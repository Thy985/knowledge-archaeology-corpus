# Candidates — dbos-transact-py（ARCH-2026-10-28-001）

> 不能确认的内容留在候选；Hypothesis 不冒充 Fact；Cross-project Candidate 不写成 Principle。

## 跨项目假说（Cross-project Candidates）
### C-D01 幂等三层组合（假说）
- **内容**: durable execution 的去重可分层组合——**workflow_id 层（init 去重）+ queue deduplication_id 层（队列去重）+ operation 层（child/step 结果缓存）**三层互补
- **当前证据**: DBOS 内 D14（duplication_policy）+ deduplication_id 字段 + D05（check_operation_execution）三层并存 [code] _core.py:705-770, _sys_db.py:3410, 3648
- **缺失证据**: Restate 的幂等（compute_idempotency_id/is_idempotent）是否同构三层——未对照验证
- **验证路径**: 周五 7-way 横向（Dapr/Restate/Temporal/DBOS 幂等模型对比表）

### C-D02 虚拟对象/单例执行范式（假说）
- **内容**: "数据库行 + owner 认证"可替代专门的 virtual object/keyed actor 机制——DBOS 的 singleton/singleexec（test_singleton/test_singleexec）与 Restate vqueues 解决同一"按 key 串行化"问题
- **当前证据**: DBOS singleexec/singleton 测试存在 [code] tests/test_singleton.py, tests/test_singleexec.py；DBOS 无独立 vqueue 抽象（对比 restate crates/vqueues）
- **缺失证据**: singleexec 的串行化实现路径未精读（表级锁 vs 行级锁）
- **验证路径**: 精读 _dbos.py singleexec 实现 + 对照 Restate vqueues

### C-D03 journal 存储介质决定运维面（假说）
- **内容**: durable execution 的**存储介质选择决定运维面复杂度**——数据库路线（DBOS）借用已有 DB 运维；专用日志路线（Restate/Temporal）需自运维日志基础设施
- **当前证据**: DBOS README "no additional infrastructure" + D01（10 表）[code] README.md, _schemas/system_database.py:23-274；restate bifrost 三 provider
- **缺失证据**: 两类系统的生产运维成本对比数据（未实测）
- **验证路径**: 10-30 品类横向 + DBOS/Restate 生产部署文档对比

## 暂定结论（Tentative）
### C-D04 journal 版本迁移语义（暂定）
- DBOS 双后端（Postgres/SQLite）+ schema 迁移（test_schema_migration）+ 最新 commit "Serialize concurrent SQLite migrations (#923)"——并发迁移串行化是本项目近期重点（暂定：迁移正确性是双后端扩展的主要成本）
- 证据: HEAD commit 47379ca；_datasource_migration.py；test_schema_migration.py
- 状态: Observation（未深读迁移算法）

### C-D05 企业 sharding 边界（暂定）
- _enterprise.load 动态注入闭源包提供 ShardedSystemDatabase；shard_key 字段贯穿（_shard_keys.py）——开放 SDK 的架构为闭源企业版预留扩展点
- 证据: _sys_db.py:661-685（sharding 工厂）, _shard_keys.py
- 状态: Observation（闭源面不可考古，仅记录接口形态）

### C-D06 asyncio 并发与 DB 阻塞的张力（暂定）
- _warn_sync_db_call_in_async_context + _background_event_loop.submit_coroutine + asyncio.to_thread 混用——async workflow 的 DB 调用跨线程模型
- 证据: _queue.py:36, _core.py:921-926
- 状态: Hypothesis（未深读 async 执行路径的线程模型）

## 已排除
### C-D07 SQLite 生产可用性（排除）
- SQLiteSystemDatabase 明确标注 "for development and testing. PostgreSQL is recommended for production use"（_sys_db.py:764-767）——SQLite 后端非生产路线
- 状态: 已排除（README/代码双重声明）

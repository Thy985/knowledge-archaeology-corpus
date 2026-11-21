# Engineering Knowledge Graph — dbos-transact-py（ARCH-2026-10-28-001）

> 24 条 EK；每条声明 links（六类边）；证据锚 [code] file:line 可回溯。

## EK-D01 数据库即系统状态（无额外编排基础设施）
所有编排状态（workflow 状态/步骤结果/通知/事件/流/调度/队列）存 Postgres（或 SQLite）系统表 11 张；不引入专用日志段/编排服务
- links: subsystem→D05,D06,D07,D08,D09；constraint→D02
- [code] dbos/_schemas/system_database.py:23-274（SystemSchema 10 表）

## EK-D02 owner_xid 事务级执行权互斥
workflow_status 每行 owner_xid（当前执行者 token）；所有状态写 WHERE owner_xid=token；数据库事务即互斥锁
- links: mechanism→D03；causal→D04；subsystem→D01
- [code] dbos/_sys_db.py:1490-1590（update_workflow_outcome _owned 变体）、:3294-3308（_check_owned_workflow）

## EK-D03 creator_xid 不可变创建者标识
每次创建生成 uuid；直接 start 时 creator 也拥有执行权（ctx.owner_xid=creator_xid）；重试插入识别已提交行
- links: mechanism→D02；constraint→D02
- [code] dbos/_core.py:673-676（creator_xid 生成）、:762（ctx.owner_xid = creator_xid）

## EK-D04 失去所有权 → park + 轮询（adopt_recorded_outcome）
persist 的 update_workflow_outcome 返回 False（行被 CANCELLED/dead-letter/转手/消失）→ 执行者放弃并轮询 owner 已记录结果；DBOSWorkflowConflictIDError/DBOSWorkflowCancelledError 同路径
- links: causal→D02；contrast→D11（与死信不同：park 是所有权让渡非终止）
- [code] dbos/_core.py:900-935（adopt_recorded_outcome）、:929-944（not_recorded_warning）

## EK-D05 步骤 checkpoint = operation_outputs 表
每 step/子 workflow 完成写 operation_outputs（workflow_uuid+function_id PK）；重放 check_operation_execution 取结果/重抛错误
- links: mechanism→D06；subsystem→D01；causal→D07
- [code] dbos/_sys_db.py:3410-3455（record_operation_result）、:3648-3660（check_operation_execution）

## EK-D06 步骤失败也 checkpoint（replay 重抛而非重跑）
_record_operation_error：步骤异常序列化后写 operation_outputs.error；replay 时 _raise_recorded_error 重抛
- links: mechanism→D05；constraint→D07
- [code] dbos/_sys_db.py:3457-3495（_record_operation_error）

## EK-D07 错误不可序列化则不记录（fail-open 重跑）
checkpoint 的错误无法反序列化会 wedge workflow → 记录放弃，replay 重跑该步骤
- links: constraint→D06；contrast→restate NEEDS_HUMAN_REVIEW（is_completed unwrap_or(true)）
- [code] dbos/_sys_db.py:3462-3472（Not checkpointing...leave it unrecorded）

## EK-D08 dequeue 固定候选集 + SKIP LOCKED + RETURNING 原子翻转
候选集锁定（非 LIMIT 查询——SKIP LOCKED 会滑过锁头导致乱序）；UPDATE...RETURNING 翻转 ENQUEUED→PENDING
- links: mechanism→D09；constraint→D10；causal→D12
- [code] dbos/_sys_db.py:5371-5436（dequeue 事务注释）

## EK-D09 claim 时 recovery_attempts+1（DLQ 限额在 dispatch 计）
dequeue 的 UPDATE 同时 recovery_attempts+1；"Count this dispatch against the DLQ limit; no later insert does it"
- links: mechanism→D08；causal→D13
- [code] dbos/_sys_db.py:5405-5412

## EK-D10 claim_guard 重校验（防 hijack）
dequeue 二次校验 status=ENQUEUED/queue_name/queue_partition_key/version/application_name——resume 移走行不劫持
- links: constraint→D08；subsystem→D01
- [code] dbos/_sys_db.py:5387-5394

## EK-D11 死信（MAX_RECOVERY_ATTEMPTS_EXCEEDED）
dead_letter_workflows：PENDING + recovery_attempts≥min → 死信 + 清 owner/dedup/queue/started_at；resume 给新预算（recovery_attempts=0）
- links: causal→D13；contrast→D04
- [code] dbos/_sys_db.py:1444-1473（dead_letter）、:1728-1747（resume）

## EK-D12 恢复幂等（reenqueue_for_recovery executor_ids 谓词）
恢复声明一组 executor 死亡；live executor 已 dequeue 后行上 executor_id 变化 → 重复恢复请求匹配不到 → 不重复入队运行中 workflow
- links: causal→D13；mechanism→D08
- [code] dbos/_sys_db.py:5443-5475

## EK-D13 7 态状态机 + workflow_is_active 判定
PENDING/SUCCESS/ERROR/MAX_RECOVERY_ATTEMPTS_EXCEEDED/CANCELLED/ENQUEUED/DELAYED；活跃=ENQUEUED/PENDING/DELAYED
- links: subsystem→D01；constraint→D11,D12
- [code] dbos/_sys_db.py:132-151

## EK-D14 幂等/去重（duplication_policy + workflow_id_reuse_policy）
_init_workflow：DBOSQueueDeduplicatedError/DBOSWorkflowIDInUseError → return-existing 查已有 workflow attach 或重抛；child 幂等经 check_operation_execution 先行
- links: mechanism→D05；constraint→D15
- [code] dbos/_core.py:705-770（_init_workflow 去重分支）、:1478-1496（child 幂等）

## EK-D15 单事务持久化（status+inputs+父步骤记录）
_init_workflow 一次事务写 workflow_status + workflow_input（+child 时父 operation_outputs）；"Assembled once, outside the retry loop"
- links: constraint→D14；mechanism→D01
- [code] dbos/_core.py:651-662、:675-705

## EK-D16 durable asyncio.wait（并发原语 checkpoint）
DBOS.asyncio.wait 持久化 done/pending task 索引；恢复时确定性重建（recorded_indices）
- links: mechanism→D05；contrast→D17（与事件不同源）
- [code] dbos/_dbos.py:2100-2180

## EK-D17 LISTEN/NOTIFY + 数据库触发器通知
notifications 表由 DB 触发器推送（"recv is never woken before its row commits"）；coalesce 限流合并写路径外通知
- links: mechanism→D18；subsystem→D01
- [code] dbos/_sys_db.py:431-432、:637、:3883-3971

## EK-D18 事件双表（workflow_events + 不可变 history）
可变事件表 + workflow_events_history（workflow_uuid+key+function_id）向后兼容不可变审计
- links: subsystem→D01；constraint→D17
- [code] dbos/_schemas/system_database.py:146-180

## EK-D19 streams 流式 offset（durable stream）
streams 表 key+offset+function_id PK——流消费位置持久化
- links: subsystem→D01；mechanism→D05
- [code] dbos/_schemas/system_database.py:182-200

## EK-D20 数据源事务 checkpoint（共享 engine）
用户 SQLAlchemy 会话 bind 系统 engine → 数据源事务与 workflow checkpoint 同事务边界；workflow 完成/异常后删除已完成 checkpoint（_dropping_datasource_checkpoints）
- links: mechanism→D01；causal→D21；contrast→D05（步骤 vs 数据源两层 checkpoint）
- [code] dbos/_datasource.py:113-128（_checkpoint_id/step checkpoint）、:312-330（_delete_checkpoints[_if_owner]）；dbos/_core.py:855-889

## EK-D21 数据源 checkpoint 删除的 owner 条件
_delete_checkpoints_if_owner：仅 owner_xid 仍拥有时提交删除（防误删他执行者的 checkpoint）
- links: constraint→D20；mechanism→D02
- [code] dbos/_datasource.py:320-330

## EK-D22 队列分片限流（partition_* 体系）
Queue：concurrency/worker_concurrency/rate_limit + partition_concurrency/partition_worker_concurrency/partition_rate_limit（每分区）；legacy priority_enabled/partition_queue 只写不读
- links: subsystem→D08；constraint→D23
- [code] dbos/_schemas/system_database.py:243-274（queues 表）、dbos/_queue.py:356-505

## EK-D23 限流状态持久化（rate_limited 标志）
workflow_status.rate_limited 字段 + DequeueStopReason（worker_concurrency/partition_worker_concurrency/limiter/partition_limiter/concurrency/partition_concurrency 6 原因）
- links: constraint→D22；subsystem→D13
- [code] dbos/_sys_db.py:156-171（DequeueStopReason）

## EK-D24 序列化 per-row（serialization 字段向后兼容）
系统表各行带 serialization 字段（DEFAULT/PORTABLE JSON 等）；反序列化失败 → 立即 ERROR 而非无限重试
- links: mechanism→D05；constraint→D07
- [code] dbos/_sys_db.py:409-441（StepInfo/RecordedResult serialization）、dbos/_core.py:1285-1300（deser_error → ERROR）

## 聚合指标
- EK 总数 24；平均出边 2.38；游离 0；六类边覆盖：mechanism×9 subsystem×8 causal×7 dependency×0 constraint×9 contrast×4

# Flow Atlas — dbos-transact-py（ARCH-2026-10-28-001）

> 七类流；Edge 可回溯 symbol/file/condition/state transition。

## Control Flow（控制流）
| Edge | 描述 | 回溯 |
|---|---|---|
| CF-1 | start_workflow → _init_workflow（单事务持久化）→ should_execute 判定 → _execute_workflow_wthread | _core.py:1451,651-770,1167 |
| CF-2 | enqueue → queue_thread 轮询 → dequeue 原子认领 → execute_dequeued_workflow | _queue.py:891, _sys_db.py:5371, _core.py:1255 |
| CF-3 | step 调用 → check_operation_execution → 未记录执行 / 已记录取结果 | _dbos.py:2127, _sys_db.py:3648 |
| CF-4 | workflow 完成 → update_workflow_outcome（SUCCESS）→ False 时 park 轮询 | _core.py:929-944 |
| CF-5 | 恢复：重启 → recovery_workflows → reenqueue_for_recovery（executor_ids 幂等） | _sys_db.py:5443 |

## State Flow（状态流）
| Edge | 描述 | 回溯 |
|---|---|---|
| SF-1 | ENQUEUED → PENDING（dequeue RETURNING 翻转）+ owner_xid 写入 | _sys_db.py:5398-5425 |
| SF-2 | PENDING → SUCCESS/ERROR（update_workflow_outcome owner 认证）+ owner_xid=NULL | _sys_db.py:1490-1590 |
| SF-3 | PENDING → MAX_RECOVERY_ATTEMPTS_EXCEEDED（dead-letter 清理） | _sys_db.py:1444-1473 |
| SF-4 | PENDING → CANCELLED（cancel 路径，owner 认证） | _sys_db.py:1622-1634 |
| SF-5 | PENDING → ENQUEUED（resume/reenqueue + recovery_attempts=0） | _sys_db.py:1728-1747 |

## Data Flow（数据流）
| Edge | 描述 | 回溯 |
|---|---|---|
| DF-1 | 输入：start → workflow_input 表（retention_timestamp 保留） | _schemas/system_database.py:102-117 |
| DF-2 | 步骤结果：step 完成 → operation_outputs（workflow_uuid+function_id PK） | _sys_db.py:3410-3455 |
| DF-3 | 输出：workflow 完成 → workflow_output 表 | _schemas/system_database.py:119-130 |
| DF-4 | 序列化：per-row serialization 字段（DEFAULT/PORTABLE）→ 反序列化失败即 ERROR | _sys_db.py:409-441, _core.py:1285-1300 |
| DF-5 | 数据源：用户事务与 checkpoint 共享 engine → 完成后删除 checkpoint | _datasource.py:113-128, _core.py:855-889 |

## Evidence Flow（证据流）
| Edge | 描述 | 回溯 |
|---|---|---|
| EF-1 | 审计：任何 workflow 的完整生命周期=查 workflow_status+operation_outputs+events 表 | _schemas/system_database.py:23-274 |
| EF-2 | 不可变事件：workflow_events_history（uuid+key+function_id）保留历史值 | _schemas/system_database.py:146-180 |
| EF-3 | 失败证据：operation_outputs.error 序列化异常 + serialization 类型（replay 重抛） | _sys_db.py:3457-3495 |
| EF-4 | 步骤调试触发：DebugTriggers.DEBUG_TRIGGER_STEP_COMMIT（测试钩子） | _sys_db.py:3446 |

## Authority Flow（权威流）
| Edge | 描述 | 回溯 |
|---|---|---|
| AF-1 | 执行权：creator_xid（创建者）→ owner_xid（当前执行者）转移/竞争 | _core.py:673-676,762 |
| AF-2 | 写认证：所有状态写 WHERE owner_xid=token（update_workflow_outcome/outcome owned 变体） | _sys_db.py:1490-1590 |
| AF-3 | 失去权威 → park：adopt_recorded_outcome 轮询 owner 结果 | _core.py:900-935 |
| AF-4 | 恢复权威：resume 重置 recovery_attempts + 新 owner | _sys_db.py:1728-1747 |
| AF-5 | 角色权威：required_roles/default_required_roles → DBOSAssumeRole + _check_required_roles_or_finalize_error | _dbos.py:1435-1460, _core.py:1063,1175 |

## Memory Flow（记忆流）
| Edge | 描述 | 回溯 |
|---|---|---|
| MF-1 | 系统状态记忆：11 张系统表（workflow 状态/步骤结果/事件/流/调度/队列） | _schemas/system_database.py:23-274 |
| MF-2 | 跨 workflow 信号：workflow_events（可变）+ notifications（DB 触发器推送） | _schemas/system_database.py:146-180, _sys_db.py:3883-3971 |
| MF-3 | 流式消费位置：streams（key+offset+function_id） | _schemas/system_database.py:182-200 |
| MF-4 | 应用版本记忆：application_versions（version_name/version_timestamp） | _schemas/system_database.py:229-242 |

## Policy Flow（策略流）
| Edge | 描述 | 回溯 |
|---|---|---|
| PF-1 | 限流策略：Queue rate_limit_max/rate_limit_period_sec + partition_rate_limit（每分区）→ DequeueStopReason | _schemas/system_database.py:243-274, _sys_db.py:156-171 |
| PF-2 | 并发策略：concurrency/worker_concurrency/partition_* 预算 → dequeue 停止原因 | _queue.py:292-505 |
| PF-3 | 重试策略：step retries_allowed/interval_seconds/max_attempts/backoff_rate/should_retry → EnterDBOSStepRetry | _dbos.py:1374-1435, _context.py:1111 |
| PF-4 | 恢复预算：recovery_attempts（dispatch 计）→ dead_letter 阈值 → resume 重置 | _sys_db.py:1444-1473,1728-1747 |
| PF-5 | 去重策略：duplication_policy（return-existing）+ workflow_id_reuse_policy | _core.py:705-770 |
| PF-6 | 所有权认证策略：owner_xid 条件写（所有状态变更的强制前置） | _sys_db.py:1490-1590 |

## Flow→KO 交叉校验
| KO | 覆盖 Edge | 校验 |
|---|---|---|
| KO-D01 | AF-1/2/3, SF-1/2, DF-5 | PASS（事务=锁+checkpoint） |
| KO-D02 | CF-1/2/4/5, SF-1/2/3, DF-1/2/3 | PASS（生命周期=数据库因果链） |
| KO-D03 | AF-1/2/3/4, PF-6, DF-5 | PASS（owner 不变量） |
| KO-D04 | CF-3, DF-2/4, EF-3 | PASS（步骤三态） |
| KO-D05 | MF-2/3, CF-3, EF-2 | PASS（durable 原语族） |
| KO-D06 | CF-2, SF-1, PF-1/2, DF-2 | PASS（队列即行） |
| KO-D07 | CF-5, SF-3/5, PF-4 | PASS（恢复预算） |
| KO-D08 | DF-2, SF-1/2（对照 restate bifrost） | PASS（存储哲学二分，跨仓库） |

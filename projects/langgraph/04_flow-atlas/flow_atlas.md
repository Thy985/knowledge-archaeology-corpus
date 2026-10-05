# 04 Flow Atlas — LangGraph 七类流

> 全部从真实代码导出；关键 Edge 可回溯 `symbol/file/condition/state transition`。

## 1. Control Flow（控制流）——invoke 到超步
```
Pregel.invoke(config)                                pregel/main.py:invoke
  → PregelLoop.__enter__                             pregel/_loop.py:162 (SyncPregelLoop:1535)
  → tick() 循环                                       pregel/_loop.py:614
      ├─ cond: step > stop              → out_of_steps（停止）
      ├─ prepare_next_tasks(checkpoint, ...)         pregel/_algo.py:349
      ├─ cond: 无 tasks                 → status="done"（停止）
      ├─ cond: control.drain_requested → status="draining"（停止）
      ├─ cond: interrupt_before 命中    → raise GraphInterrupt   pregel/_loop.py:682
      ├─ 执行 tasks（BackgroundExecutor 并发）       pregel/_executor.py:40
      └─ after_tick()                                 pregel/_loop.py:698
          └─ apply_writes → updated_channels → 下轮 tick 触发集
```
关键 Edge：`tick→after_tick→tick`（状态转换：调度→写回→再调度）；终止转换：`done / out_of_steps / draining / interrupt_before`。

## 2. State Flow（状态流）——checkpoint 生命周期
```
线程启动: saver.get_tuple(config[thread_id])   → CheckpointTuple | None   checkpoint/base/__init__.py:240
  → channels_from_checkpoint(checkpoint)       → 各通道 from_checkpoint 重建   pregel/_checkpoint.py:387
执行中: 每超步 _put_checkpoint(metadata)        → create_checkpoint(新 id)      pregel/_loop.py:1142 / _checkpoint.py:224
  → saver.put(config, checkpoint, metadata, new_versions)                      checkpoint/base/__init__.py:278
恢复:   update_state(config, values, as_node)   → 伪装节点写 → 新 checkpoint    pregel/main.py:2593
分支:   update_state 产生新分支 → fork 语义（#9170/#9165 修复）
调试:   get_state_history(config) → 全量 checkpoint 链回放                    pregel/main.py:1510
```
关键 Edge：`执行写→版本推进→checkpoint 落盘`；`外部注入→新分支→版本分叉`（近期修复密集区）。

## 3. Data Flow（数据流）——输入到输出
```
输入 → ChannelWrite(RETURN/各通道)      pregel/_write.py
  → map_command 或节点返回值 → pending writes    pregel/_io.py:63
  → apply_writes: 排序→versions_seen→consume→update→finish    pregel/_algo.py:232
  → updated_channels 与 trigger_to_nodes 求交 → 下一超步任务集
节点读取: ChannelRead → read_channels(skip_empty) → 节点函数      pregel/_read.py:25
扇出: Send → TASKS 通道 → prepare_push_task_send → 独立任务       pregel/_algo.py:938
输出: updated_channels ∩ output_keys → values 事件                pregel/_loop.py:730-737
```
关键 Edge：`节点返回值→通道 update→trigger 传播`；`Send→TASKS→新任务`；未知通道→warning 忽略（_algo.py:310）。

## 4. Evidence Flow（证据流）——调试事件
```
debug=True: _emit("checkpoints", map_debug_checkpoint, ...)     pregel/_loop.py:647-665
_emit("tasks", map_debug_tasks, tasks)                          pregel/_loop.py:689
values 事件: _emit("values", map_output_values, ...)            pregel/_loop.py:735
stream(stream_mode=...): 上层把 debug/values/updates 分发给流       pregel/main.py:stream
v3 流式管线: stream_mode 由 transformer mux 收集（_collect_stream_modes）  pregel/main.py:386-416
  _V3_INVARIANT_KWARGS = ("stream_mode","subgraphs")——v3 调用方不可覆盖     pregel/main.py:390
```
关键 Edge：`超步完成→checkpoints 事件`；`节点写→values/updates 事件`；`v3 模式→transformer mux 注册/收集→子图范围传播`。证据流与 State/Data 流同源（同一 checkpoint/writes 对象），是"可审计性"的设计——每次状态转换都有事件记录。

## 5. Authority Flow（权威流）——HITL 与治理
```
节点内 interrupt(value) → raise GraphInterrupt（值随异常）        types.py:895
  → checkpoint 持久化挂起态（must have checkpointer）             dependency→EK-12
客户端（人）决策 → Command(graph/update/resume/goto)             types.py:833
  → map_command → RESUME/各通道写 → 恢复 → 节点重执行             pregel/_io.py:63
工程侧治理: AGENTS.md（format/lint/test 门禁 + Corridor analyzePlan）
```
关键 Edge：`GraphInterrupt→持久化→Command(resume)→重执行`；`PARENT→父图`（无父图→InvalidUpdateError）。权威注入点是 interrupt（运行时）与 AGENTS.md（工程时）。

## 6. Memory Flow（记忆流）——三层记忆
```
短时（线程）: thread_id → checkpoint 链（每超步）            EK-11/EK-12
增量: DeltaChannel 计数器 (updates, supersteps) → 快照频率      pregel/_loop.py:1157-1226
长时（跨线程）: BaseStore namespace/key → Item（get/search/put）  store/base/__init__.py:708
触发记忆: versions_seen 记录节点已见版本 → 决定下次执行         EK-14
```
关键 Edge：`thread checkpoint（精确可回放）` vs `store（语义寻址跨线程）`；`delta 计数器→快照决策`。

## 7. Policy Flow（策略流）——治理闭环
```
节点配置期: add_node(retry_policy/timeout/cache_policy/error_handler)   graph/state.py:376
  → compile: node defaults 应用（error-handler 路由、策略继承）          graph/state.py:1287-1301
执行期:  节点失败 → retry_on 判定（RetryPolicy.retry_on）→ 退避重试      types.py:427 / pregel/_retry.py
         超时 → _resolve_timeout(run/idle) → NodeTimeoutError           pregel/_retry.py:36
         错误 → 错误处理器任务（ERROR 通道）→ __default_error_handler__ graph/state.py:1292
持久化:  策略元数据进 CheckpointMetadata → 跨恢复保留
```
关键 Edge：`配置（编译期）→执行策略（运行期）→失败处理→状态恢复`；`重试/超时也作用于 error-handler 节点`（compile 注释）。治理闭环：Decision（用户配策略）→ Policy（RetryPolicy/TimeoutPolicy）→ Enforcement（_retry 执行）→ Future Decision（metadata 保留）。

## 流-边交叉校验
- KO-04（版本不变量）↔ State/Data 流：`版本推进`是所有关键 Edge 的公共条件——与 Flow Atlas 一致。
- KO-03（HITL）↔ Authority 流：`GraphInterrupt→Command` 完全对应。
- KO-06（失败可恢复）↔ Policy 流：`失败→重试/错误处理器→checkpoint 恢复` 闭环一致。
- 无 Flow 与 KO 矛盾。

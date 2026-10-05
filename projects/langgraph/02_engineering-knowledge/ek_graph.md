# 02 Engineering Knowledge — LangGraph（EK Graph，48 条）

> 每条 EK 必须声明 `links`（六类边：mechanism/subsystem/causal/dependency/constraint/contrast），否则是模块说明。证据一律 `file:symbol` 可回溯。游离 EK = 0。

## 核心机制簇（执行引擎）

**EK-01 Pregel 超步循环**（L2）——tick() 每轮：step 上限检查→prepare_next_tasks→interrupt_before 检查→执行；after_tick() 收尾：apply_writes→values 输出→清 pending writes。循环终止条件：无任务（done）/超步上限（out_of_steps）/drain 请求。
- 证据：pregel/_loop.py:614 `tick()`；:698 `after_tick()`
- links：mechanism→EK-02（同一超步语义）；causal→EK-14（versions_seen 决定任务集）；dependency←EK-11（循环依赖 checkpoint）

**EK-02 apply_writes 确定性写应用**（L2）——任务按 `path[:3]` 排序保证更新顺序确定；bump_step 由任务 triggers 决定；versions_seen 先记账；读过的通道 consume 推进版本；按通道分组写并 update；末超步（updated_channels 与 trigger_to_nodes 不相交）调 finish()。
- 证据：pregel/_algo.py:232 `apply_writes`
- links：mechanism→EK-01；causal→EK-14（写→版本→触发）；dependency→EK-03（版本函数）

**EK-03 版本计数乐观并发**（L2）——channel_versions 单调递增（默认 int+1）；并发节点无需锁，靠"写入时推进版本 + 下一轮按版本差推导触发"保证正确性；`get_next_version` 由 saver 注入（可自定义版本类型）。
- 证据：pregel/_algo.py:227 `increment`；:271-292；checkpoint/base/__init__.py:693 `get_next_version`
- links：mechanism→EK-04（同是无锁并发语义）；causal→EK-14；constraint→EK-13（thread 粒度版本）

**EK-04 LastValue 每步至多一值**（L2）——单值通道；一步收到多值抛 InvalidUpdateError（INVALID_CONCURRENT_GRAPH_UPDATE，提示用 Annotated key 归约）。空通道 get() 抛 EmptyChannelError。
- 证据：channels/last_value.py:20 `LastValue.update`；:66 `get`
- links：constraint→EK-31（读通道语义）；contrast→EK-05（覆盖 vs 累积）；mechanism→EK-06（同为 update 语义原语）

**EK-05 Topic PubSub 累积语义**（L2）——消息型通道；`accumulate=False` 时每步清空（不跨步累积），True 时跨步累积；checkpoint 存列表（向后兼容 tuple）；典型用于消息列表。
- 证据：channels/topic.py:23 `Topic`
- links：contrast→EK-04；subsystem→EK-06/EK-07（channels 原语族）；dependency→EK-41（react agent 依赖消息流）

**EK-06 BinaryOperatorAggregate 归约通道**（L2）——字段标注 `Annotated[type, operator]` 时生成归约通道；多值按操作符合并（如加法、取最大）；schema 推导（_is_field_binop）。
- 证据：channels/binop.py:65；graph/state.py:1904 `_is_field_binop`
- links：mechanism→EK-04（update 语义原语）；subsystem→EK-05/EK-07

**EK-07 DeltaChannel 增量快照**（L2）——增量通道：checkpoint 只存"自上次快照以来的 delta + 种子值"，而非全量值；计数器（updates 数, supersteps 数）驱动快照频率；exit 模式额外累加 delta 写入；fork/update_state/branch 场景有专门修复史（#9141/#9142/#8548/#9165/#9170）。
- 证据：pregel/_checkpoint.py:63 `delta_channels_to_snapshot`；_loop.py:1157-1226 `_put_checkpoint`；git log 修复族
- links：causal→EK-03（版本推进）；subsystem→EK-05（channels 族）；contrast→EK-11（全量快照 vs 增量）；dependency→EK-40（conformance 测试）

**EK-08 reserved 通道/节点名校验**（L2）——编译期禁止用户通道/节点使用 RESERVED 名（TASKS/RESUME/INTERRUPT 等内部通道）；违规即 ValueError。
- 证据：pregel/_validate.py:13 `validate_graph`
- links：constraint→EK-41（agent 构建命名）；causal→EK-16（compile 时触发）；subsystem→EK-09（同为验证）

**EK-09 输入通道必须被订阅**（L2）——input_channels 必须存在于已知通道且被至少一个节点订阅，否则编译失败——防止死图/悬空输入。
- 证据：pregel/_validate.py:55-70
- links：causal→EK-08；dependency→EK-01（保证循环可达）

**EK-10 并发执行 = 提交层 + 执行层**（L2）——BackgroundExecutor 提供跨线程并发提交（submit/done 回调；Async 版 asyncio.Future；`next_tick` 支持 deferred 调度）；pregel/_runner.py 是实际任务执行层（调度 run_with_retry、backpressure、链路追踪、GraphBubbleUp 错误冒泡）。
- 证据：pregel/_executor.py:40 `BackgroundExecutor`；pregel/_runner.py（执行调度）；pregel/_retry.py:573 `run_with_retry`
- links：mechanism→EK-03（并发支撑）；subsystem→EK-01（执行阶段）；dependency→EK-26（执行层调用重试）

## 状态与生命周期簇

**EK-11 Checkpoint 结构**（L1）——v（格式版本，当前 1）/id（唯一且单调递增）/ts/channel_values/channel_versions/versions_seen/updated_channels；copy_checkpoint 额外保留 pending_sends。
- 证据：checkpoint/base/__init__.py:93 `Checkpoint`；:127 `copy_checkpoint`
- links：subsystem→EK-12/EK-13（checkpoint 契约族）；dependency→EK-02（写应用更新它）

**EK-12 CheckpointSaver 契约**（L2）——抽象接口：get/get_tuple/list/put/put_writes/delete_thread/delete_for_runs/copy_thread/prune + 全套 async 对；put_writes 存中间写（task_id/task_path 粒度）；实现必须自己处理线程表/checkpoint 表/写表三件套。参考实现：InMemorySaver（memory/__init__.py:33，get_tuple:230/put:421/put_writes:467）。
- 证据：checkpoint/base/__init__.py:177 `BaseCheckpointSaver`；checkpoint/memory/__init__.py:33 `InMemorySaver`
- links：mechanism→EK-39（同为可插拔存储接口）；dependency→EK-36（依赖 serde）；constraint→EK-37（白名单约束反序列化）

**EK-13 thread_id 是持久化主键**（L1）——config.configurable.thread_id 决定 checkpoint 存取；单次工作流用 uuid4，会话记忆复用同一 thread_id；无 thread_id 则无法保存/恢复/interrupt/time-travel。
- 证据：checkpoint/base/__init__.py:183-201 docstring
- links：constraint→EK-20（interrupt 必须 checkpointer）；subsystem→EK-12；causal→EK-05（会话记忆复用同一线程累积 Topic）

**EK-14 versions_seen 决定节点再执行**（L2）——每个节点记录"已见"的通道版本；仅当节点触发的通道版本推进超过其 versions_seen 时才再次调度——这是图循环/条件执行的核心判定。
- 证据：Checkpoint.versions_seen（:116）；apply_writes 记账（_algo.py:262-269）；prepare_next_tasks（_algo.py:349）
- links：causal→EK-01→EK-03（版本→触发→执行）；subsystem→EK-11

**EK-15 状态恢复 from_checkpoint**（L1）——每通道 from_checkpoint 从快照重建自身；LastValue 取 value、Topic 取列表；MISSING 表示从未写过。
- 证据：channels/last_value.py:45 `from_checkpoint`；channels/topic.py:47
- links：dependency→EK-11（读取快照）；subsystem→EK-11/EK-14

**EK-16 StateGraph.compile 编译流水线**（L2）——checkpointer 校验（None=继承父图/False=禁用）→ STRICT_MSGPACK 时构建 serde allowlist → validate → output/stream channels 推导（排除 managed）→ node defaults 应用（error-handler 路由、cache_policy 仅常规节点，retry/timeout 也作用于 error-handler）。
- 证据：graph/state.py:1177 `compile`；:1231-1350
- links：causal→EK-08/EK-09（触发验证）；causal→EK-37（触发白名单）；subsystem→EK-17（同是编译期）

**EK-17 通道从 schema 推导**（L1）——TypedDict 字段类型映射到通道原语：普通字段→LastValue、Annotated[_, operator]→BinaryOperatorAggregate、Annotated[_, Topic]→Topic、ManagedValue 注解→managed 通道；_get_channel 带继承逻辑。
- 证据：graph/state.py:1815 `_get_channels`；:1839 `_get_channel`；:1876 `_is_field_channel`
- links：causal→EK-01（推导结果进执行引擎）；subsystem→EK-16；contrast→EK-43（managed 由同一机制推导）

**EK-18 update_state 外部状态注入**（L2）——把外部值"伪装成节点 as_node 的写"注入；as_node 缺省时取"最后一个更新者"（若不歧义）；返回新 checkpoint config。支持 bulk（多 StateUpdate）。
- 证据：pregel/main.py:2593 `update_state`
- links：contrast→EK-21（外部注入 vs 中断恢复）；causal→EK-11（产生新 checkpoint）；dependency→EK-14（注入后按版本触发后续节点）

**EK-19 subgraph checkpointer 继承**（L1）——compile(checkpointer=None) 继承父图 saver；False 显式禁用；子图 checkpoint 走父图 saver 的独立 checkpoint_ns（隔离线程历史）。
- 证据：graph/state.py:1196-1202 docstring；pregel/_loop.py:1187 `parents` 元数据
- links：subsystem→EK-16；dependency→EK-12（复用 saver）；constraint→EK-13（子图线程隔离）

## interrupt / HITL 簇

**EK-20 interrupt() 挂起协议**（L2）——节点内首次调用抛 GraphInterrupt，值随异常发给客户端；必须启用 checkpointer（依赖持久化）；response_schema 可约束恢复值。
- 证据：types.py:895 `interrupt`；errors.py:102 `GraphInterrupt`
- links：dependency→EK-12；causal→EK-24（编译期配置决定挂起点）；subsystem→EK-21/EK-22

**EK-21 恢复=重执行协议**（L2）——客户端用 Command(resume=...) 恢复；图**从节点起始处重跑全部逻辑**（非从断点续跑）。resume 值两种形态：①interrupt id→值映射（推荐，精确恢复指定中断）②单一值恢复"下一个"中断（按调用顺序匹配）；多 interrupt 并行时按任务隔离，resume id 映射解决顺序歧义。
- 证据：types.py:842-857（Command.resume 字段）；types.py:895-915 docstring；tests/test_interruption.py（`Command(resume={interrupt_id: value})` + 并行部分恢复场景）
- links：causal→EK-20（挂起→恢复）；contrast→EK-18；dependency→EK-13

**EK-22 Command 四字段指令**（L1）——graph（None=当前图/PARENT="__parent__"）/update（状态写）/resume（中断恢复值）/goto（下一节点名或 Send 序列）；是节点向图发出的"结构化动作"。
- 证据：types.py:833 `Command`
- links：subsystem→EK-23（映射为写）；causal→EK-21；dependency→EK-34（goto Send 走 TASKS）

**EK-23 map_command 三流映射**（L2）——Command→pending writes：goto（str→`branch:to:<name>` 触发 START；Send→TASKS 通道）/ resume→RESUME 通道 / update→各通道写；PARENT 无父图抛 InvalidUpdateError。
- 证据：pregel/_io.py:63 `map_command`
- links：dependency→EK-22；causal→EK-02（写进 apply_writes）；subsystem→EK-31/EK-33（io 族）

**EK-24 interrupt_before/after 编译期配置**（L1）——compile(interrupt_before=["node"] / interrupt_after=... / "*" 通配)；运行期命中挂起点→raise GraphInterrupt；仅渲染意义的 destinations 不影响执行。
- 证据：graph/state.py:1216-1218；_loop.py:682-686 `should_interrupt`
- links：causal→EK-20；subsystem→EK-16

**EK-25 answered interrupts 不再报告**（L2）——get_state 只报未应答 interrupt；已通过 resume 应答的中断不再出现在状态查询里（修复 #9103）。
- 证据：git log e6cf9ea `fix(langgraph): stop reporting answered interrupts in get_state (#9103)`
- links：mechanism→EK-21（应答状态跟踪）；subsystem→EK-20/EK-21

## 错误与重试簇

**EK-26 RetryPolicy 重试策略**（L2）——initial_interval=0.5s / backoff_factor=2.0 / max_interval=128s / max_attempts=3（含首次）/ jitter=True / retry_on 异常白名单（默认重试特定异常）；按节点配置。
- 证据：types.py:427 `RetryPolicy`
- links：subsystem→EK-27（同为节点策略）；causal→EK-28（触发重试的异常）；dependency→EK-06（重试写回归约）

**EK-27 TimeoutPolicy 超时策略**（L2）——run_timeout（总执行时长）/ idle_timeout（空闲超时）/ refresh_on（auto|heartbeat，心跳续期）；sync 环境超时不可用（sync_timeout_unsupported）。
- 证据：pregel/_retry.py:36-58 `_ResolvedTimeout`/`_resolve_timeout`
- links：causal→EK-28（超时→NodeTimeoutError）；subsystem→EK-26

**EK-28 异常分层**（L1）——GraphBubbleUp 家族（GraphInterrupt/NodeInterrupt/ParentCommand——会冒泡跨子图）+ NodeError（包装用户异常）/NodeCancelledError/NodeTimeoutError/InvalidUpdateError/GraphRecursionError（recursion_limit 超限）。
- 证据：errors.py:50-190
- links：causal→EK-29（错误处理器路由）；dependency→EK-26（retry_on 判定用异常类）；subsystem→EK-30

**EK-29 错误处理器节点**（L2）——节点可配 error_handler；builder 有默认错误处理器时自动生成 `__default_error_handler__` 节点（重名 ValueError）；错误经 ERROR/ERROR_SOURCE_NODE 通道路由。
- 证据：graph/state.py:1292-1301；pregel/_algo.py:764-798 `_read_errors_from_pending_writes`
- links：causal→EK-28（异常→处理器）；subsystem→EK-30；constraint→EK-08（保留名）

**EK-30 pending writes 中的错误传播**（L2）——错误作为保留通道写（ERROR/ERROR_SOURCE_NODE/RETURN）随 checkpoint 持久化；error handler 任务由 pending 错误驱动调度（prepare_node_error_handler_task）。
- 证据：pregel/_algo.py:1110 `prepare_node_error_handler_task`；:770 `_read_errors_from_pending_writes`
- links：dependency→EK-29；causal→EK-02（错误写也走 apply_writes）；subsystem→EK-28

## 数据流 / 读写簇

**EK-31 ChannelRead 读取封装**（L1）——节点输入经 ChannelRead 读取（读指定通道/全通道，skip_empty 控制空通道处理）；catch=False 时空通道抛 EmptyChannelError。
- 证据：pregel/_read.py:25 `ChannelRead`；pregel/_io.py:16 `read_channel`
- links：dependency→EK-01（循环里被调用）；constraint→EK-04（LastValue 空读语义）；subsystem→EK-33

**EK-32 ChannelWrite 写入封装**（L1）——节点输出转 ChannelWriteEntry 序列（含 RETURN 保留通道）；写目标通道由编译期绑定。
- 证据：pregel/_write.py:ChannelWrite；pregel/_call.py:14（convert to Runnable with ChannelWrite）
- links：subsystem→EK-31；causal→EK-02（写进 apply_writes）；dependency→EK-35（未知通道处理）

**EK-33 read_channels skip_empty 语义**（L1）——多通道批量读；skip_empty=True 时缺省通道跳过不报错；False 时抛空通道异常。
- 证据：pregel/_io.py:37 `read_channels`
- links：subsystem→EK-31；dependency→EK-14（按需读取触发判定）

**EK-34 Send 动态扇出**（L2）——节点可返回 Send(node, input) 序列；Send 写进 TASKS 通道，下一超步为每个 Send 生成独立任务（prepare_push_task_send）；实现 map-reduce 扇出。
- 证据：pregel/_algo.py:938 `prepare_push_task_send`；_io.py:63-72（map_command goto Send）
- links：mechanism→EK-10（并发执行扇出任务）；subsystem→EK-23；causal→EK-14（新任务按版本触发）

**EK-35 未知通道写被忽略**（L2）——任务写到不在 channels 的通道 → warning（"Task ... wrote to unknown channel ... ignoring it"）而非报错——容错但隐藏拼写错误。
- 证据：pregel/_algo.py:310-313
- links：contrast→EK-08（编译期严格 vs 运行期宽容）；constraint→EK-09（输入校验的另一面）

## 序列化与存储簇

**EK-36 serde 接口**（L2）——SerializerProtocol：dumps_typed→(type, bytes)/loads_typed；SerializerCompat 包装未类型化序列化器；CipherProtocol 加密基类。
- 证据：checkpoint/serde/base.py:6-55
- links：dependency→EK-12；subsystem→EK-37/EK-38（serde 族）

**EK-37 STRICT_MSGPACK 反序列化白名单**（L2）——compile 时收集所有 schema 类型 build_serde_allowlist；checkpointer.with_allowlist() 克隆 saver 并派生白名单 msgpack（JsonPlus 或 Encrypted 内层）；不支持的序列化器仅 warning（"will not be enforced"）。
- 证据：graph/state.py:1233-1254；checkpoint/base/__init__.py:714 `with_allowlist`
- links：constraint→EK-12；mechanism→EK-36；causal→EK-16

**EK-38 加密序列化**（L2）——EncryptedSerializer 包装内层 serde + cipher（AES-GCM 类）；加密实现不破坏类型化接口。
- 证据：checkpoint/serde/encrypted.py
- links：mechanism→EK-36（同为 serde 包装）；contrast→EK-37（加密 vs 白名单两种安全手段）

**EK-39 BaseStore 长时记忆**（L2）——namespace/key 两级寻址的 Item 存储；get/search（含 match_conditions/TTL）/put/delete/list_namespaces + batch（Op 原子批）；与 checkpoint（短时）互补为长时跨线程记忆。
- 证据：store/base/__init__.py:708 `BaseStore`；:157 GetOp / :368 ListNamespacesOp / :431 PutOp
- links：mechanism→EK-12（同为可插拔存储接口）；contrast→EK-11（长时 vs 短时）；subsystem→EK-40（store 实现族）

**EK-40 checkpoint-conformance 一致性契约**（L2）——跨 checkpointer 实现（memory/postgres/sqlite）运行同一契约测试套件（put/get/list/versions 语义），防止实现漂移；依赖图显示 checkpoint 为底座。
- 证据：libs/checkpoint-conformance/；AGENTS.md 依赖图
- links：dependency→EK-07（delta 通道一致性）；constraint→EK-12（契约约束实现）；subsystem→EK-39

## 高层框架与治理簇

**EK-41 create_react_agent 工具循环 agent**（L2）——高层 API：model+tools → 循环"模型调用→工具执行"直到停止条件；v1/v2 版本；已标记 deprecated→create_agent（langchain 包，中间件体系）；支持动态模型选择（(state, runtime)→model）。
- 证据：prebuilt/langgraph/prebuilt/chat_agent_executor.py:278
- links：dependency→EK-42（ToolNode）；dependency→EK-05（消息 Topic）；subsystem→EK-44（prebuilt 高层）

**EK-42 ToolNode 工具执行节点**（L2）——把工具集封装为节点：ToolMessage 回写状态；支持并行工具调用；tool_call 流式（_tool_call_stream）。
- 证据：prebuilt/langgraph/prebuilt/tool_node.py；_tool_call_stream.py
- links：dependency→EK-41；mechanism→EK-34（工具并行=Send 扇出变体）；subsystem→EK-41

**EK-43 ManagedValue 托管状态**（L2）——跨节点托管值（非通道）：IsLastStepManager/RemainingStepsManager 由 schema 注解推导；生命周期绑定 run 而非 thread。
- 证据：managed/base.py:18 `ManagedValue`；managed/is_last_step.py
- links：contrast→EK-17（通道 vs 托管推导）；subsystem→EK-17

**EK-44 AGENTS.md 工程治理**（L2）——monorepo 多库贡献规则：库内改动跑该库 format/lint/test；跨库改动全量验证；Corridor `analyzePlan` 安全分析（若可用）先于提交。
- 证据：AGENTS.md
- links：mechanism→EK-40（契约测试治理）；constraint→EK-41（高层改动须过底层测试）

**EK-45 依赖分层**（L1）——checkpoint 是底座（postgres/sqlite 基于它）；langgraph 依赖 checkpoint/prebuilt/sdk；prebuilt 依赖 langgraph——依赖方向单向（无环）。
- 证据：AGENTS.md 依赖图；pyproject dependencies
- links：dependency→EK-40/EK-12（上层依赖底座）；subsystem→EK-44

## 测试揭示行为簇

**EK-46 test_pregel 核心行为测试**（L2）——9,856 行核心测试（+async 9,718）：并发写冲突、interrupt 恢复、checkpoint 回放、条件边路由——揭示系统真正重视的行为是"版本化状态转换的正确性"。
- 证据：tests/test_pregel.py（9,856 行）；tests/test_pregel_async.py
- links：mechanism→EK-47/EK-48（测试揭示族）；constraint→EK-01（循环行为被测试锁定）

**EK-47 delta channel 专项测试族**（L2）——test_delta_channel_{fork, migration, id_stability, update_state, subgraph, supersteps_bound, exit_mode, benchmark}：增量通道在 fork/迁移/update_state/子图/超步边界下的正确性被单独锁定——该区域修复史最密集。
- 证据：tests/test_delta_channel_*.py 系列
- links：dependency→EK-07；causal→EK-40（契约测试覆盖）；subsystem→EK-46

**EK-48 time-travel 调试测试**（L2）——get_state_history 全量历史回放测试（3,966 行）：checkpoint 链允许"回到过去任一 checkpoint 重建状态"——time-travel 是架构副产品而非独立功能。
- 证据：tests/test_time_travel.py；pregel/main.py:1510 `get_state_history`
- links：mechanism→EK-11（checkpoint 链支撑回放）；causal→EK-18（update_state 产生分支历史）；subsystem→EK-46

## EK Graph 汇总
- 总 EK：48；links：96（每条 ≥1）
- 边分布：mechanism 24 / subsystem 14 / causal 13 / dependency 10 / constraint 14 / contrast 6
- 游离 EK：0；聚合规则覆盖率：100%（见 03）

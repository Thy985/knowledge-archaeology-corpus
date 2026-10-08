# 05 Candidates — LangGraph 未验证假说

> 与 KO 严格区分：以下内容证据不足或未验证，保持 Hypothesis 状态，不冒充 Fact/Pattern/Principle。

## C-01 [Hypothesis] 分布式 checkpointer 下的乐观并发冲突行为
- 假说：postgres 实现下，跨进程并发写同一 thread 可能产生版本冲突；引擎依赖版本单调性，冲突时后写者胜（last-writer-wins）或抛错——具体行为未知。
- 当前证据：版本机制为内存语义（_algo.py:227）；postgres 实现未深读。
- 缺失证据：libs/checkpoint-postgres 的事务/锁/重试细节。
- 验证路径：读 postgres saver 的 put/put_writes 事务实现；或跑双进程并发 invoke 实验。

## C-02 [Hypothesis] 大扇出下 versions_seen 触发精确性
- 假说：Send 动态扇出（数百任务）时，versions_seen 按"触发通道版本"记账可能在扇出任务间产生额外重复执行。
- 当前证据：prepare_push_task_send（_algo.py:938）为每 Send 建独立任务；触发语义依赖版本差。
- 缺失证据：无大规模扇出测试断言。
- 验证路径：构造 1000-Send 图，检查任务数是否精确等于 Send 数。

## C-03 [Hypothesis] STRICT_MSGPACK 白名单的向后兼容策略
- 假说：strict 模式启用后，未列入 allowlist 的历史 checkpoint 类型将反序列化失败；框架通过 warning 降级而非硬失败（with_allowlist 注释）。
- 当前证据：compile 时构建 allowlist（state.py:1233-1254）；不支持序列化器仅 warning。
- 缺失证据：strict 模式默认值、迁移路径文档。
- 验证路径：读 _serde.STRICT_MSGPACK_ENABLED 默认值及迁移测试。

## C-04 [Hypothesis] subgraph 状态隔离语义
- 假说：subgraph 与父图共享 saver 但 checkpoint_ns 隔离；子图 channel 与父图 channel 无直接共享（除显式传递）。隔离边界细节未深读。
- 当前证据：_put_checkpoint 记录 parents 映射（_loop.py:1187）；subgraph checkpointer 继承（state.py:1196）。
- 缺失证据：subgraph 编译（CompiledStateGraph 嵌套）的 channel 映射实现。
- 验证路径：读 graph/_node.py 的 subgraph 处理与 test_subgraph 系列。

## C-05 [Hypothesis] durability="exit" 模式的权衡
- 假说：exit 模式仅在退出写 checkpoint → 吞吐高但中断恢复粒度粗（interrupt 点可能无 checkpoint）；每超步模式反之。
- 当前证据：do_checkpoint = saver 存在 and (exiting or durability != "exit")（_loop.py:1194-1196）。
- 缺失证据：exit 模式与 interrupt 组合行为的文档/测试。
- 验证路径：跑 exit 模式 + interrupt 图，检查恢复点粒度。

## C-06 [Hypothesis] delta 快照频率默认值的工程权衡
- 假说：DeltaChannel 计数器（updates, supersteps）驱动快照的阈值是工程权衡：快照太频→存放大；太疏→恢复需重放长链。默认行为可能无显式阈值（每超步计数但快照频率受限于存储策略）。
- 当前证据：计数器机制（_loop.py:1157-1226）；test_delta_channel_supersteps_bound 存在（边界被测试）。
- 缺失证据：快照频率的默认配置参数。
- 验证路径：读 delta.py 与 checkpoint 存储的快照触发条件。

## C-07 [Hypothesis] LangGraph v1 迁移的中间件系统差异
- 假说：create_agent（langchain 包）的中间件体系替代 create_react_agent 的图配置，迁移会改变扩展方式（中间件 vs 显式节点）；对既有 react agent 生态的兼容性未知。
- 当前证据：create_react_agent 标记 deprecated 并指向 create_agent（chat_agent_executor.py:278 附近 docstring）。
- 缺失证据：langchain 包 create_agent 实现（不在本仓库）。
- 验证路径：读 langchain 包对应实现或迁移指南。

## 跨项目连接（供 corpus 对照，非本仓库结论）
- CP-01 [Hypothesis]：langgraph 的"版本化状态机"与 letta-code 的 memory 层 / mem0 的实体记忆存在互补（状态机时序 vs 语义记忆）——三者均已在 corpus，可做跨项目模式对照（暂不升维）。
- CP-02 [Hypothesis]：omnigent 的 supervisor 模式若用 langgraph 实现，其状态设计可复用 KO-01 的版本化状态机模型（与雷达 next-step TeamMind 隔离 vs 共享 直接相关）。

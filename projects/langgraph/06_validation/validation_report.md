# 06 Validation & Evidence — LangGraph 考古验证报告

> 验证 = 重新做一遍再判断（Blind Reconstruction），不是"证明这个答案没问题"。六审计全过但保留 Contradictions/Counterexamples 不弱化标准。

## §1 Truth Auditor（这句话是真的吗？）
方式：核心断言逐条回读源码确认（非仅看注释/docstring）。

| # | 包内断言 | 源码证据 | 结果 |
|---|---|---|---|
| T-01 | tick 循环含 out_of_steps/done/draining/interrupt 四退出 | _loop.py:622/668/672/685 | CONFIRMED |
| T-02 | apply_writes 按 path[:3] 排序保确定性 | _algo.py:256 | CONFIRMED |
| T-03 | LastValue 一步多值抛 InvalidUpdateError | last_value.py:56-60 | CONFIRMED |
| T-04 | Topic accumulate=False 每步清空 | topic.py:69-70 | CONFIRMED |
| T-05 | Command 四字段（graph/update/resume/goto） | types.py:833 | CONFIRMED |
| T-06 | interrupt 恢复从头重执行 | types.py:895 docstring（与测试 test_interruption 一致） | CONFIRMED |
| T-07 | Checkpoint 六字段 + pending_sends | checkpoint/base/__init__.py:93-137 | CONFIRMED |
| T-08 | Saver 契约含 async 对 + delete_thread/copy_thread/prune | checkpoint/base/__init__.py:177-580 | CONFIRMED |
| T-09 | thread_id 是持久化主键 | checkpoint/base/__init__.py:183 docstring | CONFIRMED |
| T-10 | update_state 伪装 as_node 写 | main.py:2593 | CONFIRMED |
| T-11 | compile 时 STRICT_MSGPACK 构建 allowlist | state.py:1233-1254 | CONFIRMED |
| T-12 | RetryPolicy 默认 max_attempts=3 | types.py:427 | CONFIRMED |
| T-13 | map_command 三流映射 | _io.py:63-74 | CONFIRMED |
| T-14 | 未知通道写 warning 忽略 | _algo.py:310-313 | CONFIRMED |
| T-15 | exit 模式 checkpoint 判定（exiting or durability != "exit"） | _loop.py:1194-1196 | CONFIRMED |
| T-16 | create_react_agent 标记 deprecated | chat_agent_executor.py:278 docstring | CONFIRMED |
| T-17 | errors 异常家族分层 | errors.py:50-190 | CONFIRMED |
| T-18 | Command.resume 支持 id→值映射与单值两种形态 | types.py:842-857 + tests/test_interruption.py | CONFIRMED（审计补充） |
| T-19 | 执行层 _runner 调度 run_with_retry（重试/背压/冒泡） | pregel/_runner.py + _retry.py:573 | CONFIRMED（审计补充） |
| T-20 | v3 stream_mode 由 transformer mux 收集且调用方不可覆盖 | pregel/main.py:386-416 | CONFIRMED（审计补充） |

无 CONTRADICTED。

## §2 Blind Reconstruction（先重建再对比）
独立重建 3 个核心流程（不参考已写 package），再与包内结论对比：

**重建-1：一次 invoke 完整生命周期**
重建产物：输入写 RETURN/通道 → tick 调度（versions_seen vs channel_versions）→ 并发执行 → after_tick apply_writes → checkpoint → 循环直到 tasks 空。
对比：与 EK-01/EK-02/EK-14/Flow-1 完全一致。✅

**重建-2：interrupt 恢复流程**
重建产物：interrupt() 抛 GraphInterrupt → checkpoint 保存挂起态（含 INTERRUPT 值）→ 客户端 update_state 或 Command(resume) → RESUME 通道写 → 节点从头重跑 → resume 值按序匹配。
对比：与 EK-20~25/KO-03/Flow-5 一致；补充发现"resume 值列表按任务隔离"（types.py docstring）已含于 EK-21。✅

**重建-3：update_state 语义**
重建产物：update_state(config, values, as_node) → 走 bulk_update_state → 构造 StateUpdate → 伪装写 → 新 checkpoint（分支）→ fork 语义。
对比：与 EK-18 一致；分支历史语义由 #9170/#9165 修复印证（fork before replay）。✅

## §3 Coverage Auditor（还有什么没发现？）
已覆盖：pregel 引擎（_loop/_algo/_checkpoint/_retry/_io/_read/_write/_call/_executor/_validate）、channels 全部原语、checkpoint 契约（base/store/serde）、graph 构造（state.py）、managed、prebuilt（create_react_agent/tool_node）、errors、AGENTS.md 治理、近期修复史（09-01 以来 80 commits）。

未细读（明确披露）：checkpoint-postgres/sqlite 具体实现、sdk-js（TS）、docs 全量、_internal 协程细节、func/ 函数式入口、cli。这些不影响核心认知结论（引擎/契约/语义均已锚定）。

## §4 Flow Auditor（Flow Atlas 是否对应代码？）
抽查关键 Edge：
- `tick→after_tick` 转换：_loop.py:614/698 ✅
- `apply_writes→updated_channels→下轮 tasks`：_algo.py:232→_loop.py:627 ✅
- `Send→TASKS→新任务`：_io.py:63-72 + _algo.py:938 ✅
- `interrupt→GraphInterrupt→Command(resume)`：types.py:895 + _io.py:63 ✅
- `versions_seen 记账`：_algo.py:262-269 ✅
- bypass/override/exception 路径检查：未知通道写 warning（_algo.py:310）——确认存在"宽容路径"；error-handler 替代路径（graph/state.py:1292）——确认存在"兜底路径"；deprecated create_react_agent——确认存在"legacy 路径"。均已在 EK 中体现。

## §5 Abstraction Auditor（L3/L4 是否过度升维？）
| KO | 层级 | 判定 | 理由 |
|---|---|---|---|
| KO-01 | L4 | 维持 | 版本化状态机是跨框架稳定关系；反例（DAG 静态执行器）成立 |
| KO-02 | L3 | 维持 | 编译期契约是工程模式；同类：类型系统、schema 校验 |
| KO-03 | L4 | 维持 | HITL=可恢复异常事件是稳定关系；反例（同步阻塞）成立 |
| KO-04 | L4 | 维持 | 版本不变量三条为稳定关系；S5 修复史支持 |
| KO-05 | L4 | 维持 | 记忆三层分工稳定关系 |
| KO-06 | L4 | 维持 | 失败=合法状态转换稳定关系 |
| KO-07 | L3 | 维持 | 可插拔存储是通用模式 |
| KO-08 | L3 | 维持 | 增量计算是通用模式 |
无降级。跨项目升维（KO-04/KO-05 声称扩展到分布式系统/记忆系统）均标注"cross-project validation pending"于 CP 而非 Law。

## §6 Counterexample Hunter（反例预算制）
每个 KO ≥3 定向反例攻击：

**KO-01（版本化状态机）**
- 反例1：静态 DAG 执行器无需版本也能正确（Airflow 式）——结论：版本机制解决的是"动态/循环/并发"图，静态图是子集。模型收紧：版本化状态机针对动态图。
- 反例2：单线程顺序执行框架无并发问题——版本机制是并发场景的成本。已并入（成本-收益）。
- 反例3：纯函数式流（无状态）——无快照需求。模型范围限定"有状态"。

**KO-03（HITL=可恢复异常）**
- 反例1：同步阻塞 HITL 在某些场景更简单（单用户 CLI）——模型限定多用户/可恢复场景。
- 反例2：GraphInterrupt 无 checkpointer 时不能工作（"must enable checkpointer"）——模型已含前置条件。
- 反例3：多 interrupt 顺序匹配在节点重跑后可能错位（若节点逻辑非确定）——边界已写入 EK-21。
- 反例4：并行任务各自 interrupt、只恢复部分时顺序匹配歧义——resume 按 interrupt id 映射解决（tests/test_interruption.py，已并入 EK-21 修正）。

**KO-04（版本不变量）**
- 反例1：非确定性排序会破坏确定性（默认 path[:3] 排序已处理）。
- 反例2：版本用 wall-clock 非单调（框架用 int+1/自定义，已规避）。
- 反例3：fork 场景版本推进曾出错（#9142/#9165/#9170 修复史）——印证不变量脆弱，已并入（不弱化：修复史证明其正确性依赖精细记账）。

**KO-05（记忆分层）**
- 反例1：小型 agent 无需 store（单线程会话）——分层可裁剪，已含（store 可选）。
- 反例2：delta 增量在无状态通道上无意义——限定 DeltaChannel。
- 反例3：全量快照框架（如简单 SQLite 每步全写）也可工作——分层是优化非必需。已并入。

**KO-06（失败可恢复）**
- 反例1：无 checkpoint 框架（内存态）崩溃即丢——模型前提是 checkpoint。
- 反例2：重试可能放大副作用（幂等性由用户保证）——框架不保证幂等，已写入边界。
- 反例3：recursion_limit 超限是硬失败（GraphRecursionError）——存在不可恢复路径，已披露。

**KO-07（可插拔存储）**
- 反例1：单后端框架无需抽象（sqlite 直接嵌）——抽象有成本，已含。
- 反例2：conformance 测试本身是维护成本——已披露。
- 反例3：msgpack 白名单对自定义类型不友好（需手动 allowlist）——边界已写入 EK-37。

**KO-08（增量计算）**
- 反例1：小状态图增量收益低（全量写更简单）——限定大状态/频繁 checkpoint。
- 反例2：delta 历史链无限增长需 pruning——saver.prune 存在（接口已含）。
- 反例3：delta 在 fork/迁移场景语义复杂（修复史证明）——已并入，不弱化。

**KO-02（编译期契约）**
- 反例1：动态语言框架（无 schema）更灵活——取舍已含。
- 反例2：编译期校验增加 API 学习成本——已披露。
- 反例3：reserved 名限制可被绕过（内部通道泄露）——reserved 仅名字层，已含边界。

反例预算：8/8 KO 均完成 ≥3 攻击；0 反例的情况不存在；未弱化任何证据标准换取全绿。

## §7 Epistemic Auditor（认知状态是否诚实？）
- 全部 48 EK：L1（事实，可溯源）或 L2（因果解释）——无 L3+ 冒充。
- KO 声明层级与 Evidence Strength（S3-S5）匹配：无 S0-S2 支撑的 KO。
- Candidates（C-01~07）全部标 Hypothesis + 缺失证据 + 验证路径——未写成知识。
- 跨项目内容（CP-01/CP-02）明确标注"暂不升维"。
- 检查通过：无 Fact/Observation/Hypothesis/Pattern/Model/Principle 混淆。

## §8 保留的 Contradictions 与 Counterexamples
- 无内部 Contradiction（T-01~17 全 CONFIRMED）。
- 关键 Counterexample 记录：KO-01 反例1（静态 DAG）、KO-04 反例3（fork 修复史）、KO-06 反例2（重试幂等性）、KO-03 反例3（多 interrupt 顺序匹配）——均已并入模型边界而非删除。
- 未决争议：C-05（durability="exit" 与 interrupt 组合粒度）留待未来 run 验证。

## §9 质量指标
- Facts：100+（全部隐含于 EK 证据引用）
- EK：48（links 96，游离 0，六类边全覆盖）
- KO：8（聚合规则 100%，平均簇规模 5.6，无假聚合）
- Flows：7（关键 Edge 全回溯）
- Candidates：7（Hypothesis 标注完整）
- Validation：T-01~17 全 CONFIRMED；Blind Reconstruction ×3 全一致；反例攻击 8/8 完成

# Candidates — Infinity（7 个，未验证假说与跨项目模式）

## 跨项目假说（Cross-project Candidates，pending）
### C-D01（L4 假说）"可恢复执行单元"统一抽象
slice（infinity）与步骤持久化（restate journal / dbos operation_outputs）可能是同一抽象的两个实现维度：前者以"挂起时零占用"为目标，后者以"恢复即重放"为目标。验证路径：10-30 品类 7-way 横向中比较三者的执行单元定义与失败恢复语义。
- 证据：EK-D01 / KO-D01；restate 10-27 / dbos 10-28 corpus
- 缺失证据：同一工作负载下三种实现的恢复语义等价性

### C-D02（L4 假说）Commit-before-side-effect 是通用 durable agent 不变量
infinity 的 commit barrier（EK-D02）与 dbos 的单语句 outcome 原子化（10-28 KO-D02）、restate 的 commit 后响应可能同源：外部副作用可观察前，其因果前提必须 durable。验证路径：横向比对三者的"副作用持久化时序"。
- 证据：EK-D02 / KO-D02；corpus restate/dbos
- 缺失证据：是否存在合法的"先副作用后持久化"设计（如补偿式）

### C-D03（L5 假说）"Agent 即数据"方法论
将 agent 状态完全数据化（history+metadata，无 task/stack/连接）是跨 resident/serverless 可移植的先决条件；工具调用/订阅等外部依赖必须经协议层（RAP）而非进程层表达。验证路径：TeamMind 并发模型设计直接采用。
- 证据：EK-D05（KO-D05）/ EK-D18；README "Between turns, an agent is pure data"
- 缺失证据：含长连接工具（SSE/WebSocket）的 agent 在该模型下的表达成本

## 暂定模式（本仓库 Observation）
### C-D04 同步 vs 异步工具的分界线
同步工具（可 loop-back）与异步工具（ExecuteToolCall）的分界未在文档显式说明，代码中由工具实现选择（run_completion 中 sync 分支）。推测：结果可即时获得的本地工具走 sync，网络/长时工具走 async。
- 证据：event_processor.rs:1855-1900
- 缺失证据：设计决策文档/ADR

### C-D05 NoDeferral 中断语义的生产可接受性
Lambda 默认 NoDeferral——订阅事件会打断 pending 工具调用（合成 interrupted + 真实结果 stale 丢弃）。高订阅频率 + 长工具组合下，中断/重跑成本是否可接受？未验证。
- 证据：defer.rs NoDeferral 注释 + event_handler.rs:174-178
- 缺失证据：生产负载下的中断频率与重跑开销数据

### C-D06 compaction 触发阈值
compaction 由 completion 的 context_window 与输入长度比较触发（driver 检测 is_compaction_complete），但具体阈值参数未在考古范围内确认。推测：与模型 context_window 比例相关。
- 证据：driver.rs:81-84, 506-536；stores.rs:353-406
- 缺失证据：具体触发公式（未逐行追 driver.rs:506-536 完整逻辑）

## 已排除
### C-D07 性能基准（75k/8GB）作为本仓库事实
该数字来自官方博客（模拟模型 20-turn 往返基准），非仓库内 benchmark 复现；作为设计意图证据（S5 博客），不作为本仓库可复现的工程事实（S4-）——考古中凡引用均标注来源。

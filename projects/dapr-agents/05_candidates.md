# Candidates — Dapr Agents（6 个）

## 跨项目假说（Cross-project Candidates，pending）
### C-D01（L4 假说）"执行步骤持久化" vs "状态挂起"是 durable agent 的两个实现端点
Dapr（步骤 activity 持久化 + 重放恢复）与 infinity（状态纯数据 + slice 挂起恢复）都实现"崩溃恢复 + 等待零占用"，但恢复语义不同：重放不重执行 vs 从最后 slice 继续。推测：两者可用统一抽象描述（可恢复执行单元），差异在"恢复成本 vs 内存成本"的权衡轴。验证路径：同一工具调用工作负载在 Dapr/infinity/restate/dbos 四路线的恢复成本对比。
- 证据：KO-D01/D05；corpus restate/dbos/infinity（10-27~10-29）
- 缺失证据：四路线同一负载的基准数据（无 Docker/容器环境未实跑）

### C-D02（L4 假说）HITL 审批必须与重放兼容
审批事件（ApprovalRequiredEvent/ResponseEvent）用确定性 id 关联暂停点——若审批事件 id 不确定，重放会匹配错误。推测：任何 durable agent 的 HITL 设计都必须保证审批事件的确定性可关联。验证路径：infinity 的 interrupted 合成事件（确定性 id）与 dbos 的审批（若有）对照。
- 证据：EK-D05 + durable_executor.py（deterministic ids 注释）
- 缺失证据：跨框架 HITL 实现对照

### C-D03（L4 假说）工具副作用必须可重入（activity 至少一次）
executor 至少一次交付 + 会话恢复意味着已执行工具可能重跑——工具副作用必须可重入/幂等，否则恢复即重复副作用。推测：durable agent 框架的"至少一次"交付将幂等责任推到工具层。验证路径：infinity 的"dispatch barrier + error result"（失败不悬挂）与 Dapr 的至少一次对照。
- 证据：EK-D06（模块 docstring delivery semantics）
- 缺失证据：工具幂等违约的故障模式实测

## 暂定模式（本仓库 Observation）
### C-D04 AgentRunner 三模式的边界
run/subscribe/serve 三模式共享 workflow 入口，但 serve 额外挂 HITL 路由 + 流式；推测：三模式是同一 execution core 的三个入口面，未来可能收敛（如 subscribe+serve 组合部署）。未在文档确认。
- 证据：workflow/runners/agent.py:206,871,911
- 缺失证据：部署形态文档/ADR

### C-D05 AgentStateConfig schema bundle 的扩展边界
entry_factory/message_coercer 可自定义状态模型（bundle 注入），但"已注入不同 bundle raise RuntimeError"表明单 agent 单状态模型。推测：复杂 agent（多角色混合）可能需要多 bundle 支持。未验证。
- 证据：configs.py:246-312
- 缺失证据：多 bundle 需求场景

### C-D06 重放日志抑制的成本
WorkflowReplayFilter 用 contextvars 标记重放态，但仅覆盖经 with_logger_context 包装的函数；未包装代码的日志在重放时会重复。推测：框架内 logger 调用点覆盖率需审计。未验证。
- 证据：utils/logger.py:53-79
- 缺失证据：logger 调用点全覆盖审计

## 已排除
（无——本仓库候选均保留观察）

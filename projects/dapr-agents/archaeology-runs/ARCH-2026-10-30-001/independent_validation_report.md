# Independent Validation Report — ARCH-2026-10-30-001

> Auditor 盲重建：独立重读仓库核心文件（durable.py / durable_executor.py / agent.py / activity_bridge.py / configs.py / logger.py）后对比判定。禁止修改原考古产物；修正经 Reconciliation 写入 corpus。

## 判定统计
| 判定 | 数量 | 项 |
|---|---|---|
| CONFIRMED | 12 | IV-1..IV-12（见下） |
| CONTRADICTED | 0 | — |
| PARTIALLY_CONFIRMED | 1 | IV-13（"每步 activity 持久化"对标准循环需限定：ctx 工具内联执行） |
| DOWNGRADED | 0 | KO-D05 已标 Cross-project pending，风险已缓解 |
| MISSING | 4 | storage 三族实现 / document 模块 / observability wrappers / ext（drasi/claude）集成——与主考古 Coverage=PARTIAL 一致 |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 12 CONFIRMED（重点）
1. **agent_workflow 三分支**：durable.py:631-760（orchestrator 子 workflow / _executor_workflow / 标准循环）
2. **executor-as-activity 至少一次**：durable_executor.py:36-58 模块 docstring 明示 delivery semantics（retry 恢复会话、METADATA_RETRYABLE=False fail）
3. **HITL 审批闭环 + 确定性 id**：approval_request_id = uuid5(f"{instance_id}:{tool_call_id}") 重放一致（durable.py:1221-1290）；PausedExecutorRun → 发布事件 → wait external event → _resume_input 恢复（durable_executor.py:141-218）
4. **审批上限自动拒绝**：round_ > max_approval_rounds → break → _LIMIT_REASON → 最后机会（durable_executor.py:177-218）
5. **重放日志抑制**：WorkflowReplayFilter + with_logger_context（contextvars is_replaying）（utils/logger.py:24-79）
6. **双 state store + schema bundle**：AgentStateConfig.ensure_bundle 注入 StateModelBundle（configs.py:246-312）；DaprSessionStore（executors/dapr_session_store.py）
7. **agent-as-tool = child workflow**：AgentWorkflowTool ctx.call_child_workflow + _child_instance_id 确定性（tool/workflow/agent_tool.py:118-228）
8. **bridge 限制**：activity 内跨 app 子 workflow 调度 raise ToolError（activity_bridge.py:298-308）
9. **标准循环工具执行双路径（补充发现）**：普通工具 = activity_tasks（ctx.call_activity 并行）；WorkflowContextInjectedTool = 内联 workflow body（"cannot be ran within an activity bc activities do not have the workflow context"）（durable.py:840-960）；all_tasks = workflow_tasks + activity_tasks 统一调度
10. **AgentRunner 三模式**：run（wait=False 返回 instance_id）/ subscribe（TTLDedupeBackend 去重）/ serve（FastAPI + HITL 路由 + 流式）（agent.py:206,871,911）
11. **TriggerAction 消息 + message_router + on_broadcast**：schemas.py:171-183；durable.py:1962-1986
12. **orchestration 5 activity 链**：initialize→select_next_task→handle_response→check_completion→finalize（durable.py:1879-1947）

## 1 PARTIALLY_CONFIRMED
- IV-13："每步 activity 持久化"对 executor 分支与 call_llm 严格成立；标准循环中 **WorkflowContextInjectedTool 内联执行**（非 activity）——它们是"需要 ctx 的确定性 workflow 操作"（call_child_workflow 本身即持久化原语），故非违规，但表述需限定。

## 3 计数类复核（攻击目标，全部通过）
1. EK 20 / 平均出边 2.35：python 重算 47 边 → 2.35 ✓（游离 0）
2. Flow 28 Edge：grep `^| (CF|SF|DF|EF|AF|MF|PF)-` = 28 ✓
3. KO 7 / 层级分布 L3×2/L4×4/L5×1：声明核对 ✓

## 补充发现（Observation，Reconciliation 合入）
- **approval_request_id 生成**：uuid5(NAMESPACE_DNS, f"{instance_id}:{tool_call_id}")——确定性关联审批事件与暂停点（durable.py:1243-1246）
- **工具执行双路径**（见 IV-9）：标准循环中普通工具批量 activity、ctx 工具内联——写为 EK 边界
- **hook 路径 vs executor 路径审批**：hook RequireApproval（_request_approval，durable.py:1201）与 executor 暂停审批（_await_approval 变体）是两条审批入口，共用 ApprovalRequiredEvent/ResponseEvent 模型——写为 EK 对照

## 遗漏（MISSING，主考古 Coverage=PARTIAL 声明确认）
- storage 三族（daprstores/graphstores/vectorstores）实现细节
- document 模块（fetcher/reader/splitter/embedder）
- observability/wrappers（OpenTelemetry 封装）
- ext 集成（drasi/claude，DAPR_AGENTS_REQUIRE_DRASI=1）

## Benchmark case（供 skill CI）
**Case**: "多 agent 系统中，进程崩溃后如何保证长时程 agent 工作流不从头重跑、不重复执行已完成 LLM/工具调用？"
- 期望：workflow 步骤持久化 → 崩溃 → 重启 → 从最后持久化步骤继续（重放不重执行）
- Dapr Agents 证据：executor-as-activity（durable_executor.py:36-58）+ 双 state store（configs.py:246）+ 确定性 id（uuid5 审批 id / child instance id）+ HITL 审批（durable.py:1221-1290）
- 反例面：无 Docker 未实跑中断 demo（S5 待补）；普通工具 activity 至少一次 → 工具副作用可重入性依赖工具实现（C-D03）
- 对照：infinity（slice 挂起恢复）/ restate（journal 重放）/ dbos（数据库 checkpoint）——四种"崩溃后续跑"实现路径

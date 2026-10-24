# Independent Validation Report — ARCH-2026-10-25-001（dapr-agents）

> Auditor 盲重建原则：不把考古结果当事实来源；独立重读仓库（durable.py 三分支区/重放检测/审批全链路/hooks 定义/lifecycle/orchestration 主体/sstore）建立 Independent Findings 后与 Package 对比。未修改任何考古产物。

## 判定统计（13 项）
| 判定 | 数量 | 项 |
|------|------|----|
| CONFIRMED | 11 | #1,#2,#3,#4,#5,#6,#7,#8,#11,#12,#13 |
| PARTIALLY_CONFIRMED | 1 | #9 |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 1 | #10 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 逐项判定
1. **CONFIRMED** — 三分支执行模型：独立重读 durable.py:660-716，顺序 orchestrator（orchestration_mode）→ executor（self.executor is not None）→ 标准循环（max_iterations）；EK-D01/D03 准确
2. **CONFIRMED** — call_llm 重放检测：独立重读 durable.py:2547-2575，确定性 ID = instance + activity attempt key（task_execution_id，稳定跨 re-delivery 与 retry）+ input（隔离跨 run 复用）；EK-D02 机制正确且证据锚全对
3. **CONFIRMED** — HITL 审批链路：独立重读 durable.py:1215-1260 + 1286，uuid5 确定性 / publish 缓存 / timer 竞速 auto-deny / 解析失败 auto-deny；EK-D05 全对
4. **CONFIRMED** — hooks 五态 + dispatch 前收集：独立重读 hooks.py:69-125 + durable.py:772-816，Proceed/Skip(result 直用)/Mutate(替换 payload)/RequireApproval(通用触发，非仅 decorator)/Deny(合成 ToolMessage)；EK-D04 主体正确
5. **CONFIRMED** — skip_messages 双模式：独立重读 durable.py:3054-3110 docstring；EK-D09 准确
6. **CONFIRMED** — MCP auto-discovery：独立重读 base.py:530-547；EK-D14 准确
7. **CONFIRMED** — lifecycle OSS-clean + intentionally tiny：独立重读 lifecycle.py:24-43；EK-D15 准确
8. **CONFIRMED** — DaprSessionStore manifest+chunk：独立重读 sstore:171-275；EK-D12 准确
9. **PARTIALLY_CONFIRMED** — EK-D18 证据锚不完整：orchestration_workflow **主函数**在 durable.py:1332（get_team_members → plan_prompt → call_llm(IterablePlanStep, STREAM_PHASE=planning)）；考古只引策略活动区（1871-1955）→ **R-1 修正**
10. **MISSING** — executor 实现文件未点出：`_executor_workflow` 定义在 **dapr_agents/agents/durable_executor.py:121**（非 durable.py）；EK-D13 只引 durable.py:694-698（调用点）→ **R-2 修正**
11. **CONFIRMED** — Mutate 三 slot 语义：独立重读 hooks.py:86-103——before_tool_call **完全替换**工具参数、before_llm_call **浅合并** generate kwargs、after_llm_call **替换** assistant 消息；考古"改写参数"粒度不足 → **R-3 细化**
12. **CONFIRMED** — orchestration 结构化规划：独立重读 durable.py:1332-1360，TASK_PLANNING_PROMPT + response_format=IterablePlanStep + STREAM_PHASE=planning；EK-D18 主体事实成立
13. **CONFIRMED** — 测试覆盖存在：tests/agents/durableagent/ 下 test_durable_agent.py / test_durable_executor.py / test_durable_executor_approval.py / test_hitl.py / test_hitl_persistence.py / test_hitl_workflow.py / test_llm_hooks.py / test_raise_approval_event.py → Benchmark 可复跑

## 3 个成功（盲重建独立成立）
- **S1**: call_llm 重放检测的 key 设计（instance + task_execution_id + input 三重隔离）——是 activity 幂等的最佳实现证据
- **S2**: HITL 全链路（确定性 ID → 发布缓存 → 事件竞速 → auto-deny → 持久化恢复 _persist/_restore_pending_approvals）——fail-closed 闭环
- **S3**: skip_messages 双模式（消息链完整性与观测并存的工程取舍）

## 3 个错误/修正
- **E1**（→R-1）: EK-D18 证据锚缺 orchestration_workflow 主函数（1332）
- **E2**（→R-2）: EK-D13 未点出 executor 实现文件（durable_executor.py:121）
- **E3**（→R-3）: EK-D04 Mutate 未区分三 slot 语义（hooks.py:86-103）

## 遗漏（Auditor 独立发现，未在考古 02-05 覆盖）
- **O1**: orchestration_workflow 主体活动序列（1332-1550 区域：plan → IterablePlanStep → agent loop → check_completion/finalize）未逐函数核对
- **O2**: AgentRunner HTTP 路由契约（register_routes / _build_pubsub_specs / discover_entry，runners/agent.py:687-728）未深读
- **O3**: lifecycle DecisionDict 的 forward-looking approver-authorization 字段内容未读
- **O4**: 内置工具注册表（durable.py:408 built-in ask_user/advance_team）未整理
> O1-O4 已记入 05_candidates C-D08~C-D11（Hypothesis 级）

## 攻击重点专项结论
| 攻击面 | 结论 |
|--------|------|
| 单案例→Pattern | KO-D02 有 MXC/strands-box 跨项目证据支撑，成立；KO-D03 仅 dapr 内多子系统，保持 L3 Pattern（不升 Principle） |
| Pattern→L4 | KO-D01/D05/D06 全标 `Cross-project validation pending`，无过升维 |
| 项目经验→通用 Principle | 未发现冒充 Principle 的条目 |
| ADR→实现事实 | 无 ADR 目录证据（docs/ 以 getting-started/development 为主）；README 特性与代码一致，无文档-实现矛盾 |
| Flow Edge 真实性 | 7 类流 12 个关键 Edge 全部可回溯文件:行 |
| bypass/override/exception/alternate | ask_user 无流短路（835）、WorkflowContextInjectedTool 内联守卫（3008-3022）、executor 分支、retry-policy 全 activity 覆盖——异常路径均有处理且已记录 |
| admin path/fallback/legacy path | fallback：publish 无 pubsub→HTTP /hitl/approvals（configs.py:516-528）；schema task_execution_id 缺失 fallback task_id（2547 区域）——已记录 |
| Epistemic 混淆 | Fact/Observation/Hypothesis/Pattern/Model 严格区分；Candidates 7 条全标 Hypothesis |

## Benchmark cases（可复跑）
- **BC-DAPR-1**（重放/消息链）: `pytest tests/agents/durableagent/test_durable_agent.py tests/agents/durableagent/test_llm_hooks.py -m "not integration"`
- **BC-DAPR-2**（HITL 全链路）: `pytest tests/agents/durableagent/test_hitl.py tests/agents/durableagent/test_hitl_workflow.py tests/agents/durableagent/test_hitl_persistence.py tests/agents/durableagent/test_raise_approval_event.py tests/agents/durableagent/test_durable_executor_approval.py -m "not integration"`
- **BC-DAPR-3**（executor 事件流）: `pytest tests/agents/durableagent/test_durable_executor.py -m "not integration"`

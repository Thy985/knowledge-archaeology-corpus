# Reconciliation — ARCH-2026-10-25-001（dapr-agents）

> Independent Auditor 修正合并（不弱化证据标准；不修改生产 skill）。

## R-1（PIPELINE）· EK-D18 证据锚补全
- **修正**: EK-D18 links/证据列补 `orchestration_workflow` 主函数 durable.py:1332（get_team_members → TASK_PLANNING_PROMPT → call_llm(response_format=IterablePlanStep, STREAM_PHASE=planning)）；保留策略活动区锚（initialize_orchestration 1868 / select_next_task 1884 / check_completion / finalize）
- **依据**: Auditor #9 PARTIALLY_CONFIRMED；独立重读 1332-1360

## R-2（PIPELINE）· EK-D13 实现文件定位
- **修正**: EK-D13 补 `dapr_agents/agents/durable_executor.py:121（_executor_workflow）`；durable.py:674 为调用点；AgentExecutorBase 定义在 executors/base.py:35-112
- **依据**: Auditor #10 MISSING

## R-3（SCOPE）· EK-D04 Mutate 三 slot 语义细化
- **修正**: EK-D04 补充 Mutate 语义——before_tool_call：payload **完全替换**工具参数；before_llm_call：payload **浅合并** generate kwargs（tools/response_format/tool_choice 保留）；after_llm_call：payload **替换** assistant 消息（证据 hooks.py:86-103）
- **依据**: Auditor #11；RequireApproval 通用性（"not just ones with a decorator"）一并补入 EK-D05

## R-4（OMISSION）· Candidates 追加 4 项
- C-D08（orchestration 主体活动序列深读）、C-D09（AgentRunner HTTP 路由契约）、C-D10（lifecycle DecisionDict 前瞻字段）、C-D11（内置工具注册表 ask_user/advance_team 语义）——Auditor O1-O4，Hypothesis 级
- **依据**: Auditor 遗漏清单

## R-5（METADATA）· run_metadata validation_result 更新
- validation_result.status: pending_auditor → **passed_with_fixes**
- auditor_verdict: **PASS_WITH_FIXES（13 判定：11 CONFIRMED / 1 PARTIALLY / 1 MISSING / 0 CONTRADICTED；3 修正已合入）**

## Reconciliation 范围声明
- 合并对象：本次 Package（02 EK Graph / 05 Candidates / 06 Validation / run_metadata）
- 未改动：01 Project Layer（地图事实无争议）、00 Overview（六条主线全部 CONFIRMED）、03 KO（无降级/无过升维）
- 未改动：生产 skill / 目标仓库 / KnowlegeMap

# Infinity… no — Dapr Agents — Project Archaeology Overview（ARCH-2026-10-30-001）

> **run**: ARCH-2026-10-30-001 · repository: dapr/dapr-agents · commit `1fd68b36`（2026-10-27）· skill v3.2 · mode=initial · P0

## 一句话定位
CNCF Dapr（微软发起）× NVIDIA 联合推出的**分布式 Agent 运行时**（Python，Apache-2.0）：把 Dapr 成熟的 workflow 引擎（durable execution）+ actor + state store + pub/sub 搬到多 agent 系统——agent 的每一步推理与工具调用都持久化，进程被杀后从最后完成步骤恢复，不重复执行已完成的 LLM/工具调用。

## 六条认知主线
1. **Workflow 即执行模型**：agent 的每一步都是 Dapr workflow 步骤——executor 回合 = activity（run_executor）、HITL 审批 = wait external event、子 agent = child workflow、标准循环 = call_activity(call_llm) + 工具执行（普通工具批量 activity；需 ctx 的 workflow 工具内联执行）。agent 逻辑与执行持久化强绑定（durable.py:580-760；durable_executor.py:130）
2. **重放安全纪律（event sourcing）**：workflow 代码在 await 后重放重建本地状态，非确定性操作必须经 activity 边界；确定性 ID（审批/子 workflow）保证重放不漂移；`ctx.current_utc_datetime` 而非 `datetime.now()`（durable_executor.py:151；durable.py:700 附近）
3. **HITL 审批 = 暂停-恢复闭环**：工具调用暂停（PausedExecutorRun）→ 发布审批事件 → wait external event/timeout → 以 ToolCallDecision 恢复执行；超过 max_approval_rounds 自动拒绝并给最后机会（durable_executor.py:141-218）
4. **executor 会话至少一次交付**：run_executor 是 activity（至少一次）；重试恢复前次 executor session（任务重发、已执行工具可能重跑）；非 retryable 失败（METADATA_RETRYABLE）fail workflow 而非重试（durable_executor.py:36-58 模块 docstring）
5. **双 state store + schema bundle**：agent-memory（会话历史）与 agent-workflow（durable execution state，actorStateStore 必需）分离；AgentStateConfig 是 schema bundle 注入机制（entry_factory/message_coercer 按 agent 类型选择状态模型）（configs.py:246；品类卡 §5）
6. **消息驱动三模式 Runner**：AgentRunner run（ad-hoc，返回 instance_id）/ subscribe（pub/sub TriggerAction 消费）/ serve（FastAPI HTTP：流式 SSE/NDJSON + HITL 审批路由 + pub/sub 路由）（workflow/runners/agent.py:206,871,911）

## 关键数字
- 代码：非测试 py 51,516 行；测试 160 文件 41,490 行（0.81x）
- DurableAgent = DurableExecutorMixin + AgentBase；仓库无 `Agent` 类（"Agent deprecated"为文档层事实）
- 依赖 `dapr[fastapi,workflow]>=1.19.0rc4`——durable 引擎本体在 Dapr runtime/sidecar，Python 侧为 SDK 绑定
- v1.1.0（PyPI 2026-10-30 实测）

## 与已有 Corpus 连接（durable execution 四路线）
| 路线 | 仓库 | 恢复语义 | Corpus |
|---|---|---|---|
| Workflow 步骤持久化 | Dapr Agents | 重放不重执行（事件溯源） | **本轮** |
| 步骤持久化（Java） | Restate | 日志重放 | 10-27（PR #56） |
| 状态表化 | DBOS | 数据库 checkpoint | 10-28（PR #57） |
| 并发挂起（Rust） | Infinity | 挂起恢复零占用 | 10-29（PR #58） |

## 审计声明
本考古以仓库实际代码为准（commit 1fd68b36 快照）；品类卡/官方文档仅作对照标注（"Agent deprecated"等文档层事实单独标记）；无 Docker 环境，未运行中断恢复 demo——运行行为证据来自代码语义与公开文档，标 S3-S4 级，不冒充 S5。

# Project Overview — dapr-agents（ARCH-2026-10-25-001）

**定位**: CNCF Dapr × NVIDIA 协作的分布式 Agent 运行时框架（Apache-2.0，Python ≥3.11），把 Dapr 成熟的 workflow 引擎（durable execution）+ actor + state store + pub/sub 搬到多 agent 系统——"agent 的每一步推理与工具调用都持久化，崩溃后从最后完成步骤恢复，不重放已完成的 LLM/工具调用"。

**版本**: v1.0.0（schemas/agent-metadata latest.json）· HEAD `8b4cc8d`（2026-10-22，chore(typing) #837）· 764 stars · pushed 2026-10-22

## 六条认知主线（详细证据见 02/03/04）

1. **durable execution 是"三件套"而非单点特性**：持久化点（workflow entry state）+ 幂等活动（call_llm 重放检测 / publish 缓存）+ 确定性 ID（approval uuid5）——三者缺一，崩溃恢复就会重放或重复
2. **HITL 审批默认 fail-closed**：超时 auto-deny、响应解析失败 auto-deny、无交互通道工具短路；只有显式"无限等待"配置（default_timeout_seconds=None）才挂起
3. **workflow 与 LLM 的确定性边界是精心维护的**：I/O 全部 activity 化（replay-friendly）、结构化输出在 mid-tool-loop 省略 tools（防模型回 tool call 而非 JSON）、stream_ctx 冻结 per-session（防 partial stream——历史 bug 修复）
4. **扩展点双轨**：hooks（决策型：Deny/Skip/Mutate/RequireApproval）+ lifecycle Protocol（拦截型：BEFORE_AGENT_INVOKE/BEFORE_TOOL_CALL，OSS-clean）——auth/OBO/observability 不进框架核心
5. **状态分层**：execution state（workflow entry）≠ session memory（DaprSessionStore manifest+chunk）≠ tool history（observability）——OpenAI 消息链约束（tool 必须紧跟 assistant+tool_calls）被显式处理（skip_messages 双模式）
6. **治理代码化 + schema 化**：AGENTS.md 提交门（ruff/flake8/mypy/pytest 全链）+ DCO 强制 + agent-metadata schema 版本化（v0.11.1→v1.0.0）+ schema-compat CI

## 工程规模
- 核心包 16 子包；DurableAgent 4246 行 / AgentBase 2217 行 / configs 943 行
- 测试 129 个 test_*.py + integration（API keys）+ ext（Drasi 门控）+ CI 8 workflows
- 32 个 examples + 15 个 quickstarts（03_durable_agent_http.py 中断恢复 demo 可复现）

## 跨项目连接
- **TeamMind**（用户多 Agent 运行时，Runtime 周头号锚点）：durable state/崩溃恢复/actor 并发设计基线
- **deepseek-harness**（validated）：approval 机制对照（dsh 权限预设 vs dapr HITL fail-closed）
- **MXC / strands-box**（10-23/10-24）：执行隔离（沙箱遏制）→ 本项目的分布式可靠执行（运行时层）——"隔离 + 可靠"是 Agent 基础设施两条腿
- **langgraph / letta-code / opencode**（已考古）：编排框架/记忆/agent 运行时的谱系对照

# Candidates — dapr-agents 未确认内容（Hypothesis / Cross-project pending）

> 与 KO 严格区分：以下全部是不能确认的内容。

## C-D01 · AgentExecutorBase 将取代标准循环（Hypothesis）
- **主张**: Claude executor（事件流模型）是未来统一执行模型，标准 call_llm→run_tool 循环逐步被 executor 路径替代
- **当前证据**: durable.py:694-698（executor 分支存在）；executors/claude.py（完整实现）；"The executor owns the tool/reasoning loop"
- **缺失证据**: 框架路线图/弃用声明；executor 覆盖所有 provider 的证据
- **验证路径**: 追踪 executor 分支使用率 + roadmap 文档

## C-D02 · HITL 无限等待（None）生产安全性（Hypothesis）
- **主张**: default_timeout_seconds=None（无限挂起）在多租户生产环境存在挂起风险（资源占用/审批僵尸）
- **当前证据**: configs.py:540（"None means wait indefinitely"）；无强制告警机制证据
- **缺失证据**: 挂起实例的回收/超时告警实现
- **验证路径**: 检查 workflow timeout 配置与实例回收机制

## C-D03 · Dapr 1.18 rc 依赖漂移（Hypothesis）
- **主张**: dapr>=1.18.0rc0,<1.19.0 绑定 rc 版本，sidecar 正式发布可能引入行为漂移
- **当前证据**: pyproject.toml（1.18.0rc0 pin）；bump-dapr-deps.yaml（CI 自动 bump）
- **缺失证据**: rc→GA 变更记录对照
- **验证路径**: bump-dapr-deps workflow 产物 + changelog

## C-D04 · MCP auto-discovery 权限模型（Hypothesis）
- **主张**: sidecar 暴露的 mcp_servers 无细粒度权限控制，任意 agent 可发现全部 MCP 工具
- **当前证据**: base.py:530-547（发现逻辑无过滤）；README（"Zero-config"）
- **缺失证据**: access scopes / 资源过滤的 agent 侧实现（README 提到 "Access scopes... platform teams" 属平台侧）
- **验证路径**: access scope 文档 + agent 侧资源过滤代码

## C-D05 · 跨项目假说：确定性与 fail-closed 是 Agent 基础设施两支柱（Cross-project Hypothesis）
- **主张**: dapr-agents 的"确定性三件套"（持久化点/幂等活动/确定性 ID）与 MXC/strands-box 的 fail-closed 链条是同一治理模式的两端：**前者保证"该完成的一定完成（且不重复）"，后者保证"该拒绝的一定拒绝"**——可靠性与遏制性是 Agent 基础设施的两个正交维度
- **当前证据**: dapr KO-D01（三件套）；MXC KO-M01（fail-closed 链条，10-24 run）；strands-box KO-B04（fail-closed 加载契约，10-23 run）
- **缺失证据**: 两支柱在同一系统共存的实例证据（dapr 无沙箱遏制；MXC 无 durable execution）
- **验证路径**: 需要同时具备二者的第三项目（如 deepseek-harness 的 permission-presets + session-telemetry 是否兼有）；**未验证前不得写成 Principle**

## C-D06 · orchestration 结构化输出在大型多 agent 网络的扩展性（Hypothesis）
- **主张**: IterablePlanStep/NextStep 结构化规划在 100+ agent 网络会遇 token/规划深度瓶颈
- **当前证据**: durable.py:1998-2183（plan 提取）；execution.max_iterations 有上限
- **缺失证据**: 大规模基准（examples 04-multi-agent-workflow-k8s 规模有限）
- **验证路径**: k8s 大规模压测 + 观测规划 token 消耗

## C-D07 · observability 覆盖缺口（Hypothesis）
- **主张**: OTel instrumentor 对 approval 事件/streaming 事件覆盖不完整（主要覆盖 llm/tool）
- **当前证据**: observability/instrumentor.py + wrappers/（目录存在）；无 approval 埋点直接证据
- **缺失证据**: 埋点清单与 approval/stream 关联证据
- **验证路径**: instrumentor 代码逐点核对 + demo-otel-k8s 追踪

## Candidates 统计
- 7 条（C-D01~C-D07）；全标 Hypothesis / Cross-project pending；无一条冒充已验证知识

## C-D08 · orchestration_workflow 主体活动序列深读（Hypothesis · Auditor O1）
- **主张**: orchestration_workflow 主体（1332-1550）的 plan→agent loop→finalize 细节可能揭示结构化规划的另一层约束
- **当前证据**: durable.py:1332-1360（plan_prompt → IterablePlanStep）；策略活动区 1868-1955
- **缺失证据**: 1332-1550 中 agent 循环内 call_llm 的 response_format 变化与 check_completion 语义
- **验证路径**: 逐函数读 orchestration_workflow 主体

## C-D09 · AgentRunner HTTP 路由契约（Hypothesis · Auditor O2）
- **主张**: register_routes/_build_pubsub_specs（runners/agent.py:687-728）定义的服务端点契约影响多实例部署
- **当前证据**: runners/agent.py:134（AgentRunner）；_HitlRespondBody:76
- **缺失证据**: 端点清单与多 replica 广播语义
- **验证路径**: 读 register_routes 全文 + quickstarts HTTP demo

## C-D10 · lifecycle DecisionDict 前瞻字段（Hypothesis · Auditor O3）
- **主张**: DecisionDict 中 forward-looking approver-authorization 字段预示未来权限模型
- **当前证据**: lifecycle.py:30-32（"plus a few forward-looking approver-authorization fields"）
- **缺失证据**: 字段清单与 dispatcher 实现（框架外）
- **验证路径**: 读 lifecycle.py 完整 DecisionDict + TypedDict 定义

## C-D11 · 内置工具注册表语义（Hypothesis · Auditor O4）
- **主张**: built-in tools（ask_user/advance_team，durable.py:408 注册）与 hooks 决策交互有未整理细节
- **当前证据**: durable.py:241-246（BuiltinTool.ASK_USER 映射）；ask_user 无流短路 835
- **缺失证据**: advance_team 语义与 broadcast 交互
- **验证路径**: 读 built-in tools 构造与 advance_team 测试

## Candidates 统计（更新）
- 11 条（C-D01~C-D11）；全标 Hypothesis / Cross-project pending；无一条冒充已验证知识

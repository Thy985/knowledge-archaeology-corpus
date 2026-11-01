# Validation & Evidence — ARCH-2026-10-25-001（dapr-agents）

> 六 Auditor 视角 + Blind Reconstruction 原则。

## 1. Truth Auditor（这句话是真的吗）
| 断言 | 验证 | 结果 |
|---|---|---|
| agent_workflow 三分支（orchestrator/executor/标准循环） | durable.py:660-716 实读 | ✅ |
| call_llm 重放检测（_call_llm_message_id/_find_recorded_message） | durable.py:2300-2316 实读 | ✅ |
| publish_approval_request replay 缓存 | durable.py:1275-1281 实读 | ✅ |
| approval timeout 竞速 auto-deny | durable.py:1215-1260（when_any/timer） | ✅ |
| approval_request_id 确定性 UUID（uuid5） | durable.py:1228-1230 实读 | ✅ |
| ask_user 无流短路 | durable.py:817-836 实读 | ✅ |
| save_tool_results skip_messages 双模式 | durable.py:3054-3110 docstring | ✅ |
| 结构化输出 mid-loop 省略 tools | durable.py:2352-2378 实读 | ✅ |
| DaprSessionStore manifest+chunk | sstore:171-275 实读 | ✅ |
| MCP auto-discovery（sidecar mcp_servers） | base.py:530-547 实读 | ✅ |
| AgentApprovalConfig 双通道 + 默认 300s | configs.py:508-540 实读 | ✅ |
| AGENTS.md 提交门 + DCO | AGENTS.md 实读 | ✅ |
**Truth: 12/12 通过**

## 2. Coverage Auditor（还有什么重要东西没发现）
- [x] 主执行模型（agent_workflow/call_llm/run_tool/save）→ EK-D01~D03/D08~D10
- [x] 审批/交互（approval/ask_user/hooks）→ EK-D04~D06
- [x] 工具系统（WorkflowContextInjectedTool/agent-as-tool/MCP）→ EK-D07/D14
- [x] 状态/存储（entry state/session store/双 store）→ EK-D12/D17
- [x] 扩展点（executor/lifecycle）→ EK-D13/D15
- [x] 配置/版本（AgentMetadata schema/schema-compat）→ EK-D16/D20
- [x] 恢复（start 幂等/审批恢复）→ EK-D19
- [x] 编排（orchestration 活动序列）→ EK-D18
- [ ] **缺口 1**: streaming 实现细节（streaming/ 包、chunk 协议 AgentStreamChunk）未逐函数深读
- [ ] **缺口 2**: llm/ 各 provider 适配内部差异未逐 provider 核对（统一接口 vs 特性差异）
**Coverage: 声明缺口 2 处，不弱化全绿**

## 3. Flow Auditor（Flow Atlas 是否对应代码）
- 三分支 Edge：durable.py:660-716 ✅
- 审批 Edge：_await_approval 全文 ✅
- session store Edge：append/load 实读 ✅
- MCP Edge：base.py 发现逻辑 ✅
- 治理 Edge：AGENTS.md ✅
**Flow: 5/5 Edge 可回溯**

## 4. Abstraction Auditor（L3/L4 是否过度升维）
- KO-D01（三件套 L4）：从 D02/D05/D19 机制聚类——解释范围确实扩大（任何 durable agent 系统），但仅 dapr 单项目 → **L4 + Cross-project validation pending**
- KO-D02（审批 fail-closed L3）：MXC/strands-box 跨项目对照支持 → Pattern（不升 Principle）
- KO-D05（状态分层 L4）：letta-code 记忆分层对照存在但未深考古 → L4 + pending
- KO-D06（显式扩展成本 L4）：MXC backend registry 对照 → L4 + pending
**Abstraction: 无过度升维；L4 全部 pending 标记**

## 5. Counterexample Hunter（反例预算）
| KO | 定向反例攻击 | 结果 |
|---|---|---|
| KO-D01 | approval None 无限挂起阻塞 workflow——"可靠"不含"可终结" | **反例成立**（写入 KO） |
| KO-D02 | default_timeout_seconds=None 显式放弃 fail-closed | **反例成立**（写入 KO：默认态 vs 显式配置） |
| KO-D03 | executor 分支绕过标准循环的消息链纪律？ | 反例不成立（executor 自己维护事件流，非破坏） |
| KO-D04 | lifecycle Protocol 有 plugin 依赖就会破坏 OSS-clean？ | 反例部分成立（注释声明 OSS-clean，实现未深读——标 NEEDS_HUMAN_REVIEW？不，留 Candidates C-D07 邻域） |
| KO-D06 | MCP "zero-config" 恰恰是"不显式"——反例？ | **反例成立**（zero-config 是例外：发现自动而非显式声明；但工具接入后执行仍走注册路径，部分成立） |
**反例预算: 3 成立 / 1 部分 / 1 驳回；全部保留**

## 6. Epistemic Auditor（认知状态诚实）
- Fact: 项目地图全部可回溯（文件:行）
- EK（L1/L2）: 20 条 S3+；无"我认为"
- Pattern（L3）: KO-D02/D03/D04（MXC 对照跨项目支持 KO-D02）
- Cognitive Model（L4）: KO-D01/D05/D06 全标 pending
- Candidates: 7 条全标 Hypothesis；C-D05 明确"未验证不得成 Principle"
- **Contradictions 保留**: 无文档-实现重大矛盾（README 特性与代码一致）
- **Counterexamples 保留**: 3 成立 + 1 部分，全部写入 KO/Candidates

## 质量指标（初版，pending Auditor 复核）
- EK 20（links 100%，avg 出边 ~2.0，游离 0）；KO 6（R1×2 R2×1 R3×1 R4×2）；Candidates 7；Flow 7 类；evidence ~55 处
- Truth 12/12；Coverage 缺口 2 声明；Flow 5/5；Abstraction 无过升维；反例 3 成立 + 1 部分保留

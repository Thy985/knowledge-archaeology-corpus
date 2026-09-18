# Independent Validation Report — OpenClaw（ARCH-2026-09-17-001）

> Auditor 独立盲重建：不把 Archaeology Result 当事实来源，独立重读仓库建立 Independent Findings 后对比。审计路径：src/agents/subagents/swarm/、src/agents/tools/（sessions-*）、src/acp/approval-classifier.ts、src/agents/agent-tools.policy.ts、docs/gateway/config-tools/sessions-and-subagents.md、docs/gateway/security/hardened-baseline.md、docs/gateway/security/audit-checks.md。

## 0. 判定统计
| 判定 | 数量 | 对象 |
|---|---|---|
| CONFIRMED | 26 | EK-01/02/03/05/06/07/08/09/10/11/12/13/14/15/16/17/18/19/20/21/22/23/24/25/27/28（核心 26 条） |
| PARTIALLY_CONFIRMED | 7 | EK-04（子 agent 例外未提）、EK-26（signal 检测机制确认，severity 阈值部分）、EK-30（测试确认，但未全量跑）、EK-34/36/37/38 |
| DOWNGRADED | 1 | KO-01（"默认权限面漂移"升 L4 前需限定"主 agent 之间"） |
| OVER_GENERALIZED | 2 | KO-01（"默认开放"未限定子 agent deny-wins 例外）、KO-04（"收紧缝隙"未含 allow 回退语义） |
| MISSING | 4 | M-01 SUBAGENT_TOOL_DENY_ALWAYS；M-02 ACP approval-classifier 控制面分类；M-03 hardened-baseline 文档；M-04 删除 agent 后 allow 回退 allow-all |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 2 | N-01 全量 15,643 测试未运行；N-02 "协作便利优先"是否为设计意图（需 maintainer 确认） |

## 1. 原 Archaeology 最重要的 3 个成功
1. **S-1 默认值漂移 = 权限变更的核心叙事**：准确捕捉 `visibility agent→all` + `agentToAgent false→true` 的省略值漂移，并绑定"升级即扩权"（docs/releases/2026.9.2.md:3245），这是本次考古最高价值的发现。
2. **S-2 信任模型声明与实现现实的张力**：SECURITY.md "usability features, not security boundaries" 与完整权限收紧体系并存，KO-02 的"信任边界位置决定默认值"抽象正确。
3. **S-3 Flow Atlas 的 Edge 真实可回溯**：七类流全部锚定真实 symbol/file（session-visibility.ts:148-276、audit-extra.summary.ts:180-251、swarm-scheduler.ts:30-65），Auditor 重读确认主要 Edge 存在。

## 2. 最重要的 3 个错误
1. **E-1（MISSING）子 agent 工具 deny-wins 例外未覆盖**：`src/agents/agent-tools.policy.ts:26-42` 定义 `SUBAGENT_TOOL_DENY_ALWAYS`——子 agent 无论深度都禁止 `sessions_send`/`message`/`gateway`/`agents_list`/`openclaw` 等 12 工具，leaf 还额外禁止 sessions_spawn/list/history/search。考古 KO-01/EK-04 说"跨 agent 会话访问默认开放"，但**未限定这是主 agent 之间的默认**——子 agent 层是 deny-wins。原结果把"开放"说得过宽。
2. **E-2（MISSING）ACP approval-classifier 控制面分类未覆盖**：`src/acp/approval-classifier.ts:26-32` 把 `sessions_spawn`/`sessions_send`/`session_status` 归入 `CONTROL_PLANE_TOOL_IDS`（autoApprove=false，prompt-required）——即 ACP 通道下 session 工具是控制面级审批。考古 EK-17 只提 gateway/cron owner-only，遗漏 ACP 通道的 session 工具分类。
3. **E-3（MISSING）删除 agent 后 allow 回退 allow-all 未覆盖**：docs/gateway/config-tools/sessions-and-subagents.md:29 "Deleting an agent … prunes its id from allow; if that empties the list, the policy falls back to allow-all"——**配置从"受限"变"空"会静默回退全允许**，这是 EK-28 语义的补充缺口（考古只写了省略/空=允许、配置空=拒绝，未写删除导致的回退）。

## 3. 关键遗漏（除 E-1/2/3 外）
- **M-4**：docs/gateway/security/hardened-baseline.md 提供官方加固基线（sessions.visibility:"agent" + agentToAgent.enabled:false + deny sessions_spawn/sessions_send + exec security:deny + elevated:false）——考古 03 KO-03 remediation 未引用此现成基线文档。
- **M-5**：sandbox clamp 在 docs 中的完整语义（sessions-and-subagents.md:58 "access stays limited to spawned sessions even if the caller is main"）确认 EK-24，但考古未引用此文档锚点。

## 4. 错误升维检查
- KO-01 L4 "默认值即策略"：**需限定**——默认值漂移影响的是**主 agent 之间**的会话访问；子 agent 层 deny-wins 不适用。判定 DOWNGRADED（降为带限定条件的 Pattern）。
- KO-04 "收紧完备性由剩余可达集定义"：**补充**——需加入"配置变更（agent 删除）导致 allow 回退 allow-all"这一缝隙。判定 OVER_GENERALIZED（补限定）。
- 其余 KO 升维站得住（L3/L4 均未宣称跨项目 Law）。

## 5. 事实错误检查
- 未发现实质事实错误（版本/路径/默认值/checkId 均准确）。
- 一处精确性修正：EK-28 "空白条目拒绝"正确，但需补"删除 agent 致空列表→回退 allow-all"（见 E-3）。

## 6. Flow 错误检查
- Control/State/Data/Evidence/Authority/Memory/Policy 七类流的主要 Edge 经重读确认真实。
- **补充 Flow Edge**：Authority Flow 缺 ACP 通道分支——`sessions_send` 经 ACP 时归 control_plane（autoApprove=false）→ prompt-required（approval-classifier.ts:238-240）。
- **补充 Flow Edge**：子 agent 工具面在 spawn 时被 SUBAGENT_TOOL_DENY_ALWAYS 收缩（agent-tools.policy.ts:26-42）→ 跨 agent 发送被 deny（sessions-send-tool.ts:692 实际调用点确认 isAllowed 存在）。

## 7. 新 Benchmark / Regression Case
- **B-01（回归）**：SUBAGENT_TOOL_DENY_ALWAYS 断言——子 agent 永不允许 sessions_send/message/gateway，leaf 额外禁 sessions_spawn/list/history/search。可作为 knowledge-archaeology-skill 的"子 agent 权限例外"fixture。
- **B-02（回归）**：删除 agent 后 allow 空→回退 allow-all 语义（docs 明确，代码 createAgentToAgentPolicy 空列表返回 true 一致）。
- **B-03（基准）**：ACP 通道 sessions_send 分类为 control_plane（autoApprove=false）——跨通道权限差异基准点。
- **B-04（基准）**：audit severity 漂移（info→warn 当 multi-user signal）的判定边界（audit-extra.summary.ts:241）。
- **B-05（回归）**：invalid visibility 解析到 all（fail-open 方向）已有测试覆盖，可升级为 skill 反例预算 fixture。

## 8. Auditor 独立结论
- 原考古对"2026.9.2 权限模型变更"的主叙事正确且高价值，但存在 **4 个系统性 MISSING**（子 agent deny-wins、ACP 控制面分类、hardened baseline、allow 回退语义），导致 KO-01/KO-04 存在**过度泛化**（"默认开放"未限定主 agent 范围）。
- 需 Reconciliation：补充 EK-39/40/41（SUBAGENT deny-wins、ACP 分类、allow 回退）、修正 EK-04/KO-01/KO-04 限定词、Flow Atlas 补 2 条 Edge、新增 B-01~05 benchmark case。
- 不需要人审的严重事实错误或安全漏洞指控；N-01/N-02 列为需要人工确认的开放项。

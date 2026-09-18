# 05 Candidates — OpenClaw（未验证假设 / 跨项目假说 / 待验证模式）

> 不能确认的内容留在 Candidate，不冒充已验证知识。全部标 Hypothesis 状态。

## C-01 跨项目假说：默认权限面漂移是协作型工具的普遍演化路径
- **Hypothesis**：协作型 agent 平台（多 agent/多用户）在演进中普遍会"默认扩大协作面 + 用审计提示补偿"，与多租户系统"默认收紧"相反。
- **当前证据**：OpenClaw 2026.9.2 单项目实例（visibility agent→all、agentToAgent false→true）。
- **缺失证据**：其他协作型 agent 平台（如多用户 MCP gateway、团队部署 agent）的默认值演化史。
- **验证路径**：对比 2-3 个协作型 agent 系统的权限默认值 changelog。

## C-02 待验证模式：审计 severity=info/warn 与"operator 决策责任"的对应关系
- **Hypothesis**：审计项 severity 隐含决策责任归属（info/warn→operator 自决；critical→系统拦截）。
- **当前证据**：OpenClaw 跨 agent 会话访问报 info/warn 不 fail；audit-checks.md 有 critical 级（fs.config.perms_writable）。
- **缺失证据**：是否所有 info/warn 项都"允许 operator 无视"？critical 是否全部"系统级拦截"？未系统核验全部 audit 项。
- **验证路径**：遍历 audit-checks.md 全部 checkId 的 severity 分布与 fail 行为。

## C-03 范围不确定结论：swarm 默认开启对多 agent 共享 Gateway 的 DoS 面
- **Hypothesis**：swarm 默认开启（maxConcurrent=8）在多 agent 共享 Gateway 下可被任一 agent 消耗并发预算。
- **当前证据**：swarm-config.ts 默认 enabled=true；容量上限有界。
- **缺失证据**：实际并发耗尽场景（多 agent 同时 swarm）的调度测试；容量 lane 是 per-group 还是 per-gateway。
- **验证路径**：读 swarm-scheduler.ts lane 作用域 + 并发压力测试。

## C-04 潜在原则：解析方向（fail-open/fail-closed）是权限模型的哲学投影
- **Hypothesis**：权限系统的"歧义解析方向"与其默认值哲学一致（OpenClaw 全部向允许倾斜）。
- **当前证据**：invalid visibility→all、空 allow→全允许、Code Mode fail-closed 例外。
- **缺失证据**：这是否是普遍规律（需要跨项目验证）。
- **验证路径**：对比 09-13/14/16 轮四个防火墙项目的解析方向。

## C-05 未解决矛盾：SECURITY.md"非安全边界"声明 vs 权限模式/审计的"边界感"
- **Observation**：SECURITY.md 说 session visibility 不是安全边界，但 permission-modes.md/agentToAgent/audit 提供了完整收紧体系——语义上接近"安全控制"。
- **Hypothesis**：这是"产品定位声明"与"工程实现事实"的张力：实现层提供边界感，声明层否认边界性。
- **影响**：operator 可能误读"非安全边界"为"无需收紧"，或误读收紧体系为"可对抗恶意 agent"。
- **验证路径**：读 2026.9.2 PR #136755 的原始讨论与 maintainer 意图。

## C-06 待验证：allow 空数组语义漂移（未设置 vs 配置后空）的 operator 认知风险
- **Hypothesis**：`allow: []`（显式空数组）在 operator 心智中是"拒绝所有"，实际语义是"未设置→全允许"——高风险认知偏差。
- **当前证据**：session-visibility.ts:244-276 注释 "Omitted or empty counts as unset: every agent pair is allowed by default; blank entries deny"；审计测试覆盖 "explicit all visibility and empty allow list" 报 1 条。
- **缺失证据**：真实 operator 配置事故案例。
- **验证路径**：GitHub issues 检索 allow:[] 误配置案例。

## C-07 范围不确定：原生 harness 权限旁路的实际攻击面
- **Hypothesis**：原生 harness（codex 等）自有工具面不受 OpenClaw-managed 规则覆盖，构成权限模型旁路。
- **当前证据**：permission-modes.md "Native harnesses can retain their own tool surface under their own permission controls"。
- **缺失证据**：具体 harness 的工具面与 OpenClaw 策略的交互测试。
- **验证路径**：读 codex-harness-runtime 插件文档 + harness 权限测试。

## C-08 跨项目假说：trust boundary 位置决定权限默认值（KO-02 的推广）
- **Hypothesis**：任何权限系统，其默认值的开放/收紧程度由"信任边界画在系统内还是系统外"决定。
- **当前证据**：OpenClaw（边界在外→默认开放）+ AGT/Guardian（边界在内→fail-closed）两个对照点。
- **缺失证据**：更多对照样本（至少 3 个系统）。
- **验证路径**：对 corpus 中 13 个项目逐一标注"信任边界位置"与"默认权限方向"。

## C-09 待验证：升级迁移窗口的默认值漂移普遍性
- **Hypothesis**：发布说明中 "Default: true for one migration window"（schema.help.runtime.ts:60）表明迁移窗口期默认值漂移是 OpenClaw 的常规机制，不止 2026.9.2。
- **当前证据**：同模式案例存在（Browser Relay Authentication v2）。
- **缺失证据**：迁移窗口机制的完整清单与安全影响评估。
- **验证路径**：检索 CHANGELOG 中所有 "migration window" 案例。

## C-10 未解决：incognito 会话唯一隐藏例外 vs 其他隐藏机制
- **Observation**：incognito sessions 对所有工具隐藏（EK-36）；memory_search 是 agent-scoped（EK-23）；sessions_search 是 permission-scoped。
- **Hypothesis**：OpenClaw 存在多套正交的"隐藏/可见"语义，operator 难以完整建模。
- **缺失证据**：这三套语义的交叉矩阵文档/测试。
- **验证路径**：读 incognito 实现 + 交叉测试。

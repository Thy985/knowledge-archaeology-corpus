# 03 Knowledge Layer — OpenClaw（Generalized KO，aggregation_rule R1-R4）

> 每个 KO 声明 aggregation_rule（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）+ 簇内 EK + 边类型。未跨项目验证的升维一律标注 Cross-project validation pending。

## KO-01 默认权限面漂移 = 最隐蔽的权限变更（R2 因果链簇；限定：主 agent 之间）
- **aggregation_rule**：R2 —— EK-01 → EK-27 → EK-37 → EK-02 → EK-03（causal 链：默认值变更 → 升级迁移生效 → 升级提示 → 审计暴露 → 收紧路径）+ EK-39 constraint（**限定：仅主 agent 之间**；子 agent 层 deny-wins）
- **L3 Pattern**：协作型工具在"降低协作摩擦"与"扩大默认权限"之间选择前者时，把补偿机制放在"升级说明 + 审计提示 + 显式收紧路径"三个暴露层，而不是默认 fail-closed。**该模式限定于主 agent 之间**——子 agent/子进程层应保持 deny-wins（OpenClaw SUBAGENT_TOOL_DENY_ALWAYS 即此例）。
- **L4 Cognitive Model**：**默认值即策略**——配置省略值的变化是策略变化，其影响面与显式配置变更等同，但可见性低一个数量级；**且默认开放的语义范围必须按执行体层级限定**（主 agent 开放 ≠ 子 agent 开放）。
- **L5 Methodology**：权限默认值变更必须：(1) 在发布说明顶部给升级提示；(2) 提供可审计检查项；(3) 提供逐级收紧路径；(4) 在升级迁移窗口内生效并明示；(5) **逐层声明默认开放的作用域**（主 agent/子 agent/子进程分别标注）。
- **Evidence**：docs/releases/2026.9.2.md:11,3245,3247；audit-extra.summary.ts:241-256；agent-tools.policy.ts:26-42
- **status**：Validated Pattern（OpenClaw 单项目强证据；Cross-project validation pending）
- **value**：A

## KO-02 默认开放 + 显式收紧 ≠ fail-open 漏洞，但需独立信任边界兜底（R3 不变量簇）
- **aggregation_rule**：R3 —— EK-04/EK-07/EK-16/EK-24 汇聚到同一不变量：**单 Gateway 不是安全边界**；session visibility/agentToAgent 是 usability 控制，真实隔离必须独立 Gateway。
- **L3 Pattern**：local-first 单 operator 工具用"默认开放"换协作便利，用"信任模型声明 + 独立部署边界"兜底安全——与多租户系统的 fail-closed 是同一问题的两种解法。
- **L4 Cognitive Model**：**安全边界的位置决定权限默认值**——把信任边界画在"单 Gateway 外部"（独立部署），则内部默认开放不构成漏洞；把边界画在"agent 之间"（多租户），则必须 fail-closed。
- **L5 Methodology**：设计权限默认值前先回答"信任边界画在哪"；若边界在系统外部，内部默认开放 + 审计提示即可；若边界在系统内部，默认必须收紧。
- **Evidence**：SECURITY.md "Shared Agents"；docs/gateway/security/access-control.md:61；session-visibility.ts:148-276
- **status**：Validated Pattern（与 AGT 轮 fail-closed 形成对照对；Cross-project validation pending）
- **value**：A

## KO-03 审计"暴露层"与"门禁"是两种治理哲学（R4 主题簇）
- **aggregation_rule**：R4 —— EK-02/EK-26/EK-30 围绕"安全审计角色"主题：审计报 info/warn 不 fail、trust-boundary signal 升 warn、表驱动测试锁定行为。
- **L3 Pattern**：安全审计的两种定位——**暴露层**（提示 operator 自决，fail-open）vs **门禁**（CI/质量门拦截，fail-closed）；OpenClaw 选前者，AGT/Guardian 类选后者。
- **L4 Cognitive Model**：**审计严重度的语义 = 谁承担决策责任**——info/warn 表示"operator 决策"，critical/fail 表示"系统决策"。
- **L5 Methodology**：为安全审计定义严重度时，显式声明每个 checkId 的决策责任归属；不能把"应该 operator 决定"的项伪装成"系统已拦截"。
- **Evidence**：audit-extra.summary.ts:180-251；audit-cross-agent-session-access.test.ts；docs/gateway/security/audit-checks.md（severity 表含 critical）
- **status**：Validated Pattern（跨项目对照：AGT/Guardian fail-closed audit）
- **value**：A

## KO-04 收紧路径的"缝隙"决定安全语义完整性（R2 因果链簇）
- **aggregation_rule**：R2 —— EK-03 → EK-24 → EK-05 → EK-41（收紧 → sandbox clamp 局限 → visibility 语义 → **allow 回退 allow-all**）：agentToAgent=false 后 requester-owned native/ACP child 仍可达；sandbox 不隐藏 transcript；tree 仍含同 agent 全会话；**删除 agent 可致 allow 空→回退 allow-all（配置从受限静默变全允许）**。
- **L3 Pattern**：权限收紧功能必须列出"收紧后仍可达的边界"**及"配置变更可使收紧静默失效的路径"**——否则 operator 以为已隔离，实际留缝。
- **L4 Cognitive Model**：**权限收紧的完备性由剩余可达集与回退路径共同定义**，不由"已关闭的开关数"定义。
- **L5 Methodology**：任何收紧开关的文档必须附带"此开关不覆盖的路径"清单 + "哪些配置操作会重置该开关"警示；审计输出应显示剩余可达者（OpenClaw audit 的 reachers/nonReachers 即此模式）。
- **Evidence**：schema.help.runtime.ts:117-121；docs/releases/2026.9.2.md:3249；audit-extra.summary.ts:197-235；docs/gateway/config-tools/sessions-and-subagents.md:29
- **status**：Validated Pattern（OpenClaw 单项目；Cross-project validation pending）
- **value**：A

## KO-05 权限默认值方向的 fail-open 解析是系统性选择（R3 不变量簇）
- **aggregation_rule**：R3 —— EK-25/EK-28/EK-05/EK-36 汇聚到不变量：**配置解析与权限评估在歧义时向"允许"倾斜**（invalid visibility→all、空 allow→全允许、incognito 唯一例外）。
- **L3 Pattern**：配置解析的 fail-open/fail-closed 方向与权限模型的默认开放方向一致——OpenClaw 全部向"允许"倾斜，仅在 blank-configured-allow 一处向"拒绝"倾斜。
- **L4 Cognitive Model**：**解析方向的 fail-open 是权限模型的默认值哲学的投影**——解析器与策略层共享同一风险偏好。
- **L5 Methodology**：审查权限系统时，把"歧义解析方向"列为独立检查维度（invalid 枚举值、空列表、缺失字段各自如何解析），不与"显式配置"混为一谈。
- **Evidence**：session-visibility.ts:148-161,244-276；audit-cross-agent-session-access.test.ts "invalid visibility" 用例
- **status**：Hypothesis → Validated Pattern（OpenClaw 内证据闭合，跨项目验证中）
- **value**：A

## KO-06 容量治理与权限治理是两套正交机制（R4 主题簇）
- **aggregation_rule**：R4 —— EK-08/EK-09/EK-10/EK-32/EK-34 围绕"运行时治理"主题：swarm 容量有界（8/50/200/600s）、lane 队列、collector 状态机、worker 准入——都与权限模型正交。
- **L3 Pattern**：并发编排（swarm/worker）与权限模型（visibility/agentToAgent）分离设计：容量治理防资源耗尽，权限治理防越权，两者可独立配置。
- **L4 Cognitive Model**：**失控的维度至少有两个：权限（谁）与容量（多少）**——编排系统必须分别治理。
- **L5 Methodology**：为并发子系统设计时，容量上限必须有硬编码默认值 + 配置化上限（OpenClaw 用 readBoundedPositiveInteger 双向约束）。
- **Evidence**：swarm-config.ts:11-34；swarm-scheduler.ts:30-50；docs/agent-runtime-architecture.md "Compute workers"
- **status**：Pattern（中等证据）
- **value**：B

## KO-07 控制面工具与执行面工具分离（owner-only 控制面）（R1 机制簇）
- **aggregation_rule**：R1 —— EK-16/EK-17/EK-19 共享机制边（控制面 vs 执行面分离），跨 gateway/approval/control-plane 子系统出现。
- **L3 Pattern**：控制面工具（改配置/创建常驻任务/重启）与执行面工具（发消息/读文件）分离，控制面 owner-only 或需审批。
- **L4 Cognitive Model**：**影响持续性的动作（常驻、配置、重启）比影响即时性的动作需要更高权威**——常驻任务在会话结束后继续存在。
- **L5 Methodology**：识别"会话结束后仍生效"的工具并提升其权威要求（owner-only 或审批）。
- **Evidence**：docs/gateway/security/tool-permissions.md；src/gateway/approval-channel-custody.ts
- **status**：Validated Pattern（跨项目普遍；OpenClaw 实例证据闭合）
- **value**：A

## KO-08 升级即安全审计触发点（R2 因果链簇）
- **aggregation_rule**：R2 —— EK-01 → EK-27 → EK-37 → EK-02（默认变更 → 迁移生效 → 升级提示 → 审计检查项）：权限默认值变更天然绑定"升级后必须跑审计"的动作。
- **L3 Pattern**：发布权限默认值变更时，同步发布对应安全审计检查项，让升级动作自带验证闭环。
- **L4 Cognitive Model**：**升级窗口是安全暴露的高危期**——默认值漂移在升级瞬间生效，早于 operator 阅读文档。
- **L5 Methodology**：升级 SOP 中强制"权限敏感版本升级后先跑 security audit 再投入使用"。
- **Evidence**：docs/releases/2026.9.2.md:11,3251（"Run `openclaw security audit` to identify agents…"）
- **status**：Validated Pattern（OpenClaw 单项目；Cross-project validation pending）
- **value**：B

---
## KO 统计
- KO 总数：8（L3 8 个；L4 8 个；L5 8 个——每个 KO 含完整 L3-L5 链）
- 聚合规则分布：R1×1、R2×3、R3×2、R4×2
- 无"同子系统=聚合理由"的假聚合
- 全部 KO 可回溯 EK 与 Evidence

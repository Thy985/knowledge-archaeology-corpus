# 03 — Knowledge Layer（Generalized Knowledge，窄尖顶）

> L3 Pattern / L4 Cognitive Model / L5 Methodology。每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证。8 个 Core KO。

## KO-01 [L3] 分层防御：粗粒度前置门 + 细粒度授权（Ring → Policy）
- **aggregation_rule**：R2 因果链簇（EK-02 → EK-20 → EK-03 → EK-05）
- **内容**：执行路径先过粗粒度快速门（执行环/沙箱），再过细粒度授权（策略引擎）；被粗门拒绝的请求不消耗细粒度决策资源。与 AI Protector 的双门（身份→策略）同构，但 AGT 把"运行时风险"门与"授权"门分成两个独立机制。
- **解释范围扩大**：从 AGT 到任何分层安全系统（WAF→App 授权、内核→用户态），同类对照 AI Protector（proxy gate + agent gate）
- **epistemic**：Validated Pattern（AGT + AI Protector 双项目）

## KO-02 [L4] 治理决策确定性分层：核心授权确定性，非确定性只有"阻止权"
- **aggregation_rule**：R3 不变量簇（EK-33 + EK-13 + EK-05 + EK-04 + EK-37）
- **内容**：核心授权决策必须确定性（同输入同输出）、可证明（fail-closed）、可解释（BOM）——**"可被模型影响的批准不是批准，是建议"**。但修正早期绝对表述：非确定性层（advisory）存在且被允许，前提是**只有阻止权（block/flag），无放行权（不能 allow），失败默认 allow 且标 deterministic:false**——非确定性被结构性隔离在"拒绝侧"。
- **解释范围扩大**：从 AGT 的确定性策略引擎 + 受限 advisory → 任何 Agent 治理系统应回答两个问题："批准路径里有没有模型？"（应无）"拒绝路径能否被模型附加？"（可，但不能反过来）
- **epistemic**：Principle（AGT 强证据 + AI Protector 对照 + 独立验证修正）

## KO-03 [L4] 信任必须是可衰减的：委托不放大权限（Trust Ceiling）
- **aggregation_rule**：R3 不变量簇（EK-12 + EK-11 + EK-14）
- **内容**：子代理权限 ≤ 父代理权限（单调收敛）——**信任沿委托链只能递减**。这是对"agent 复制自身权限"问题的结构性回答，类比 capability 系统"不能授予超过自己持有的能力"。
- **解释范围扩大**：从 AGT 的 trust ceiling → 任何多 agent 委托系统；与 Aigis（session 隔离）形成"隔离 vs 收敛"两种策略
- **epistemic**：Principle（AGT 强证据 + capability 安全先例）

## KO-04 [L3] 审计可证明性三层：防篡改 → 可重构 → 可互操作
- **aggregation_rule**：R1 机制簇（EK-08 + EK-09 + EK-25 + EK-26，跨 audit/decision_bom/trace_sink 三模块同机制族）
- **内容**：Merkle 链防篡改（改/删/插可检测）+ Decision BOM 可重构（非侵入事后重建决策上下文）+ TRACE 信任记录（互操作格式）。审计不是"记日志"，是"可证明当时发生了什么"。
- **解释范围扩大**：从 AGT 审计 → 任何合规敏感 agent 系统；对照 Guardian（审计链 EK-06）——AGT 多了决策上下文重构层
- **epistemic**：Validated Pattern（AGT + Guardian 双项目）

## KO-05 [L4] 治理失败模式必须可见：fail-closed 优于 fail-open
- **aggregation_rule**：R3 不变量簇（EK-05 + EK-22b + EK-28）
- **内容**：fail-open 的错误是静默的（策略被绕过不报错），fail-closed 的错误是可见的（误拒立即暴露）——**可见性驱动修正，静默性滋养漏洞**。AGT 连"规则没写对"（DSL 静默不匹配）都要求显式测试。
- **解释范围扩大**：从策略引擎 → 任何安全控制系统（AV 误报 vs 漏报、防火墙默认 deny）
- **epistemic**：Principle（AGT 强证据 + 安全工程通例）

## KO-06 [L4] 治理栈从 OS 借力是 AGT 的设计选择（参考实现，非品类共性）
- **aggregation_rule**：R4 主题簇（EK-18 + EK-19 + EK-21 + EK-30 + EK-31，执行环/MCP 网关/供应链/威胁模型同主题）
- **内容**：AGT 是"Agent 的 Linux"——执行环（OS ring 特权级）、SRE（熔断/混沌/SLO）、供应链（SLSA/SBOM）、威胁建模（STRIDE 式）——**成熟的系统软件治理词汇表可直接映射到 agent 运行时**。修正：Guardian/AI Protector 未采用 OS 词汇表（sidecar/proxy 单层），这是 AGT 的全栈参考实现选择，品类内孤例。
- **解释范围扩大**：从 AGT 全栈 → Agent 平台可借鉴 OS 治理词汇表；但"全栈 OS 移植"是重量级选项，不是 agent 治理必然
- **epistemic**：Cognitive Model（AGT 强证据 + 系统软件通例 + 品类对照限定）

## KO-07 [L3] 策略即代码需要"逃生舱 + 约束"：DSL + 通用引擎 + 显式边界
- **aggregation_rule**：R4 主题簇（EK-03 + EK-22 + EK-22b + EK-23，策略表达三机制同主题）
- **内容**：YAML DSL（常用）+ Rego/Cedar（逃生舱）+ extends（继承）——但每个机制都有显式边界（DSL 无 field-vs-field、Rego 输入包装形状、extends additive-only 防削弱）。**策略表达能力越强，静默错误空间越大，必须配显式测试**。
- **解释范围扩大**：从 AGT 策略语言 → 任何策略 DSL 设计（Kubernetes admission webhook、IAM policy）
- **epistemic**：Pattern（AGT 单项目 + 通例，Cross-project validation pending）

## KO-08 [L5] 方法论：Agent 治理的验收标准清单
- **aggregation_rule**：R2 因果链簇（KO-01→KO-02→KO-03→KO-04→KO-05 全链）
- **内容**（可操作准则）：
  1. 决策路径确定性？有没有 LLM 在路径内？（KO-02）
  2. 委托是否放大权限？信任沿链单调收敛？（KO-03）
  3. 失败模式可见？fail-open 是否存在？（KO-05）
  4. 审计能否证明"当时发生了什么"？（KO-04）
  5. 执行风险是否分环/分层？（KO-01）
- **解释范围扩大**：从 AGT 考古 → 任何 agent 治理系统评估；本项目可直接作为 Benchmark case 用于 knowledge-archaeology-skill 的 agent-governance 品类
- **epistemic**：Methodology（AGT 推导 + 前三项目对照支撑）

---

## KO 聚合自检
- 8 个 KO 全部声明 aggregation_rule（R1-R4 均有）✓
- 全部可回溯 EK（无悬空升维）✓
- 解释范围扩大论证每 KO 附"对照"✓
- L3: KO-01/04/07；L4: KO-02/03/05/06；L5: KO-08——分布合理，未把全部 EK 升维 ✓
- **Reconciliation 修正**：KO-02 降级（确定性分层 + advisory 阻止权）；KO-06 限定（OS 词汇表是 AGT 参考实现选择）✓

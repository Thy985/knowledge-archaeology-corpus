# 03 Knowledge Layer — Generalized KO（L3/L4/L5）

> 7 个 KO，全部声明 aggregation_rule（R1-R4），簇可回溯至 EK/Evidence。Epistemic 状态诚实标注。

## KO-01 · Skill 执行层是独立安全域（行为抽象层）
- **knowledge_layer**: L4 Cognitive Model
- **aggregation_rule**: **R4 主题簇**（EK-01/EK-04/EK-12/EK-26/EK-27，机制互补：指令-数据边界是共同根因）
- **claim**: Agent 安全不能只治理模型层（LLM Top10）与协议层（MCP）——skill 定义了"工具如何被编排成行为"，是独立且当前最薄弱的安全域；其独特脆弱性源于**自然语言指令被当作执行指令**。
- **证据链**: [ast01:Why Unique][ast08:Why Unique][ast10:Why Unique]；MCP 轮 KO-07 的"垂直分层"在此获得行为层补全
- **五条件**: 内聚 ✅（指令-数据边界机制边）| 跨实例 ✅（AST01/05/08/10 四文件独立陈述）| 解释扩大 ✅（从单风险到全生态）| 可命名 ✅ | 可回溯 ✅
- **状态**: Validated Pattern（本仓库内跨 4+ 风险页重复出现；跨项目验证见 C-07）

## KO-02 · 攻击经济学的"双向量模型"
- **knowledge_layer**: L3 Pattern
- **aggregation_rule**: **R1 机制簇**（EK-01/EK-02/EK-03/EK-07/EK-17：同一"恶意内容继承 agent 权限"机制，跨 registry/parser/repo-config 三实例）
- **claim**: Agent skill 攻击的最小有效单元是"伪装合法内容 + 继承执行权限"——注册表投毒、元数据载荷、repo 配置执行、身份文件后门是同一机制在不同入口的实例化。
- **证据链**: [ClawHavoc][ToxicSkills][CVE-2025-59536][SOUL.md persistence]
- **状态**: Validated Pattern（本仓库 4+ 独立实例）

## KO-03 · 权限检查必须在意图层而非工具调用层
- **knowledge_layer**: L4 Cognitive Model
- **aggregation_rule**: **R3 不变量簇**（EK-03/EK-12/EK-13/EK-14/EK-22：五条 EK 汇聚到同一安全不变量"工具可调用 ≠ 被授权执行"）
- **claim**: 权限清单声明（manifest）与运行时强制（policy layer）分开是必要条件；模型把 skill 输出当 operator 指令是 LPCI 根因——"指令层级 + 持久状态显式同意"是不变量。
- **证据链**: [ast03:LPCI][ast03:Mitigations 7-8][execution-boundary-flow.md][universal-skill-format.md]
- **状态**: Validated Pattern（映射 MCP 轮 EK-25 allowed-tools 审批门 → 跨协议一致性 ✅）

## KO-04 · "签名≠安全"与"扫描≠防线"的防御层级观
- **knowledge_layer**: L4 Cognitive Model
- **aggregation_rule**: **R4 主题簇**（EK-24/EK-25/EK-08/EK-28/EK-30：签名/扫描/注册表治理是互补而非替代的防御层）
- **claim**: Agent skill 生态没有任何单点防御可独立成立：签名证明作者、扫描可被语言变异性绕过、注册表可被投毒——安全 = 签名（作者）+ 行为扫描（内容）+ 治理（清单/审批/审计）的组合，且必须以"可绕过"为默认假设设计。
- **证据链**: [ast01:Mitigation 1][Trail of Bits 绕过][ast02:registry poisoning]
- **状态**: Validated Pattern（本仓库多文件一致陈述）

## KO-05 · 记忆/身份文件是 agent 的"行为密钥"
- **knowledge_layer**: L3 Pattern
- **aggregation_rule**: **R1 机制簇**（EK-26/EK-27/EK-14：身份/记忆文件防写机制跨攻击场景与防御格式两处出现）
- **claim**: 在 agentic 系统中，身份是行为性的（SOUL.md/MEMORY.md 编码了可复制的行为状态），因此身份文件既是持久后门载体又是凭据等价物——必须 deny_write 默认 + 显式授予 + 版本控制。
- **证据链**: [ast01:Soul Persistence / Identity Cloning][Vidar 窃取身份文件][universal-skill-format:deny_write]
- **状态**: Validated Pattern（对照 dsh-memory-evolve 记忆安全 → 跨项目候选 C-05）

## KO-06 · B1-B4 管线化威胁建模方法论
- **knowledge_layer**: L5 Methodology
- **aggregation_rule**: **R2 因果链簇**（EK-21→EK-22→EK-18→EK-19：Developer→Agent→Repo→CI/CD→Prod 五步链 + 每步的 ALLOW/DENY + 收据 + 治理映射）
- **claim**: 对 AI coding agent 的安全治理应沿信任边界管线化（B1-B4），识别"控制点序列"而非孤立控制；每边界映射 AST 风险 + 明确控制动作。
- **证据链**: [trust-boundary-model.md:Four Boundaries][execution-boundary-flow.md]
- **状态**: **DOWNGRADED 候选**——原声明"通用 AI 管线威胁模型"，独立审计判定其证据仅覆盖 coding agent 工作流（Developer/Repo/CI-CD），对通用 agent（非 coding）适用性未证实 → 见 Validation V-11。

## KO-07 · 外部文本依赖需要"prose lockfile"治理
- **knowledge_layer**: L4 Cognitive Model
- **aggregation_rule**: **R3 不变量簇**（EK-05/EK-10/EK-15/EK-07：文本依赖不可固定性汇聚到不变量"可变输入必须 pin 或视为不可信"）
- **claim**: skill 的外部指令源（URL/远程文档）是唯一"可变、越界、无 lockfile"的依赖——治理上必须把 prose 当代码依赖同等对待（内容 hash/pinning/持续重扫），否则"被审计的 skill 不是运行的 skill"。
- **证据链**: [ast05:Description][ast07:Update Drift][Air Security 26,000 agents]
- **状态**: Validated Pattern（跨 AST05/AST07/AST04 一致）

---

## KO 聚合规则矩阵

| KO | Rule | 簇内 EK | 边类型 | 解释范围扩大论证 |
|----|------|---------|--------|-----------------|
| KO-01 | R4 主题簇 | EK-01,04,12,26,27 | mechanism (指令-数据边界) | 单风险→全生态安全域 |
| KO-02 | R1 机制簇 | EK-01,02,03,07,17 | mechanism (伪装+继承权限) | 单事件→攻击经济学模型 |
| KO-03 | R3 不变量簇 | EK-03,12,13,14,22 | constraint→"意图级授权" | 权限检查→第一性原则 |
| KO-04 | R4 主题簇 | EK-24,25,08,28,30 | subsystem (防御组件) | 单工具→防御层级观 |
| KO-05 | R1 机制簇 | EK-26,27,14 | mechanism (身份文件防写) | 攻击面→行为密钥 |
| KO-06 | R2 因果链簇 | EK-21,22,18,19 | causal (B1→B4→收据→治理) | 威胁建模→方法论 |
| KO-07 | R3 不变量簇 | EK-05,10,15,07 | constraint→"可变输入不可信" | 文本依赖→治理原则 |

## Reconciliation 注记（ARCH-2026-09-26-001）
- **KO-06 DOWNGRADED（V-18）**：盲重建证实 trust-boundary-model.md 标题限定 "for AI Code Generation Agents"——KO-06 的通用方法论声明降级为 **coding-agent 管线特化方法论**；对通用 agent（个人助理/浏览器 agent）的外推见 C-08。

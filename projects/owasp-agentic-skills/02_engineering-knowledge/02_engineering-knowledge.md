# 02 Engineering Knowledge — EK Graph（OWASP AST10）

> 31 条 EK，48 出边（avg 1.55），0 游离。证据锚点格式 `[文件:section]`。边类型：mechanism / subsystem / causal / dependency / constraint / contrast。

## EK 清单

### EK-01 恶意 skill 双向量攻击（代码层 + 自然语言指令层）
- **分类**: AGENT / ATTACK
- **知识**: 恶意 skill 同时利用 shell/python 代码载荷与 SKILL.md 散文指令两个向量；Snyk ToxicSkills 证实 100% 恶意 skill 两者兼备。"三行 markdown 足以外泄 SSH key"。
- **证据**: [ast01.md:Why Unique][index.md:Incident 2026-02-03]
- **links**: mechanism→EK-04（都是"指令被当作执行"）；subsystem→EK-02；causal→EK-03（载荷需要权限放大）

### EK-02 注册表/供应链投毒是生态级攻击
- **分类**: SUPPLY CHAIN / ATTACK
- **知识**: ClawHub 成为首个被系统性投毒的 agent skill 注册表——ClawHavoc 1,184 恶意 skills / 12 账户 / 单 C2 IP；发布门槛仅"一周龄 GitHub 账户 + SKILL.md"。
- **证据**: [index.md:Incident Jan-Feb 2026][ast02.md:Why Unique]
- **links**: subsystem→EK-01；causal→EK-03；contrast↔EK-08（发布门槛 vs 扫描防线）

### EK-03 权限放大：skill 继承 agent 全部凭据
- **分类**: PERMISSION / ARCHITECTURE
- **知识**: 恶意/被投毒 skill 执行时继承 host agent 完整权限（API keys/SSH/钱包/浏览器数据/shell），无 per-skill 权限作用域——"OpenClaw tools run on the host for the main session, so the agent has full access"。
- **证据**: [ast03.md:Real-World Evidence][ast06.md:Description]
- **links**: dependency→EK-06（需隔离）；causal←EK-01/EK-02；constraint→EK-14（Universal Format 权限清单）

### EK-04 指令-数据边界是 skill 安全第一性原理
- **分类**: PRINCIPLE / ATTACK
- **知识**: skill 文本（散文/外部文档/记忆文件）被模型当作**指令**而非**数据**执行；"无限语言变异性"使正则/签名扫描失效（"retrieve the file... send it to the address below"无代码签名可测）。
- **证据**: [ast08.md:Why Unique][ast05.md:Why Unique]
- **links**: mechanism→EK-01/EK-12（LPCI）；constraint→EK-13（指令层级）；causal→EK-05

### EK-05 外部文本指令是"可变、越界、不可固定"的依赖
- **分类**: SUPPLY CHAIN / ATTACK
- **知识**: skill 引用的外部文档/URL 运行时被当作指令消费；无 hash 固定、无 lockfile、签名不覆盖 URL 返回内容；PoC 恶意 skill（Air Security）达 26,000 agents 全扫描器放行；12.4% 活跃 skills 依赖不可信外部指令源。
- **证据**: [ast05.md:Description][index.md:Incident 2026-06]
- **links**: causal→EK-04；mechanism→EK-08（扫描盲区）；constraint→EK-15（源清单/pinning）

### EK-06 沙箱缺失 = 每个 skill 都是潜在全系统入侵
- **分类**: RUNTIME / ARCHITECTURE
- **知识**: skill 默认与 host agent 同安全上下文执行（文件/网络/shell 全通），沙箱可选或默认关；135,000+ OpenClaw 实例公网暴露 + ClawJacked（CVSS 9.9 localhost WebSocket 劫持）证明隔离缺失的现实后果。
- **证据**: [ast06.md:Description][index.md:Incident 2026-02-26/03]
- **links**: dependency→EK-03；contrast↔EK-17（Claude Code repo 级执行）；constraint→EK-09（默认沙箱=平台开发职责）

### EK-07 元数据是双重攻击面（语义欺骗 + 解析执行）
- **分类**: PARSING / ATTACK
- **知识**: skill 元数据（name/description/permissions/risk_tier/YAML）既是安装者的信任信号，又是反序列化攻击面；unsafe deserialization 可在加载时、用户动作前触发载荷。
- **证据**: [ast04.md:Description][ast04.md:Why Unique]
- **links**: mechanism→EK-04；causal→EK-01；constraint→EK-14/EK-16（schema 校验 + 安全解析）

### EK-08 扫描器失效是结构性而非工程性
- **分类**: TESTING / ATTACK
- **知识**: pattern-matching 扫描器被绕过是语言变异性导致的结构缺陷：Trail of Bits 证实全部公开扫描器 <1 小时可绕过（padding 截断/二进制归档隐藏/LLM judge 提示注入）；SkillSpector 等语义扫描是补强而非根治。
- **证据**: [ast08.md:Description][index.md:Incident 2026-06-03][skill-scanner-integration.md]
- **links**: mechanism→EK-04；contrast↔EK-02；causal→EK-09

### EK-09 治理（AST09）是生态级最终防线
- **分类**: GOVERNANCE / RUNTIME
- **知识**: 扫描器可绕过、注册表可投毒 → 持久防线 = skill 清单 + 审批流 + 审计日志 + agentic identity 控制；53,000+ 暴露实例无 SOC 可见性 = 治理缺位实证。
- **证据**: [ast09.md][index.md:Incident 2026-03]
- **links**: dependency←EK-08；constraint→EK-19（AI TIPS 治理映射）；mechanism→EK-18（收据审计）

### EK-10 版本漂移双向危险（补丁缺失 + 伪补丁）
- **分类**: SUPPLY CHAIN / FAILURE
- **知识**: 无不可变 pinning 时，要么补丁不应用（漏洞敞开），要么"v1.0.1 补丁"本身是攻击者 payload；ClawJacked 补丁滞后窗口被利用。
- **证据**: [ast07.md:Description][index.md:Incident 2026-02-26]
- **links**: causal→EK-03；constraint→EK-14（content_hash pinning）；mechanism→EK-02

### EK-11 跨平台复用丢安全元数据（AST10）
- **分类**: PORTABILITY / FAILURE
- **知识**: skill 跨平台移植时安全属性不翻译（权限清单被剥离）；"MCP 标准化协议，AST10 治理行为抽象"——metadata-loss-simulator 展示移植后安全元数据丢失。
- **证据**: [ast10.md:Why Unique][index.md:Universal Skill Format]
- **links**: causal→EK-10（移植后 pinning 失效）；constraint→EK-14/EK-16；contrast↔MCP（协议 vs 行为层）

### EK-12 LPCI：逻辑层权限提升注入类
- **分类**: ATTACK / PERMISSION
- **知识**: 编码/延迟/条件触发的载荷（记忆/向量库/工具输出中）被模型当作 operator 级指令，skill 自主调用工具并跨会话写持久状态（Atta et al., arXiv:2507.10457）——权限检查在工具调用层而非意图层。
- **证据**: [ast03.md:Real-World Evidence LPCI]
- **links**: mechanism→EK-04；causal→EK-03；constraint→EK-13

### EK-13 指令层级是 LPCI 的解药
- **分类**: PRINCIPLE / MITIGATION
- **知识**: 严格指令层级 System > Operator > User > Skill/Tool Output；skill/工具输出永不提升为指令级；外部内容打 provenance 标记；持久状态变更需 operator 显式同意。
- **证据**: [ast03.md:Preventive Mitigations 7-8]
- **links**: constraint→EK-12/EK-04；mechanism→EK-14（deny_write 同理：身份文件显式授予）

### EK-14 Universal Format 把安全元数据变成可治理数据
- **分类**: CONFIG / MITIGATION
- **知识**: Universal Skill Format v1.0 规范化权限（files.read/write/deny_write、network.allow 域名白名单、shell 布尔）、risk_tier L0-L3、signature+content_hash、scan_status——使 AST08/09 机器可读、AST01/02 可 Merkle 校验。
- **证据**: [universal-skill-format.md:Canonical YAML][index.md:Format design rationale]
- **links**: constraint→EK-03/EK-10/EK-11；mechanism→EK-16；subsystem→EK-09

### EK-15 源清单 + 内容 pinning 治理外部指令
- **分类**: MITIGATION / SUPPLY CHAIN
- **知识**: AST05 缓解 = source inventory + content pinning + 持续重扫；把"文本依赖"当作"代码依赖"同等治理（hash/lockfile 语义移植到 prose）。
- **证据**: [ast05.md:Preventive Mitigations][index.md:Summary Table AST05]
- **links**: constraint→EK-05；mechanism→EK-10

### EK-16 安全解析 + schema 校验治理元数据载荷
- **分类**: MITIGATION / PARSING
- **知识**: AST04 缓解 = 静态分析 + 安全解析器 + 沙箱加载 + manifest 声明与沙箱观测行为校验；禁用危险 YAML/JSON tag。
- **证据**: [ast04.md:Preventive Mitigations][index.md:For Platform Developers]
- **links**: constraint→EK-07/EK-14；mechanism→EK-13

### EK-17 仓库级配置文件 = 执行层（Claude Code CVE）
- **分类**: ATTACK / RUNTIME
- **知识**: CVE-2025-59536（CVSS 8.7）/CVE-2026-21852（5.3）：克隆未信任项目即可在信任对话框前触发 RCE + API key 外泄——"repository-level configuration files now function as part of the execution layer"。
- **证据**: [index.md:Incident 2026-02-25][ast02.md:Real-World Evidence]
- **links**: contrast↔EK-06；causal→EK-04；constraint→EK-20（信任确认门）

### EK-18 双份收据（admission+outcome）实现防篡改审计
- **分类**: TESTING / MITIGATION
- **知识**: Bilateral Execution Receipt：执行前 admission（7 字段）+ 执行后 outcome（attempt_id 内容哈希 JCS→sha256 可独立重算）；operator 可改日志但 verifier 可独立校验收据；check.py 离线验证 denied-before-dispatch 属性（exit 0/1/2，无加密依赖）。
- **证据**: [ast09.md:Execution Receipts][examples/ast09-execution-receipts/check.py]
- **links**: mechanism→EK-09；causal→EK-09→EK-19；constraint→EK-21

### EK-19 企业治理层映射（AI TIPS 8-pillar）
- **分类**: GOVERNANCE / MITIGATION
- **知识**: mappings.md 把 AST10 映射到 AI TIPS 8-pillar（Cybersecurity/Privacy/Ethics/Transparency/Explainability/Regulations/Audit/Accountability）+ 六阶段 gated lifecycle + Trust Index 0-100；AST10 答"检查什么"，AI TIPS 答"何时/谁/何证据"。
- **证据**: [mappings.md:Executive Summary][mappings.md:Eight Pillars]
- **links**: constraint→EK-09；mechanism→EK-20；contrast↔MAESTRO（EK-23）

### EK-20 信任确认门：repo 配置不得先于用户信任执行
- **分类**: MITIGATION / RUNTIME
- **知识**: "Do not allow repository-controlled configuration to execute before explicit user trust confirmation"（Platform Developers #6）；B2 边界 human review gate 同旨。
- **证据**: [index.md:For Platform Developers][trust-boundary-model.md:B2]
- **links**: constraint→EK-17；mechanism→EK-13/EK-15

### EK-21 B1-B4 把威胁链管线化
- **分类**: THREAT MODEL / ARCHITECTURE
- **知识**: B1 Developer↔Agent（AST03/09）、B2 Agent↔Repo（AST01/02/04）、B3 Repo↔CI/CD（AST02/07/08）、B4 CI/CD↔Prod（AST06/08/09）；"识别控制点序列而非孤立控制"。
- **证据**: [trust-boundary-model.md:Four Boundaries]
- **links**: subsystem→EK-01~20（各 AST 归属）；causal→EK-22；contrast↔EK-23

### EK-22 策略执行层（Policy Enforcement Layer）在副作用前判定
- **分类**: RUNTIME / CONTROL
- **知识**: Execution Boundary Flow：LLM→Skill→Tool Request→Policy Enforcement Layer，副作用前确定性 ALLOW/DENY，决策返回调用层（proceed/retry/escalate/abort）。
- **证据**: [docs/diagrams/execution-boundary-flow.md]
- **links**: constraint→EK-13/EK-14；mechanism→EK-20；causal→EK-18（执行后收据）

### EK-23 MAESTRO 7 层映射 = 威胁定位坐标
- **分类**: THREAT MODEL / METHODOLOGY
- **知识**: CSA MAESTRO（L1 Foundation→L7 Ecosystem）为每个 AST 风险定位层组合（如 AST05→3/2/7/6、AST06→4/6/3）；使跨层风险分析可执行。
- **证据**: [index.md:MAESTRO Mapping]
- **links**: contrast↔EK-19/EK-21；subsystem→EK-01~20

### EK-24 签名证明作者而非安全（signing ≠ safety）
- **分类**: MITIGATION / PRINCIPLE
- **知识**: "A signature proves authorship, not safety: a verified publisher can still ship malicious content"；签名须绑可解析/可撤销发布者身份（key id + did:web + 公开验证 key），且与行为扫描/声誉组合。
- **证据**: [ast01.md:Preventive Mitigations 1]
- **links**: constraint→EK-14；contrast↔EK-08；causal→EK-25

### EK-25 Merkle root 签名实现注册表透明度
- **分类**: MITIGATION / SUPPLY CHAIN
- **知识**: 注册表级 Merkle root 签名 + 透明度日志（AST01/02 缓解）；"registry transparency, provenance tracking"。
- **证据**: [ast01.md:Mitigation 2][ast02.md:Summary Table]
- **links**: constraint→EK-24/EK-14；mechanism→EK-02 反制

### EK-26 身份文件（SOUL.md/MEMORY.md）是持久后门载体
- **分类**: MEMORY / ATTACK
- **知识**: 恶意 skill 写回身份/记忆文件（MEMORY.md/SOUL.md）→ 会话持久后门 + 身份克隆（behavioral identity 复制）；Vidar 变体专门窃取 openclaw.json/soul.md/memory.md。
- **证据**: [ast01.md:Attack Scenarios SOUL.md Persistence / Identity Cloning][index.md:Incident 2026-02]
- **links**: mechanism→EK-12（跨会话持久）；constraint→EK-14（deny_write）；subsystem→EK-27

### EK-27 记忆即指令通道（memory poisoning）
- **分类**: MEMORY / ATTACK
- **知识**: skill 向 MEMORY.md 注入恶意上下文 → 未来会话执行攻击者命令；身份文件防写 = deny_write 默认保护（identity files require explicit grant）。
- **证据**: [ast01.md:Memory Poisoning][universal-skill-format.md:deny_write]
- **links**: causal→EK-26/EK-12；constraint→EK-14

### EK-28 行为扫描优于模式扫描（semantic + behavioral）
- **分类**: TESTING / MITIGATION
- **知识**: AST08 缓解 = 语义 + 行为多工具管线（NVIDIA SkillSpector 静态 + 可选 LLM 语义分析，0-100 风险分，SARIF 输出）；发布时 + 安装时双点扫描。
- **证据**: [ast08.md:Preventive Mitigations][skill-scanner-integration.md:SkillSpector]
- **links**: contrast↔EK-08；constraint→EK-01/EK-05；mechanism→EK-16

### EK-29 平台安全模型分化（platform comparison）
- **分类**: ARCHITECTURE / CONTRAST
- **知识**: OpenClaw（容器沙箱 + ed25519 + 细粒度 scope + 发布/安装双扫描）vs Claude Code/Cursor（manifest 校验为主）vs VS Code（security analyzer）；"verified publisher" 徽章体系。
- **证据**: [platform-comparison.md:Platform Deep Dive]
- **links**: contrast↔EK-11；subsystem→EK-06；constraint→EK-14

### EK-30 扫描工具生态（SkillSpector 等）是治理组件
- **分类**: TOOLING / ECOSYSTEM
- **知识**: 扫描器（SkillSpector/ClawHub VirusTotal/LLM guard/skills.sh scanners）作为注册表与开发流水线组件；但全部可绕过 → 只能当治理的一层，不能当唯一防线。
- **证据**: [skill-scanner-integration.md][index.md:Incident 2026-06-03]
- **links**: dependency→EK-08/EK-28；subsystem→EK-09

### EK-31 事件响应流程已成型（incident playbook）
- **分类**: OPERATIONS / MITIGATION
- **知识**: incident-response.md 提供 skill 妥协响应剧本 + incident-template.md 结构化模板；metrics-monitoring.md 定义安全指标；remediation-guide.md 提供修复路径。
- **证据**: [incident-response.md][metrics-monitoring.md]
- **links**: subsystem→EK-09；mechanism→EK-18（收据支撑审计）

## 质量门统计
- 总 EK: 31
- 出边: 48（avg 1.55，≥1 ✅）
- 游离 EK: 0（<20% ✅）
- 聚合规则覆盖率: 100%（见 03）

## Reconciliation 注记（ARCH-2026-09-26-001）
- **EK-08 范围收紧（V-08 OVER_GENERALIZED 修正）**：原断言"扫描器失效是结构性"超出现有证据；收紧为"**模式/签名扫描对自然语言指令失效**"，语义扫描（SkillSpector 等）的进化路径未决（见 C-04）。
- **M-02 注记**：skill-development-guide.md 提供最小权限 WRONG/CORRECT 对照示例（full_system_access vs filesystem:read 白名单），作为 EK-14 的开发者侧补充。
- **M-03 注记**：metrics-monitoring.md 定义量化 KPI（关键漏洞 30 天内 100% 修复、每周跟踪），作为 EK-31 治理指标的展开。

# 06 Validation & Evidence — OWASP AST10

> 协议：Blind Reconstruction（阶段 A 独立重读仓库 → 阶段 B 与考古产物对比）。Repository is the Source of Truth。
> 禁止修改原考古产物；本报告记录判定、反例与 Reconciliation 建议。

## 阶段 A 独立发现（盲重建，未参考 KO/EK）
1. **收据 tamper 检测盲点**：check.py 的 `ATTEMPT_ID_PREIMAGE_FIELDS` 仅含 5 字段（agent_id/action_type/scope/policy_version/timestamp_ms），**不含 decision**——即篡改 `decision` 字段不会破坏 attempt_id 重算；tampered 样例篡改的是 scope 而非 decision，未覆盖该路径。断言 "exactly seven required fields（且无 signature 扩展字段）"。
2. **B1-B4 明确限定 coding agents**：trust-boundary-model.md 标题即 "for AI Code Generation Agents"——非通用 agent 管线模型。
3. **skill-development-guide.md 提供最小权限 WRONG/CORRECT 对照示例**（full_system_access vs filesystem:read 白名单）。
4. **metrics-monitoring.md 定义量化 KPI**（关键漏洞 30 天内 100% 修复、每周跟踪、CVE 修复计数）。
5. **user-notification.md / risk-assessment.md（交互 HTML）**：通知模板与交互式评估工具存在，考古产物未单独提取。
6. **ClawHavoc 时间线细节**：case-studies.md 提供 12/2025 侦察 → 01/03 首发 → 01/15 峰值（Top7 中 5 恶意）→ 01/22 发现 → 01/28 清除。

## 阶段 B 判定（20 项正式判定 + 3 遗漏）

| # | 对象 | 判定 | 依据（盲重建独立证据） |
|---|------|------|------------------------|
| V-01 | EK-01 双向量攻击 | CONFIRMED | ast01 "100% of malicious skills combined both" + Snyk Feb 2026 三行 markdown 案例 |
| V-02 | EK-02 注册表投毒 | CONFIRMED | ClawHavoc 1,184/12 账户/单 C2 IP 可追溯（index + case-studies 一致） |
| V-03 | EK-03 权限放大 | CONFIRMED | ast03 原文 "tools run on the host for the main session... full access" |
| V-04 | EK-04 指令-数据边界 | CONFIRMED | ast05/ast08 两文件独立论证，语言变异性实例一致 |
| V-05 | EK-05 外部文本指令 | CONFIRMED | ast05 Description 完整因果链（可变/越界/无 lockfile）+ Air 26K agents |
| V-06 | EK-06 沙箱缺失 | CONFIRMED | ast06 + 135,000+ 实例数字（index March 2026） |
| V-07 | EK-07 元数据双重攻击面 | CONFIRMED | ast04 语义/解析两弱点结构与恶意样例一致 |
| V-08 | EK-08 扫描器失效 | **OVER_GENERALIZED** | 盲重建：Trail of Bits <1h 绕过是 2026-06 现状，"结构性不可根治"超出证据；语义扫描（SkillSpector）进化路径未决 → 收紧为"模式扫描对语言指令失效" |
| V-09 | EK-09 治理最终防线 | CONFIRMED | AST09 缓解清单 + 53K 无 SOC 可见性数字 |
| V-10 | EK-10 版本漂移 | CONFIRMED | ast07 双向危险论证 + ClawJacked 补丁滞后 |
| V-11 | EK-11 跨平台丢元数据 | CONFIRMED | ast10 "not a protocol problem" + metadata-loss-simulator 存在 |
| V-12 | EK-12 LPCI | CONFIRMED | ast03 引用 arXiv:2507.10457 同行评审证据 |
| V-13 | EK-14 Universal Format | PARTIALLY_CONFIRMED | 格式规范本身成立（universal-skill-format.md），但落地未证实（proposal 态）→ 已入 C-01 |
| V-14 | EK-17 repo 配置=执行层 | CONFIRMED | CVE-2025-59536/2026-21852 记录 + "before any user consent dialog" 原文 |
| V-15 | EK-18 双份收据 | CONFIRMED | check.py 代码级验证（7 字段/DENY/attempt_id 重算/exit 契约），JSON 样例存在 |
| V-16 | EK-24 签名≠安全 | CONFIRMED | ast01 原文 "A signature proves authorship, not safety" |
| V-17 | EK-26 身份文件后门 | CONFIRMED | ast01 SOUL.md persistence/identity cloning + Vidar 窃取 |
| V-18 | KO-06 B1-B4 方法论 | **DOWNGRADED** | 盲重建发现标题限定 "for AI Code Generation Agents"——L5 通用方法论声明降为 coding-agent 管线特化（对应 C-08） |
| V-19 | Flow-1 Control ESCALATE 分支 | PARTIALLY_CONFIRMED | admission receipt 定义 decision 含 ESCALATE，但 check.py/样例仅验证 DENY 路径，ESCALATE 分支无实现证据 |
| V-20 | KO-01 行为抽象层 | CONFIRMED | ast01/08/10 三文件独立重复"行为层"表述，MCP 轮 KO-07 交叉一致 |

**判定统计**：CONFIRMED 15 / PARTIALLY_CONFIRMED 2 / DOWNGRADED 1 / OVER_GENERALIZED 1 / MISSING 3 / CONTRADICTED **0** / NEEDS_HUMAN_REVIEW 0

## 遗漏（MISSING，盲重建独立发现）
- **M-01（中）** check.py preimage 不含 `decision`——篡改 decision 字段不破坏 attempt_id 重算，收据的"防篡改"仅覆盖 preimage 字段；tampered 样例未覆盖该路径。
- **M-02（低）** skill-development-guide.md 的最小权限设计模式（WRONG/CORRECT 示例）是独立安全开发资产，EK 未单独提取。
- **M-03（低）** metrics-monitoring.md 的量化 KPI（30 天 100% 关键漏洞修复）为运营治理提供可执行指标，未被 EK-31 展开。

## 3 成功（高价值确认）
1. **EK-04 指令-数据边界**被 ast05+ast08 双文件独立证实，成为 KO-01 的机制边核心——这是整个框架的第一性原理。
2. **EK-18 双份收据 + 离线验证**通过实际运行 check.py 逻辑核验（JSON 样例 + exit 契约 + denied-before-dispatch 属性）——文档型仓库中罕见的可执行证据。
3. **EK-24 签名≠安全**的显式表述（"authorship, not safety"）直接支撑 KO-04 防御层级观，与扫描器绕过实证形成完整论证。

## 3 错误（考古产物偏差）
1. **KO-06 OVER-DECLARED**：将 B1-B4 写为通用威胁建模方法论，盲重建证实其明确限定 coding-agent 工作流 → DOWNGRADED。
2. **EK-08 过度断言**："结构性失效"超出单次实证 → OVER_GENERALIZED，收紧 scope。
3. **Flow-1 状态列表超集**：states 含 ESCALATED 但仓库无该分支实现/示例证据 → PARTIALLY_CONFIRMED。

## 反例（Counterexamples）
- **V-08 反例**：SkillSpector 的 LLM 语义分析层暗示扫描器可进化——"所有扫描必然失效"不成立（无永久失效证据）。
- **V-11 反例**：Claude Code 与 VS Code 有 manifest 校验/security analyzer（platform-comparison）——非所有平台都零元数据，AST10 风险为"移植时丢失"而非"从未存在"。
- **V-19 反例**：无任何收据样例含 ESCALATE 决策——该状态仅存在于 schema，不证明运行时可达。

## Benchmark case（跨项目对照：协议层 vs 行为层安全控制收敛）
对照 MCP（ARCH-2026-09-25-001）与 AST10：

| 维度 | MCP（协议层） | AST10（行为层） | 收敛模式 |
|------|--------------|----------------|---------|
| 核心安全原语 | 每请求 `_meta` 自描述 + 嵌套技能 fresh consent（SEP-2640） | deny_write 身份文件 + network.allow 域名白名单 | **"默认拒绝 + 显式授予"双原语** |
| 审批门 | allowed-tools 审批门（MCP EK-25） | Policy Enforcement Layer ALLOW/DENY（B4 控制） | **副作用前确定性裁决** |
| 权限检查层级 | 工具调用层 | 意图层（LPCI 教训） | MCP 需补意图级检查 |
| 记忆安全 | Roots/Sampling 废弃（2026-07-28） | MEMORY.md deny_write + 显式授予 | **记忆/身份文件需显式授权** |
| 治理 | SEP=PR 编号 ADR + AGENTS.md | OWASP Incubator + leaders + Google Doc 评审 | **规范与治理同库** |

Benchmark 结论：MCP 轮 EK-25/C-08（"扩展默认关闭→事实分裂"）在 AST10 获得**同构支持**（deny_write/network deny:*/risk_tier 默认值设计）；"意图级权限检查"是 MCP 层未覆盖、行为层强调的差异点（C-06 假说证据 +1）。

## Reconciliation 建议（写入 Corpus 版本，不修改原产物）
1. V-08：EK-08 scope 收紧为"模式扫描对语言指令失效"（Corpus 02 文件加注）。
2. V-18：KO-06 标注 DOWNGRADED（coding-agent 特化），C-08 记录外推待验证。
3. M-01：05_candidates 增加 C-09（attempt_id preimage 不含 decision 的篡改盲点）。
4. M-02/M-03：02 EK 增补注记（最小权限模式/KPI 量化）→ 以 reconciliation 注记形式并入 Corpus 版本。

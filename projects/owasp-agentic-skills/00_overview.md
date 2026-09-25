# 00 Overview — OWASP Agentic Skills Top 10（AST10）

- **run_id**: ARCH-2026-09-26-001
- **repository**: https://github.com/OWASP/www-project-agentic-skills-top-10.git
- **commit**: `d6f7d7d0de314f52a83a85d1828e06ab096e595c`（2026-08-12）
- **skill_version**: knowledge-archaeology v3.2
- **日期**: 2026-09-26

## 一句话定位

OWASP 首个 **Agentic Skills（agent 技能/行为层）安全威胁分类框架**：以 AST01-AST10 十个风险条目 + B1-B4 信任边界模型 + Universal Skill Format v1.0，回答"agent 的 skill 执行层如何被攻破、如何防御"。Mental Model：*MCP = how the model talks to tools; AST10 = what those tools actually do*。

## 考古概览

| 层 | 产物 | 规模 |
|---|---|---|
| Project Layer | 项目地图 | 架构/威胁分类法/治理/工具链 |
| Engineering Knowledge | EK Graph | 31 条 EK / 48 出边（avg 1.55）/ 0 游离 |
| Knowledge Layer | KO | 7 个（R1-R4 聚合规则全覆盖） |
| Flow Atlas | 七类流 | Control / State / Data / Evidence / Authority / Memory / Policy |
| Candidates | 未验证假说 | 8 条（含 2 条跨项目假说） |
| Validation | 盲重建判定 | 20 项（见 06） |

## 五个核心认知资产（Top-level takeaways）

1. **Skill 是"行为抽象层"而非"协议层"**：MCP 定义"模型怎么调用工具"，Skill 定义"工具怎么被编排成行为"。安全不能只靠模型或协议层，必须治理行为层（index.md Overview / ast10.md）。

2. **"指令与数据的边界"是 skill 安全的第一性原理**：skill 的文本（markdown 散文、外部文档、记忆文件）被 agent 当作**指令**而非**数据**执行——这正是 pattern 扫描器失效（AST08）、外部指令投毒（AST05）、权限检查失败（AST03 LPCI）的共同根因。

3. **恶意 skill 同时攻击代码层与自然语言指令层**：Snyk ToxicSkills 证实 100% 恶意 skill 同时使用两种攻击向量（ast01.md）；"三行 markdown 即可外泄 SSH key"（Snyk, Feb 2026）。

4. **治理是 skill 生态的最终防线**：扫描器可在 <1 小时被绕过（Trail of Bits）、注册表可被系统投毒（ClawHavoc 1,184 skills）；签名证明"作者身份"而非"安全性"（ast01 mitigation 1）；持久防线 = 权限清单 + 运行时强制 + 供应链 provenance + 审计（AST09/Universal Format）。

5. **"Lethal Trifecta"是威胁建模的判据**：私密数据访问 + 不可信内容暴露 + 外部通信能力三者同时满足 = 危险 skill（Simon Willison / Palo Alto Networks, 2026）。B1-B4 信任边界模型把这一判据管线化（Developer↔Agent↔Repo↔CI/CD↔Production）。

## 与已有 Corpus 的连接
- **MCP（ARCH-2026-09-25-001）**：MCP 轮 EK-25（allowed-tools 审批门）/C-08（扩展默认关闭→事实分裂）在此仓库获得独立佐证——AST03 权限清单、AST05 外部指令投毒、SEP-2640 嵌套技能 fresh consent 与 OWASP "identity files 需显式授予"（Universal Format deny_write）同构。
- **skillfortify（skill 供应链安全）**：OWASP AST02（供应链投毒）与 skillfortify 的 skill 供应链治理直接同域，互为基准。
- **agent-governance-toolkit（AGT）**：AGT 映射 ASI01-10；本仓库 mappings.md 把 AST10 映射到 AI TIPS 8-pillar——同一治理主题的两套实现。
- **openclaw / dsh-memory-evolve / omnigent**：AST 平台矩阵直接覆盖 OpenClaw（SKILL.md），identity files（SOUL.md/MEMORY.md）防护与 dsh-memory-evolve 的记忆安全相关。

## 验证结论
PASS_WITH_NOTES：0 CONTRADICTED；1 OVER_GENERALIZED（"扫描器无用论"断言强度过高→降级为 AST08 子结论）；1 DOWNGRADED（B1-B4 威胁模型的通用性声明降级为 coding-agent 管线特化）；2 PARTIAL；2 MISSING（antivirus 集成细节 / check.py 边界用例）。跨项目假说 C-07/C-08 获得支持。

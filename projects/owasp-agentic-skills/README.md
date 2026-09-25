# OWASP Agentic Skills Top 10（AST10）知识考古

**一句话定位**：OWASP 首个 **Agent 技能（skill）行为抽象层**安全威胁框架——AST01-AST10 十大风险 + B1-B4 信任边界模型 + Universal Skill Format v1.0。Mental Model：*MCP = how the model talks to tools; AST10 = what those tools actually do*。回答"agent 的 skill 执行层如何被攻破、如何防御"。

## 考古信息
- **run_id**: ARCH-2026-09-26-001（每日 Knowledge Archaeology Runtime SOP）
- **commit**: `d6f7d7d0de314f52a83a85d1828e06ab096e595c`（2026-08-12）
- **框架版本**: v1.0-2026（OWASP Incubator）
- **skill_version**: knowledge-archaeology v3.2
- **日期**: 2026-09-26

## 产物清单
| 目录/文件 | 内容 |
|---|---|
| 00_overview.md | 考古概览 + 5 个核心认知资产 + Corpus 连接 |
| 01_project-layer/ | 项目地图（架构/威胁分类法/生命周期/数据结构/治理/依赖） |
| 02_engineering-knowledge/ | 31 条 EK（EK Graph，六类边，48 出边，0 游离） |
| 03_knowledge-layer/ | 7 个 KO（R1-R4 聚合规则矩阵 + 五层阶梯） |
| 04_flow-atlas/ | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates/ | 8 条假说（含 4 条跨项目假说 + Reconciliation 增补 C-09） |
| 06_validation/ | 盲重建验证（20 判定：15 CONFIRMED / 2 PARTIAL / 1 DOWNGRADED / 1 OVER_GENERALIZED / 3 MISSING / 0 CONTRADICTED） |
| archaeology-runs/ARCH-2026-09-26-001/ | 本 run 元数据 + job manifest + 仓库快照 |

## 关键发现（Top 5）
1. **Skill 是"行为抽象层"而非协议层**：MCP 定义"模型怎么调用工具"，skill 定义"工具怎么被编排成行为"——安全必须治理行为层（ast10 "not a protocol problem; a behavioral abstraction problem"）
2. **指令-数据边界是第一性原理**：skill 文本被当**指令**而非**数据**执行 → 模式扫描失效（AST08）、外部指令投毒（AST05）、LPCI 权限提升（AST03）的共同根因
3. **恶意 skill 双向量攻击**：代码层 + 自然语言指令层（Snyk ToxicSkills 100% 恶意 skill 两者兼备；三行 markdown 外泄 SSH key）
4. **无单点防御**：签名=作者≠安全、扫描可被语言变异性绕过（<1h）、注册表可被投毒（ClawHavoc 1,184）——防线 = 权限清单 + 运行时强制 + provenance + 审计
5. **Lethal Trifecta 判据**：私密数据 + 不可信内容 + 外部通信三者齐备 = 危险 skill；B1-B4 把威胁链管线化（Developer↔Agent↔Repo↔CI/CD↔Prod）

## 验证结论
PASS_WITH_NOTES：0 CONTRADICTED；1 OVER_GENERALIZED（EK-08 扫描器断言收紧）；1 DOWNGRADED（KO-06 B1-B4 限定 coding agents）；3 MISSING（收据 preimage 盲点/最小权限模式/KPI 指标）；Benchmark 确认与 MCP 轮（09-25）"默认拒绝+显式授予"双原语收敛。

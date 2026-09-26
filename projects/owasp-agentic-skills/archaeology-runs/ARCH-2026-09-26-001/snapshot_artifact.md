# Snapshot Artifact — OWASP Agentic Skills Top 10

- **run_id**: ARCH-2026-09-26-001
- **repository**: https://github.com/OWASP/www-project-agentic-skills-top-10.git
- **mode**: initial（浅克隆，depth 1）
- **HEAD commit**: `d6f7d7d0de314f52a83a85d1828e06ab096e595c`
- **HEAD 提交说明**: "Replace whitepaper PDF and cover with updated version"（2026-08-12 12:27:33 -0400）
- **文件数**: 176
- **快照时间**: 2026-09-26（本地）

## 项目基础地图

| 维度 | 内容（可追溯至仓库实际文件） |
|---|---|
| **定位** | OWASP 首个 Agentic Skills 安全威胁分类框架（AST01-AST10），"Skill 执行层 = agent 真实世界影响的载体"，Mental Model：*MCP = how the model talks to tools; AST10 = what those tools actually do*（index.md Overview） |
| **类型** | 规范/文档仓库（Jekyll GitHub Pages 站点，无运行时产品），OWASP Incubator 项目 |
| **语言/技术** | Markdown + HTML（Jekyll remote_theme `owasp/www--site-theme`，_config.yml）；Python（tools/ 生成器）；无 JS 运行时 |
| **入口** | index.md（35KB 主页：Overview → Incident Timeline → Summary Table → MAESTRO Mapping → Universal Skill Format → Getting Started） |
| **核心模块** | ast01.md~ast10.md（10 风险页，统一结构：Description / Why Unique / Real-World Evidence / Attack Scenarios / Preventive Mitigations / OWASP Mapping）；checklist.md（AST01-10 评估清单 + B1-B4 Pipeline Trust Boundary Review）；trust-boundary-model.md（B1-B4 信任边界模型）；mappings.md（AI TIPS 8-pillar 企业治理映射）；docs/diagrams/execution-boundary-flow.md（策略执行层 ALLOW/DENY 流） |
| **核心数据结构** | Universal Agentic Skill Format v1.0（index.md + universal-skill-format.md）：name/version/platforms/author(identity+signing_key)/permissions(files.read/write/deny_write, network.allow 域名白名单, shell 布尔, tools)/requires/risk_tier(L0-L3)/scan_status/signature/content_hash/changelog |
| **状态/生命周期** | 项目路线图：Q2 2026 Foundation（AST01-06 完稿）→ Q3 2026 Completion（AST07-10 + Universal Format v1.0 RC）→ Q4 2026 Launch（v1.0 + Flagship 申请）；当前 version 1.0-2026（README 徽章） |
| **测试体系** | 无传统单测；examples/ast09-execution-receipts/check.py（纯 std-lib 离线校验器：7 字段存在性 + decision=DENY + attempt_id RFC 8785/JCS→sha256 重算，exit 0/1/2）；tools/build_pptx.py 由 astNN.md 实时解析生成 deck（CI workflow .github/workflows/build-pptx.yml） |
| **配置** | _config.yml（Jekyll 构建/集合/排除）、Gemfile、.gitignore |
| **权限与治理机制** | OWASP Incubator 治理：leaders.md（8 位：Ken Huang 主导 + 7 co-leads）、CONTRIBUTING.md、MAINTENANCE.md、SECURITY.md（上报走 leaders）；v1 评审走公开 Google Doc；CC-BY-SA-4.0 |
| **外部依赖** | OWASP 站点主题（remote_theme）、python-pptx/Pillow（tools 可选）、OWASP GenAI Initiative 生态 |

## 关键证据数据点（index.md 实证数字，全部可追溯）

| 指标 | 数值 | 来源标注 |
|---|---|---|
| 扫描 skills 总数 | 3,984 | Snyk ToxicSkills（2026-02） |
| 含安全缺陷 | 1,467（36.82%） | 同上 |
| 关键级问题 | 534（13.4%） | 同上 |
| 确认恶意 payload | 76+ | 同上 |
| ClawHavoc 恶意 skills | 1,184（12 发布者账户，单一 C2 IP 91.92.242[.]30） | Antiy CERT / Koi Security（2026-01/02） |
| OpenClaw 公网暴露实例 | 135,000+（53,000+ 关联既往入侵） | SecurityScorecard（2026-03） |
| ClawJacked CVE | CVE-2026-28363，CVSS 9.9 | Oasis Security（2026-02-26） |
| Claude Code 仓库级 RCE | CVE-2025-59536（CVSS 8.7）/ CVE-2026-21852（5.3） | Check Point（2026-02-25 披露） |
| 活跃 skills 依赖不可信外部指令源 | 142,836 中 17,822（12.4%，6.7M installs） | Air Security（2026-06） |
| SkillJacking 可劫持依赖 skills | 925 skills / ~134K agents | Air Security（2026-07-02） |
| 公开扫描器绕过时间 | <1 小时（Trail of Bits，2026-06-03） | index.md Incident Timeline |
| USENIX 2026 测量 | 98,380 skills / 157 恶意 / 632 漏洞（avg 4.03）/ 73.2% 影子功能 / 54.1% 单一集群 | Liu et al., arXiv:2602.06547 |

## 边界与例外（快照记录）
- 仓库为文档型：Flow Atlas 的"代码级"证据以 docs/diagrams 与 examples/ 可执行验证器为准，风险页攻击场景属威胁建模内容（非实现代码）
- 9 个 AST 文件均含 OWASP Mapping（LLM01/09、ASVS V4 等）——映射声明与 AST 本体同源
- examples/ast09 收据格式为"提案级实现"（proposals/ast-fixture-corpus 仍在提案态），未在主流平台落地

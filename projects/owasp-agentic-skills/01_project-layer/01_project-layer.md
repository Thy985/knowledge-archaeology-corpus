# 01 Project Layer — OWASP Agentic Skills Top 10

## 1.1 项目定位
OWASP Incubator 项目，首个专门针对 **AI Agent Skills（技能/行为抽象层）** 的安全威胁框架（README.md / index.md）。与 OWASP LLM Top10（模型层）、MCP（工具协议层）互补：**AST10 治理"工具如何被编排成行为"的行为层**（ast10.md "This is not a protocol problem; it is a behavioral abstraction problem"）。

## 1.2 架构（文档体系）
```
OWASP AST10 仓库（Jekyll GitHub Pages，176 文件）
├── 威胁分类核心（10 风险页）
│   ├── AST01 Malicious Skills (Critical)
│   ├── AST02 Supply Chain Compromise (Critical)
│   ├── AST03 Over-Privileged Skills (High)
│   ├── AST04 Insecure Metadata (High)
│   ├── AST05 Untrusted External Instructions (High)
│   ├── AST06 Weak Isolation (High)
│   ├── AST07 Update Drift (Medium)
│   ├── AST08 Poor Scanning (Medium)
│   ├── AST09 No Governance (Medium)
│   └── AST10 Cross-Platform Reuse (Medium)
├── 聚合入口
│   ├── index.md（35KB：Overview / Incident Timeline / Summary Table / MAESTRO / Universal Format / 指南）
│   ├── top10.md（可视化概览）
│   ├── checklist.md（19KB：AST01-10 检查清单 + B1-B4 审查 + 扫描工具推荐）
│   └── mappings.md（AI TIPS 8-pillar 企业治理映射）
├── 信任边界模型
│   ├── trust-boundary-model.md（B1-B4 管线威胁模型，Alok Tibrewala, OWASP BASC 2026）
│   └── docs/diagrams/execution-boundary-flow.md（策略执行层 ALLOW/DENY 流）
├── 操作工具
│   ├── universal-skill-format.md（v1.0 规范：权限/签名/风险分级清单）
│   ├── skill-scanner-integration.md（NVIDIA SkillSpector 等扫描集成）
│   ├── risk-assessment.md / metrics-monitoring.md / incident-response.md / remediation-guide.md
│   └── api-documentation.md / user-notification.md / solutions.md
├── 可执行示例（唯一"运行代码"）
│   └── examples/ast09-execution-receipts/（check.py 离线校验器 + 收据 JSON）
├── 提案
│   └── proposals/ast-fixture-corpus/（AST09 测试语料提案）
├── 工具
│   └── tools/（build_pptx.py / build_pdf.py：从 astNN.md 解析生成 12 页 deck）
└── 治理
    ├── leaders.md（8 位） / CONTRIBUTING.md / MAINTENANCE.md / SECURITY.md / info.md
    ├── 案例库 case-studies.md / threat-intelligence.md / community-contribution.md
    └── 培训 training-certification.md / tutorial-videos.md / tab_*.md
```

## 1.3 核心抽象
1. **AST01-10 威胁分类法**：10 个命名风险，统一结构（Description / Why Unique / Real-World Evidence / Attack Scenarios / Preventive Mitigations / OWASP Mapping），每项标 Severity + Platforms Affected（ast01-ast10.md）
2. **Lethal Trifecta**：私密数据 + 不可信内容 + 外部通信 三条件威胁判据（index.md）
3. **B1-B4 Trust Boundary**：Developer↔Agent↔Repo↔CI/CD↔Production 四信任边界管线威胁模型（trust-boundary-model.md）
4. **Universal Skill Format v1.0**：跨平台规范化 manifest（permissions.files.read/write/deny_write、network.allow 域名白名单、shell 布尔、risk_tier L0-L3、signature+content_hash、scan_status）（universal-skill-format.md / index.md）
5. **MAESTRO 7 层映射**：CSA MAESTRO（L1 Foundation Models → L7 Ecosystem）与 AST10 的层定位（index.md）
6. **Bilateral Execution Receipt**：admission receipt + outcome receipt，attempt_id = 内容哈希（RFC 8785 JCS→sha256）（ast09.md / examples/ast09-execution-receipts/）

## 1.4 生命周期
- **Skill 生命周期**（威胁覆盖视角）：Registry（AST01/02）→ Install（AST04 metadata 解析）→ Execute（AST06 隔离 / AST03 权限 / AST05 外部指令）→ Update（AST07 漂移）→ Port（AST10 跨平台）→ Govern（AST09 治理）
- **项目路线图**：Q2 2026 Foundation（AST01-06）→ Q3 2026 Completion（AST07-10 + Universal Format v1.0 RC）→ Q4 2026 Launch（v1.0 + Flagship 申请）（index.md Project Status）

## 1.5 关键数据结构
- **Skill manifest 字段**：name/version/platforms/description/author(identity did:web + signing_key ed25519)/permissions(files.read|write|deny_write, network.allow 域名白名单, shell, tools)/requires/binaries+min_runtime_version/risk_tier L0-L3/scan_status(scanner+last_scanned+result)/signature/content_hash/changelog（universal-skill-format.md）
- **Admission receipt**：attempt_id/agent_id/action_type/scope/policy_version/decision(ALLOW|DENY|ESCALATE)/timestamp_ms（ast09.md）
- **Outcome receipt**：attempt_id/action_ref/terminal_state(COMMITTED|FAILED)（ast09.md）

## 1.6 测试体系
- 无传统单元测试；唯一可执行验证器：`examples/ast09-execution-receipts/check.py`（纯 std-lib：7 字段存在 + decision=DENY + attempt_id 从 preimage 重算（JCS→sha256），exit 0/1/2，`--selftest` 跑两个内嵌收据）
- tools/build_pptx.py 从 astNN.md 实时解析生成 deck；CI 触发条件 = main 触碰 ast*.md/生成器/logo（tools/README.md）

## 1.7 配置
- `_config.yml`：Jekyll remote_theme `owasp/www--site-theme@main`、collections.docs、kramdown/rouge、exclude 列表
- Gemfile：jekyll-include-cache；无运行时配置

## 1.8 权限与治理机制
- OWASP Incubator 项目（徽章），CC-BY-SA-4.0
- 治理：8 位 leaders（Ken Huang 主导 + Hammad Atta/Fabio Cerullo/Aonan Guan/Bhavya Gupta/Niv Hoffman/Iftach Orr/Akram Sheriff）；SECURITY.md 上报走 leaders
- v1 公开评审走 Google Doc（index.md）；社区贡献：issue/PR + New AST 风险条目表单 + metadata-loss-simulator
- MAINTENANCE.md 维护约定

## 1.9 外部依赖
- OWASP 站点主题（remote_theme）、python-pptx/Pillow（tools 可选）、OWASP GenAI Initiative 生态、CSA MAESTRO/AI TIPS（映射参照）

## 1.10 重要证据基线（index.md 实证）
- Snyk ToxicSkills（2026-02）：3,984 skills → 36.82% 含缺陷 / 13.4% 关键 / 76+ 恶意 payload
- ClawHavoc（2026-01/02）：1,184 恶意 skills / 12 账户 / 单 C2 IP / AMOS 投递；峰值期 Top7 下载 5 个为恶意
- ClawJacked CVE-2026-28363（CVSS 9.9）：localhost WebSocket 无速率限制暴力劫持
- Claude Code CVE-2025-59536（CVSS 8.7）：仓库级配置文件=执行层（clone 即 RCE/API key 外泄）
- Air Security（2026-06）：PoC 恶意 skill 达 26,000 agents 全扫描器放行；142,836 skills 中 12.4%（6.7M installs）依赖不可信外部指令源
- Trail of Bits（2026-06-03）：全部公开扫描器 <1 小时绕过
- SecurityScorecard（2026-03）：135,000+ OpenClaw 实例公网暴露，53,000+ 关联既往入侵

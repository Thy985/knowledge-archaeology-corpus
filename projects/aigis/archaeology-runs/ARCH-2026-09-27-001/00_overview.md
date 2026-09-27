# Aigis — Project Archaeology Package 00 · Overview

| 字段 | 值 |
|------|-----|
| run_id | ARCH-2026-09-27-001 |
| 项目 | Aigis（PyPI `pyaigis` v2.0.1） |
| 仓库 | https://github.com/killertcell428/aigis.git @ 5874cbb5 |
| 考古日期 | 2026-09-27 |
| skill 版本 | knowledge-archaeology v3.2 |
| 一句话定位 | **企业引入 Claude Code 等自主 AI agent 的"信任层"**——对工具调用加确定性护栏、保留防篡改审计日志、从实时配置生成可直接提交的 IT/安全审批包。 |

## 为什么选这个项目（Job Manifest 摘要）

与 2026-09-26 考古的 OWASP Agentic Skills Top 10（威胁分类）形成**"威胁分类 → 运行时防御实现"闭环**：AST10 回答"agent 系统有哪些攻击面"，Aigis 回答"一个 Python 实现如何在这些攻击面上做确定性防御"。且 Aigis 走**无 LLM 判定（No LLM-judge）的确定性路线**（regex + 语义相似 + 主动解码 + CaMeL 能力访问控制），是 OWASP/MCP 威胁模型的可执行落地样本。

## 核心命题（"这个项目让我们认识到了什么"）

Aigis 在 2026-08 经历了一次**战略自反**：放弃"检测数竞赛"与"通用 guardrail 货架位"，收缩为"让 AI agent 通过日本企业内部安全审查的工具"。这个项目的认知价值不在 165+ 检测 pattern，而在四个可迁移的设计选择：

1. **双层门**：Claude Code 自身权限规则（外层，独立于 hook 生效）+ Aigis PreToolUse hook（内层）——外部工具门与自身钩子门的职责边界
2. **CaMeL 分离**（arXiv 2503.18813）：UNTRUSTED 数据永不驱动控制流工具（shell:exec/agent:spawn/code:eval/mcp:tool_call），即使有 capability grant 也无条件 DENY
3. **审计与决策解耦**：签名审计失败**不改变** allow/deny（hook fail-closed 只对解析/导入/扫描/策略错误生效）；HMAC-SHA256 + 哈希链覆盖含签名全部字段
4. **审批包即产品**：`aigis trust-pack` 从**实时本地配置**（政策/hook/审计日志）生成 EN/JA 双语审批包，且合规映射字段诚实标注"自有编号，非官方条款号"

## 产出规模

- Project Layer：1 份项目地图
- Engineering Knowledge（EK Graph）：35 条 EK，links 出边 ≥1
- Generalized Knowledge：7 个 KO（R1-R4 聚合规则）
- Flow Atlas：7 类流（Control/State/Data/Evidence/Authority/Memory/Policy）
- Candidates：5 项未验证假说
- Validation：Blind Reconstruction 报告 + 判定统计 + 反例

## 证据边界声明

本包全部事实可追溯到仓库实际内容（ARCH-2026-09-27-001/repo/ 浅克隆 HEAD 5874cbb5）。运行行为观察（Guard 实测矩阵）在 snapshot_artifact.md 中记录；未深读模块（examples/、auto-improvement 细节、docs 全量）在 05_candidates 中标注。

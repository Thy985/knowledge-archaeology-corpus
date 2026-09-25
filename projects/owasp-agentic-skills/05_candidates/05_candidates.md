# 05 Candidates — 未验证假说 / 跨项目假说

> 全部为 Hypothesis 状态，不冒充知识；跨项目假说显式标注 Cross-project validation pending。

## 本仓库内待验证

- **C-01 · 签名体系落地度未证实**: Universal Format 的 ed25519 签名 + Merkle root registry 是提案（proposal.md 状态 New Project Proposal），无证据表明任何主流平台已实现；`risk_tier` 自动治理（"enables automated governance policies without per-skill review"）依赖 registry 采纳 Universal Format——采纳率未知。
  - 当前证据: [universal-skill-format.md][proposal.md:Key Deliverables]（设计意图，无落地证据）
  - 缺失证据: 任一平台的签名强制、registry 透明度日志实现

- **C-02 · Bilateral Receipt 仅在提案/示例层**: AST09 收据模式只有 examples/ast09-execution-receipts/（check.py + 2 个 JSON）与 proposals/ast-fixture-corpus/（提案态）；无主流 agent 平台集成证据。
  - 当前证据: [ast09.md][check.py][proposals/ast-fixture-corpus/proposal.md]
  - 缺失证据: 平台集成、admission receipt 的 ESCALATE 路径实现

- **C-03 · "12.4% skills 依赖不可信外部指令源"口径外推**: Air Security 数字（142,836 中 17,822）来自 2026-06 单次扫描，定义"不可信外部指令源"的判据未在仓库中完整给出；对 2026-09 生态现状外推不确定。
  - 当前证据: [index.md:Incident 2026-06-22~24]
  - 缺失证据: 复测、判据定义、时间衰减

- **C-04 · SkillSpector 等扫描器的实际绕过弹性**: 仓库推荐语义扫描（SkillSpector 0-100 风险分）作为 AST08 缓解，但 Trail of Bits 证实所有公开扫描器 <1 小时绕过——语义扫描对"无限语言变异性"的抵抗力无独立基准。
  - 当前证据: [skill-scanner-integration.md][index.md:Incident 2026-06-03]
  - 缺失证据: 对 LLM-judge 提示注入的鲁棒性测量

## 跨项目假说（Cross-project validation pending）

- **C-05 · 身份文件 = 行为密钥（跨 agent 项目）**: AST10 主张 SOUL.md/MEMORY.md 是 agent 行为身份载体、需 deny_write 默认保护。此假说应与 dsh-memory-evolve（记忆安全）、hermes-agent、openclaw 考古交叉验证——"agent 身份是行为性的而非凭据性的"是否为通用模式。
  - 关联: KO-05 / EK-26 / EK-27；corpus: dsh-memory-evolve（已归档）、openclaw（已归档）
  - 验证路径: 检查记忆/身份文件写入权限在各 agent 实现的默认策略

- **C-06 · 指令-数据边界是 Agent 安全通律**: AST10 的"文本被当指令执行"（EK-04）与 MCP 考古的 allowed-tools 审批门、agentic-attack 卡的 prompt injection 事件群指向同一原理。假设："任何把不可信文本升格为指令通道的 agent 设计都存在系统性越权路径"。
  - 关联: KO-01/KO-03；corpus: mcp（09-25）
  - 验证路径: 跨 5+ agent 项目检查指令层级实现

- **C-07 · "防御必须默认可绕过"的层级观**: KO-04 主张签名+扫描+治理互补、无一单点可独立成立。此为 AST10 的强主张（多事件支撑），但作为通用安全原则需跨安全框架（OWASP ASVS/ASI01-10/AGT）对照。
  - 关联: KO-04；corpus: agent-governance-toolkit（AGT 映射 ASI）
  - 验证路径: AGT 参考架构的缓解映射是否呈现同样的"多层互补"结构

- **C-08 · B1-B4 模型外推至非 coding agent**: B1-B4 覆盖 Developer↔Agent↔Repo↔CI/CD↔Prod，属 coding-agent 管线。其对通用 agent（个人助理/浏览器 agent/嵌入式）的适用性（信任边界如何重定义）无证据。
  - 关联: KO-06（DOWNGRADED 关联）；corpus: computer-browser-use（候选）
  - 验证路径: 对 browser-use 类 agent 应用 B1-B4 检验边界映射

---

## Reconciliation 增补（ARCH-2026-09-26-001，独立验证 M-01）

- **C-09 · check.py attempt_id preimage 不含 decision 的篡改盲点**: 盲重建发现 `ATTEMPT_ID_PREIMAGE_FIELDS` 仅含 agent_id/action_type/scope/policy_version/timestamp_ms 五字段，**decision 不在 preimage 内**——篡改 decision 字段不破坏 attempt_id 重算；tampered 样例仅篡改 scope，未覆盖该路径。收据"防篡改"范围需在 AST09 后续版本中明确（将 decision 纳入 preimage 或对 decision 单独签名）。
  - 当前证据: [check.py:ATTEMPT_ID_PREIMAGE_FIELDS][deny-admission-receipt.tampered.json]（仅 scope 篡改样例）
  - 缺失证据: decision 篡改路径的收据样例/测试

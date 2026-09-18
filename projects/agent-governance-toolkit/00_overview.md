# 00 — Overview

## 定位

**Agent Governance Toolkit（AGT）**——微软开源的 Agent 运行时治理全栈（MIT，v5.0.0，public preview）。核心命题："**Ship agents to production without losing sleep**"——把 OS 内核/服务网格/SRE 的成熟治理模式整体移植到 agent 执行层。

**对比前 3 个防火墙品类项目**（四选实测第 4 点）：
- Aigis（09-13）：轻量单包，确定性 guardrail + 审计链
- Guardian（09-14）：sidecar，7 项确定性检查
- AI Protector（09-15）：proxy + agent 双层，7 层加权 + A2 反混淆
- **AGT（本轮）**：**全栈 OS 级**——policy enforcement（ACS Rust 核心）+ 身份（Ed25519/DID）+ 执行环 + SRE + Compliance + Marketplace + RL 治理 + 9 语言 SDK

## 关键数字（S7 项目自述 + S4 本地实测分离）

| 指标 | 值 | 证据 |
|---|---|---|
| 测试函数（全仓） | 13,834 | 代码扫描 S3 |
| conformance tests | 992（10 specs） | README S7 |
| ADR | 29 | docs/adr/ S3 |
| 策略评估延迟 | <0.1ms（ACS Rust）/ <5ms（PolicyEngine MAX_EVAL_MS） | README + policy.py:689 S7/S3 |
| 本地实测（本轮） | **368 passed** / 15 skipped / 4 env-failed | S4（agent-mesh 304 + hypervisor 64） |
| 语言 SDK | 9（Python/TS/Rust/Go/.NET + 4 CLI 适配） | 目录 S3 |
| OWASP Agentic Top10 | 7 Full + 3 Partial | badge S7 |

## 核心架构（五组件）

```
Agent Action → POLICY CHECK → Allow/Deny（<0.1ms，确定性，无 LLM 在决策路径）
  ├─ Agent OS    ：Policy Engine（ACS/确定性）+ Capability + Governance Gate + Decision BOM
  ├─ AgentMesh   ：零信任身份（Ed25519/SPIFFE）+ Trust Scoring（0-1000）+ 委托链
  ├─ Agent Runtime：执行环（Ring 0-3）+ 资源限制 + kill switch + saga
  ├─ Agent SRE   ：SLO/熔断/混沌/渐进式交付
  └─ Agent Hypervisor：执行审计 + Delta 引擎 + Merkle 链
```

## Top 5 发现

1. **决策确定性是"结构性"而非"概率性"**：策略检查在应用层拦截（确定性代码），不在 prompt 里求模型守规矩——"Actions the AGT kernel denies are structurally impossible"（README）。无 LLM 在策略路径（SECURITY.md 威胁模型）。
2. **四执行环取代 RBAC 作为运行时主权限模型**（ADR-0002）：RBAC 保留为人管/合规/scope，运行时权限用"环"（Ring 0-3）映射 blast radius——比静态角色更贴近运行时风险。
3. **fail-closed 无开关**（ADR-0013）：策略评估任何错误 → 立即 deny + error 审计，无 fail-open 配置项——错误是可见的（误拒快速暴露），不能静默绕过。
4. **Trust Ceiling Propagation**（ADR-0016）：子代理信任分 ≤ 父代理（min 单调收敛），委托链不能提权——类比 capability 系统"进程不能授予超过自己持有的能力"。
5. **审计与可证明性三层**：Merkle 哈希链（防篡改，ADR-0017）→ Decision BOM（可重构而非预构建，ADR-0018，非侵入）→ TRACE 信任记录（agentrust-trace，ADR-0032）。

## 治理证据

- SECURITY.md：威胁模型（Policy bypass / Identity spoofing / Audit tampering / Budget evasion / Tool-call injection / Supply chain / Privilege escalation via delegation）
- CHARTER.md / GOVERNANCE.md / MAINTAINERS.md / ANTITRUST.md：项目治理
- 40+ GitHub workflows（ci/codeql/scorecard/sbom/redteam-benchmark/weekly-security-audit）
- OpenSSF Scorecard + Best Practices badges

## 边界

- **Public Preview**：GA 前可能有 breaking changes（README IMPORTANT）
- agent-os-kernel / agentmesh-platform 已 deprecated，v5 合并到 agent-governance-toolkit-core（包整合中，docs/package-consolidation）
- policy-engine/core 是 deprecation shim，真实运行时在 crates.io `agent_control_spec`（基于 agent-hooks 契约）
- 4 个本地测试失败 = 缺可选依赖 agentrust-trace（TRACE sink），非产品缺陷

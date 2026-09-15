# 01 — Project Layer（项目地图）

## 是什么
微软开源的 Agent 运行时治理全栈：策略执行 + 零信任身份 + 沙箱执行环 + SRE + 合规 + 市场 + RL 训练治理。MIT，v5.0.0，public preview。

## 怎么运行
1. `pip install "agent-governance-toolkit[full]"`
2. `from agentmesh.governance import govern; safe_tool = govern(my_tool, policy="policy.yaml")`
3. 每次调用：ring check → policy evaluate → approval（如需）→ audit → 执行或拒绝

## 生命周期（agent 动作）
```
agent 动作发起
  → GovernedCallable.__call__（govern.py:231）
  → RingEnforcer（ring 拒绝不达 policy）
  → PolicyEngine.evaluate（agent_did + context，stage 过滤）
  → 冲突解决（conflict_strategy）
  → require_approval → ApprovalCoordinator（TTL 300s 默认）
  → AuditLog（Merkle 链）
  → on_deny 或 raise GovernanceDenied / 执行原函数
```

## 模块与入口

| 模块 | 入口 | 职责 |
|---|---|---|
| agent-mesh | src/agentmesh/governance/govern.py:701 | 两行集成 API + 策略引擎 + 审批 + 审计 |
| agent-mesh identity | src/agentmesh/identity/ | Ed25519/SPIFFE/Entra/attestation/委托链 |
| agent-os | src/agent_os/ | 政策执行 kernel（MCP gateway/content governance/egress） |
| agent-hypervisor | src/hypervisor/rings/ | 四执行环 + breach detector |
| agent-runtime | src/agent_runtime/ | deploy/sandbox/kill switch |
| policy-engine | core/src/lib.rs | ACS Rust 决策核心（v5 为 deprecation shim → agent_control_spec） |

## 核心数据结构
- PolicyDecision：action（allow/deny/warn/require_approval/log）+ matched_rule + reason + approvers
- Rule：condition DSL + action + stage + priority
- AuditEntry：previous_hash + entry_hash（SHA-256 链）
- TrustScore：0-1000（compliance .35 / task_success .25 / behavior .25 / identity .15）
- ExecutionRing：RING_0_ROOT~RING_3_SANDBOX + ResourceConstraints

## 状态
- 策略注册表 dict + 速率限制 dict + Merkle 链指针 + TrustStore 持久化

## 测试体系
- 核心治理 134 + govern/identity 170 + ring 64 = **本地 368 passed**
- README：992 conformance / 29 ADR / 13,834 test fn（S7）

## 配置
- policy YAML（governance.toolkit/v1 + rules）
- conflict_strategy（默认 priority_first_match；govern 默认 deny_overrides）
- AGT_TRUST_CEILING（容器部署信任上限）

## 权限与治理机制
四执行环 / fail-closed / trust ceiling / Merkle 审计 / Decision BOM / MCP Gateway / extends 合并

## 外部依赖
agentrust-trace（可选，TRACE sink）、OPA（rego 可选）、OTel、hypervisor 内部包

## 扩展机制
- 9 语言 SDK（govern() 同一契约）
- framework adapters（LangChain/CrewAI/OpenAI/ADK/smolagents）
- policy backends：YAML + Rego（OPA）+ Cedar（pluggable，ADR-0015）
- integrations/ 目录（agentmesh-integrations）

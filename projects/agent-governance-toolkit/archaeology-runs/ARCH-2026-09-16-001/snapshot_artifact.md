# Repository Snapshot Artifact — ARCH-2026-09-16-001

```yaml
repository: https://github.com/microsoft/agent-governance-toolkit.git
commit_sha: b1a9f733782a282fe191566f80f2ab414791ce4f
branch: main
repository_version: 5.0.0
analysis_timestamp: 2026-09-16T02:00+08:00
clone_type: shallow (--depth 1)
repo_size: 84 MB
file_count: 4771
python_file_count: 1907
test_fn_count: 13834（跨全部包）
skill_version: knowledge-archaeology v3.2
```

## 项目基础地图

```
agent-governance-toolkit/
├── policy-engine/                  # ACS（Agent Control Specification）Rust 决策核心
│   ├── core/src/{lib.rs,manifest_yaml.rs,artifact_validation.rs,identity.rs,telemetry_sinks.rs}
│   ├── policy/                     # 策略语言
│   ├── sdk/                        # Python SDK（maturin）
│   ├── spec/                       # ACS 规范
│   └── docs/{security-model.md,stateless-runtime.md}
├── agent-governance-python/        # Python 包家族（核心）
│   ├── agent-mesh/                 # AgentMesh：identity/governance/trust（src/agentmesh/）
│   ├── agent-os/                   # Agent OS：policy enforcement kernel（src/agent_os/，125 py，44K 行）
│   ├── agent-runtime/              # 执行环/kill switch/saga
│   ├── agent-hypervisor/           # Execution Rings（RING_0_ROOT~RING_3_SANDBOX）
│   ├── agent-sre/                  # SLO/熔断/混沌
│   ├── agent-compliance/           # EU AI Act/HIPAA/SOC2 映射
│   ├── agent-marketplace/          # 插件签名
│   ├── agent-lightning/            # RL 训练治理
│   ├── agent-governance-toolkit-core/  # v5 合并后核心发行包（dep stub）
│   └── agt-policies/               # v4→v5 迁移命令
├── agent-governance-{dotnet,golang,rust,typescript}/  # 独立语言 SDK
├── agent-governance-{claude-code,copilot-cli,opencode,antigravity-cli}/  # CLI 适配
├── benchmarks/prompt-injection/    # 280 行 smoke corpus（110 attack + 170 benign）
├── docs/                           # ARCHITECTURE + 29 ADR + specs + security
├── tests/                          # e2e_python / redteam / unit / smoke
└── .github/workflows/              # 40+ workflows（ci/codeql/scorecard/sbom/redteam-benchmark）
```

## 主要语言
Python（主）· Rust（policy-engine core）· TypeScript · .NET · Go（多语言 SDK）

## 主要运行入口
- `pip install agent-governance-toolkit[full]` → `from agentmesh.governance import govern`（govern.py:701）
- Claude Code：`/plugin install agt-governance@agent-governance-toolkit`

## 核心模块
- **govern()**（agentmesh/governance/govern.py:701）：两行集成——包装任意 callable，每次调用做 policy check + audit + enforce
- **PolicyEngine**（agentmesh/governance/policy.py:689）：YAML/JSON 策略引擎，conflict_strategy 可配（deny_overrides/allow_overrides/priority_first_match/most_specific_wins）
- **GovernedCallable**（govern.py:174）：核心原语，ring enforcement → policy evaluate → approval → audit
- **MerkleAuditChain**（audit.py:268）：SHA-256 哈希链审计防篡改
- **DecisionBOMBuilder**（decision_bom.py）：可重构决策物料清单（审计/信任分/策略/OTel 信号四源）
- **MCP Security Gateway**（agent_os/mcp_gateway.py）：MCP 工具调用治理（OWASP ASI02）
- **RingEnforcer**（hypervisor/rings/enforcer.py）：四执行环资源约束
- **ACS**（policy-engine/core）：Rust 无状态策略决策运行时（stateless/deterministic/fail-closed）
- **identity/**（agentmesh/identity/）：Ed25519/SPIFFE/Entra/attestation/delegation

## 核心数据结构
- `PolicyDecision`（action: allow/deny/warn/require_approval/log + matched_rule + reason + approvers）
- `Rule`（condition DSL + action + stage: pre_input/pre_tool/post_tool/pre_output + priority）
- `AuditEntry`（previous_hash + entry_hash，SHA-256 链）
- `TrustScore`（0-1000，四维加权：compliance 0.35 + task_success 0.25 + behavior 0.25 + identity 0.15）
- `ExecutionRing`（RING_0_ROOT / RING_1_PRIVILEGED / RING_2_STANDARD / RING_3_SANDBOX）
- `ResourceConstraints`（network/filesystem/subprocess/max_concurrent_tools）

## 核心状态
- 策略注册表（_policies: dict[name → Policy]）
- 速率限制（_rate_limits: (agent_did, policy, rule) → {count, reset_at}）
- Merkle 审计链状态（previous_hash 指针）
- 信任分（TrustStore，按 agent 持久化）

## 主要测试体系
- agent-mesh：147 测试文件，核心治理测试**本地实测 134 passed**（conflict/decision_bom/approval/policy_composition/schema/audit）+ **170 passed**（govern/identity/attestation/trust store，4 failed=缺 agentrust-trace 可选依赖）
- agent-hypervisor：ring 测试 **64 passed**
- 本地合计实测 **368 passed / 15 skipped / 4 env-failed**
- README 自述：992 conformance tests（10 specs）+ 29 ADR；全仓 13,834 test 函数（S7 自述，未全量本地复现）
- benchmarks/prompt-injection：280 行 smoke corpus（110 attack/170 benign），rust scorer 基线

## 主要配置
- pyproject：`[full]` extra 含 core
- policy YAML：apiVersion governance.toolkit/v1 + rules（condition DSL + action）
- AGT_TRUST_CEILING 环境变量（容器部署信任上限）
- conflict_strategy 默认 priority_first_match（v1 行为）/ govern() 默认 deny_overrides

## 权限 / Policy / Governance 机制
- 四执行环（ADR-0002）：RBAC 保留为人管/合规/scope，运行时权限主用 rings
- fail-closed（ADR-0013）：评估错误一律 deny，无 fail-open 开关
- trust ceiling propagation（ADR-0016）：子代理信任 ≤ 父代理，防提权
- Merkle 审计链（ADR-0017）
- reconstructible decision BOM（ADR-0018）
- policy extends 递归合并（additive-only，父 deny 不可被子级削弱——v4 已 superseded，v5 用 ACS extends）
- MCP Gateway：allowlist/denylist + 参数净化 + 速率限制 + HITL 审批 + 响应扫描

## 主要外部依赖
- agentrust-trace（TRACE 信任记录，可选，测试 4 failed 因缺此包）
- OPA（rego 评估，可选）
- OTel（可观测）
- hypervisor（agent-hypervisor 内部依赖）

## 未覆盖（本 Snapshot 边界）
- 9 语言 SDK 的逐一细节（dotnet/golang/rust/typescript/claude-code 等）
- 全仓 13,834 测试未全量运行（本地复现 368）
- benchmarks 未跑完整（smoke 280 行）
- docs/compliance 全量映射未逐条核对

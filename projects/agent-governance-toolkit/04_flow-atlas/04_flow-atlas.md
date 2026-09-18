# 04 — Flow Atlas（七类流）

> 每条 Edge 可回溯 symbol/file/condition/state transition。从真实代码导出，非架构想象。

## 1. Control Flow（控制流）——治理决策管线

```
govern(fn, policy, ...)                                    govern.py:701
  → GovernedCallable.__init__                              govern.py:174
  │   PolicyEngine(conflict_strategy)                      policy.py:704
  │   load_yaml_file / load_yaml / load_policy             policy.py:750/737/728
  │   load_rego（可选，需 OPA）                            policy.py:884
  │   _policy_bundle_hash = sha256(policy_bytes)           govern.py:213-230
  │   TRACEAuditSink（config.trace 时）                    govern.py:238
  │   RingEnforcer（config.ring 时）+ shared breach detector govern.py:247-252
  → __call__                                               govern.py:231
  │   _build_context(args, kwargs)                         govern.py:~240
  │   _check_ring() → ring_denial? → on_deny / raise       govern.py:243-253
  │   _engine.evaluate(agent_did, context)                 policy.py:1036
  │       extract_protocol_facets(context)                 protocol_facets.py
  │       [p for p in policies if p.applies_to(agent)]     policy.py:1062
  │       rule.stage != stage → skip                       policy.py:1074
  │       rule.evaluate(context) → candidates              policy.py:1076-1086
  │       _resolver.resolve(candidates)                    conflict_resolution.py
  │       rate_limit check on winning rule                 policy.py:1112-1130
  │   decision.action == require_approval → _handle_approval govern.py:~260
  │   AuditLog.log(...)                                    audit.py:465
  │   decision.action == deny → on_deny / raise GovernanceDenied govern.py:~290
  │   [Reconciliation 补边] decision.allowed → advisory 层      govern.py:310-323
  │       advisory_result.action == block → on_deny / raise（deterministic:false）
  │       advisory 失败默认 allow（非确定性不参与放行）
  │   else → fn(*args, **kwargs)
```

> **Reconciliation 修正**：Control Flow 原版缺 advisory 边（独立验证 MISSING-1）。advisory 在确定性 allow 之后、执行之前，只能收紧（block），失败默认 allow。

## 2. State Flow（状态流）

```
策略注册表   _policies: dict[str, Policy]                  policy.py:728
  加载时置入，同名替换（load_policy）
速率限制     _rate_limits: (agent_did, policy, rule) → {count, reset_at}
  每次 winning rule 命中时 count+1，window 过期 reset        policy.py:1112-1130
审计链       AuditEntry.previous_hash → 链式增长              audit.py:268
  每次 log 追加，previous_hash = 上条 entry_hash
信任分       TrustStore（按 agent）                         identity/risk.py
  行为驱动升降，trust ceiling 为上界（min 单调）            ADR-0016
执行环       RING_0~3 + breach detector（(agent, session) 共享计数）
  违规计数 → 熔断（SRE circuit breaker）                   rings/breach_detector.py
```

## 3. Data Flow（数据流）——一次策略评估的数据

```
context（kwargs 提取）→ extract_protocol_facets 补充 sql.*/k8s.*
  → rule.evaluate(context)：condition DSL（field-vs-literal 比较）
  → CandidateDecision(action, priority, scope, policy_name, rule_name)
  → PolicyConflictResolver.resolve → winning PolicyDecision
  → [Reconciliation 补边] 无 YAML 匹配 → authority resolver（DelegationInfo/TrustInfo/ActionRequest）信任窄化（policy.py:1145+）
  → AuditEntry.data（rule/reason）→ Merkle 链 + 树
  → approval 时：approval record（policy_version 绑定）
  → TRACE 时：TrustRecord（agentrust_trace 签名）
```

## 4. Evidence Flow（证据流）

```
源码 → 测试（134+170+64 本地实测）→ conformance tests（992 声称 S7）
  → benchmarks/prompt-injection（280 smoke：110 attack/170 benign）
  → artifacts/ 仅 metadata（不含 raw prompt）
  → SECURITY.md 威胁模型 + 缓解映射
  → ADR-0001~0032（设计决策证据）
```

## 5. Authority Flow（权威流）

```
Policy Source（策略定义，extends 继承 additive-only）        policy.py:750
  → PolicyEngine（评估权威）                               policy.py:689
  → ConflictResolver（多策略裁决权威）                      conflict_resolution.py
  → approval（approvers 清单，TTL 300s，policy_version 绑定） govern.py + ADR-0030
  → ring 约束（运行时执行权威上限）                         hypervisor/rings/enforcer.py
  → trust ceiling（委托链权威上限）                         ADR-0016
  → 外部 backends：OPA（Rego）/ Cedar（pluggable）          ADR-0015
```

## 6. Memory Flow（记忆流）

```
（本仓：agent 记忆不落库；治理决策记忆 = 审计链 + Decision BOM）
AuditLog（append-only Merkle 链）                          audit.py:465
  → DecisionBOMBuilder 事后从 Audit/Trust/Policy/Trace 四源重构  decision_bom.py
  → TRACE trust records（互操作格式，agentrust_trace）      trace_sink.py
```

## 7. Policy Flow（治理闭环）

```
Policy Authoring（YAML extends / Rego / ACS manifest）
  → Policy Distribution（registry，ADR-0029）
  → Enforcement（PolicyEngine / ACS intervention points）
  → Audit（Merkle 链）→ 事件（OTel/TRACE）
  → Observability → Policy Revision（ADR 化迭代）
  → Future Decisions（Decision BOM 可回溯先例）
```

## Flow→KO 交叉校验
- KO-01（分层防御）对应 Control Flow 的 ring→policy 顺序（govern.py:243-253）✓
- KO-02（确定性）对应 Data Flow 无 LLM + ACS stateless/deterministic ✓
- KO-03（trust ceiling）对应 Authority Flow 的 min 单调 ✓
- KO-04（审计可证明）对应 Memory Flow 的 Merkle + BOM + TRACE ✓
- KO-05（fail-closed）对应 Data Flow 的异常→deny 路径 ✓

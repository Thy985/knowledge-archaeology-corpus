# 04 Flow Atlas — agent-aegis（ARCH-2026-10-22-001）

> 七类流从真实代码导出；Edge 标注可回溯（symbol / file / condition / state transition）。

## Control Flow（控制流）
```
接入: import aegis（惰性 ~30ms）→ AEGIS_INSTRUMENT=1 或 auto_instrument() 或 Aegis.init(config)
  → Aegis.init：_discover_config（config 文件发现）→ _activate
    → _setup_audit → _setup_guardrails → _setup_cost_tracker → _setup_integrations（_patch_openai/_patch_anthropic）
判定: ActionClaim（agent 声明）→ assess（独立 assessor）→ policy.evaluate（LRU 缓存）
  → selection audit（可选）→ ClaimPolicyDecision（复合解释）
装卸: _deactivate → _teardown_integrations → _teardown_audit（可逆）
```
- **Edge**：`_is_cacheable` 返回 False 时 evaluate 走 `_evaluate_uncached`（policy.py:213-228）——缓存路径与未缓存路径分支条件 = "任一条件规则 regex 匹配当前 action"

## State Flow（状态流）
- Aegis singleton：`uninitialized → active（init）→ shutdown`（init.py:43-131）
- ClaimVerdict 状态机：`pending → approve | escalate | block`（action_claim.py:124-134）
- 集成补丁状态：`unpatched → patched（_patch_*）→ unpatched（_teardown_integrations）`
- Policy 缓存状态：LRU OrderedDict（hit→move_to_end；miss→插入；满→popitem(last=False)）

## Data Flow（数据流）
- 配置：`policy.example.yaml` → `Policy.from_yaml` → Policy 对象（rules+defaults+scope+version）
- 声明：ActionClaim.DeclaredFields（untrusted）→ assessor → AssessedFields（Aegis 计算）→ justification_gap
- 审计：GuardrailEngine 事件 → AGEF 事件（canonical JSON）→ MerkleAuditTree 叶哈希 → root_hash/proof
- MCP：ToolCallRecord（不可变）→ EscalationDetector 滑动窗口 → EscalationFinding（source→sink 时序）

## Evidence Flow（证据流）
- conformance 测试（test_agp_level1）钉住 fail-safe（destructive 无规则 → BLOCK/DENY）
- test_agef_schema 钉住 AGEF 事件 schema
- MerkleProof 支持审计证据链验证（_hash_leaf/_hash_pair/canonical JSON——merkle_audit.py:57-77）

## Authority Flow（权威流）
```
Agent 声明意图（DeclaredFields）→ 系统独立评估（AssessedFields——"Agent MUST NOT author these"）
  → 委托链单调校验（trust_level 非递增）
  → ClaimPolicyDecision（policy verdict + claim verdict + monotone + justification gap）
  → 执行或升级（approve/escalate/block）
身份: AgentIdentity（capabilities frozenset + glob）→ capability_matches
零信任: AgentCredential → VerificationResult → SessionToken → ZeroTrustPolicy（每请求验证）
```
- **Edge**：`validate_monotone_constraint`：len(chain)<2 → True（平凡真）；否则全对 `chain[i].trust_level <= chain[i-1]`（action_claim.py:109-121）

## Memory Flow（记忆流）
- 会话状态：SessionToken（zero_trust_agent.py:134）——每请求验证的会话身份
- 审计持久化：AuditLogger（sqlite backend，init.py:195-215；失败 warning 降级）
- Policy 缓存：OrderedDict LRU（进程内记忆）
- 审计树：MerkleAuditTree root_hash 状态（merkle_audit.py:171-187）

## Policy Flow（治理闭环）
```
决策（GOVERNANCE.md：ADR 以 adr label issue 承载 / breaking 需 Discussion+CHANGELOG）
  → 策略（policy.example.yaml + Policy.from_yaml + autopolicy LLM 生成 _validate_policy_dict）
  → 执行（ClaimPolicy.evaluate 四步 + GuardrailEngine + MCP 检测器）
  → 记录（AGEF 事件 + Merkle 树）
  → 未来决策（CHANGELOG 演进；AGEF/AGP Draft 标准演进）
```
- **Edge**：标准自述（AGEF/AGP Draft）是治理闭环的"契约面"——conformance 自测保证实现一致，但生态采纳是未来决策（C 候选）

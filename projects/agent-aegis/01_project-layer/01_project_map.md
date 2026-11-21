# 01 Project Layer — agent-aegis（ARCH-2026-10-22-001）

## 项目地图（全部可回溯浅克隆 HEAD 134580e）
- **语言/运行时**: Python 3.11+；包名 `agent-aegis`；核心依赖 `pyyaml>=6.0`；12 optional extras（playwright/langchain/crewai/google-adk/pydantic-ai/openai-agents/anthropic/httpx/mcp/starlette+uvicorn/redis/asyncpg）
- **入口**: `aegis.init()` / `aegis.auto_instrument()` / `AEGIS_INSTRUMENT=1`；`src/aegis/__init__.py`（急切加载仅 Aegis/init/shutdown/get/auto_instrument，其余惰性）
- **生命周期**: Aegis singleton：`_activate`（audit→guardrails→cost→integrations）/ `_deactivate`（teardown integrations+audit）；集成补丁 `_patch_openai/_patch_anthropic` 可逆卸载
- **核心模块**:
  - `core/policy.py`: Policy/PolicyRule/PolicyDecision（evaluate+LRU 缓存 256+from_yaml/from_yaml_files/merge/clear_cache）
  - `core/action_claim.py`: ActionClaim 三组字段 + ClaimVerdict（approve/escalate/block/pending）+ ImpactVector（magnitude/distance/asymmetric_gap）+ validate_monotone_constraint
  - `core/claim_policy.py`: ClaimPolicy.evaluate 四步 + composite explanation（policy blocked / monotone / justification gap / congruence / selection audit）
  - `core/merkle_audit.py`: MerkleAuditTree/MerkleLeaf/MerkleProof/BatchProofResult（SHA256 规范 JSON）
  - `core/agent_identity.py`: AgentIdentity（capabilities frozenset + glob 匹配）+ DelegationEvent
  - `core/autopolicy_llm.py`: OpenAIPolicyGenerator/AnthropicPolicyGenerator（自然语言→policy dict）
  - `core/mcp_*.py`: mcp_shadow（tool 影子攻击 4 类检测）/ mcp_escalation（source→sink 升级链滑动窗口）/ mcp_rate_limiter / mcp_response_scanner / mcp_sbom / mcp_vuln_db / mcp_stdlib_guard / mcp_threat_intel / mcp_transport / mcp_manifest
  - `core/zero_trust_agent.py`: AgentCredential/VerificationResult/SessionToken/ZeroTrustPolicy
  - `guardrails/engine.py`: GuardrailEngine（check/check_and_transform/acheck/from_pack）
  - `instrument/`: patch_langchain/patch_litellm/patch_google_adk/patch_google_genai/patch_pydantic_ai/patch_llamaindex/patch_instructor/patch_dspy/patch_crewai/patch_openai_agents
  - `adapters/`: anthropic/crewai/langchain/mcp/openai_agents/playwright/httpx + base
  - `cli/`: check/scan/score/plan/diff/monitor/compliance/test_cmd/colors
  - `contrib/pydantic_ai.py`; 独立包 `packages/{crewai,langchain,openai-agents}-aegis`
- **核心数据结构**: Policy/PolicyRule/PolicyDecision；Action/ActionClaim/ClaimVerdict/ImpactVector；MerkleAuditTree/MerkleProof；AegisConfig；AgentIdentity；ZeroTrustPolicy
- **状态**: Aegis singleton（init→active→shutdown）；instrument patch 状态（_state.py）；Policy 缓存 OrderedDict LRU
- **测试体系**: `tests/`（cli/* + conformance/*：test_agef_schema / test_agp_level1）+ `packages/*/tests`；`core/policy_test_suite.py`+`policy_test_runner.py`（策略即测试）；README 徽章 6400+ tests / coverage 92%；benchmarks/
- **配置**: `policy.example.yaml`（version "1"：defaults + rules：match type→risk_level/approval + conditions 如 bulk>100 升级）；`policy.schema.json`；`aegis-server.example.yaml`；server.json/smithery.yaml（MCP server）；Dockerfile；action.yml（GitHub Action）
- **权限与治理机制**: Policy evaluate（approval: auto/approve/escalate）→ ActionClaim 审批链（单调约束）→ RBAC / sandbox_policy / circuit_breaker / cascade_guard / cross_tool_privacy / data_isolation / taint / cost_policy+budget / merkle audit / otel_export
- **开放标准（自述）**: AGEF v1（事件格式 schema.json，Draft 2026-03-24）· AGP v1（protocol.md，Draft 2026-03-24）——conformance 测试自我钉住
- **治理元机制**: GOVERNANCE.md（ADR 以 `adr` label issue 承载；breaking 需 Discussion+CHANGELOG）；SECURITY.md；CODE_OF_CONDUCT.md
- **外部依赖**: 仅 pyyaml 必需；框架适配/存储/可观测全 optional（redis/postgres/otel 等）
- **错误与失败路径**: init 失败 warning 不阻断（audit sqlite 异常 warning）；guardrail on_block（block/warn）；fail-safe 默认（AGP Level1：destructive action 无规则匹配 → BLOCK/DENY）；_is_cacheable 防遮蔽

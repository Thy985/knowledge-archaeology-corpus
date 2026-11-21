# 00 Overview — agent-aegis（ARCH-2026-10-22-001）

## 定位
**agent-aegis（Acacian/aegis，v1.0.0，Python/MIT/★18）**：AI agent 的治理层运行时——"The governance layer for AI agents. One API, 12 frameworks, every governance primitive."（README 首屏）。`pip install agent-aegis` → `aegis.auto_instrument()` → 12 框架受治理，零代码改动；核心依赖仅 pyyaml，其余框架适配全部 optional。
本周定位：Harness/治理 周 C 层（权限与策略治理）+ D 层（运行时防御）的**一体化框架候选**（KnowlegeMap 卡 [cand]agent-aegis-governance-2026.md），周五 guardrail-策略引擎对比池成员（Aigis·MS AGT·agent-aegis）。

## 四条认知主线（本文档结论，证据见 02/04）
1. **声明 vs 评估分离**：ActionClaim 三组字段（DeclaredFields 代理自述 / AssessedFields Aegis 计算，Agent 无权 author / ChainFields 委托链）——justification_gap = 自述影响与独立计算影响的差距，是审批的核心判据
2. **策略即数据 + 可缓存但防遮蔽**：YAML → Policy（LRU 缓存 key=(type,target,agent_id)）；`_is_cacheable` 对含条件规则的 action 禁用缓存——防"无条件缓存命中遮蔽条件规则"
3. **信任委托单调约束**：delegation chain 的 trust_level 必须非递增（`validate_monotone_constraint`）；违反进 composite explanation
4. **零代码接入 vs 完整 API 并存**：`AEGIS_INSTRUMENT=1` / `auto_instrument` / `patch_<framework>`（10 框架 + 2 raw API）与 `Aegis.init(config)` 显式生命周期并存；惰性 import 保证 `import aegis` ~30ms

## 关键文件锚（考古唯一事实源：浅克隆 HEAD 134580e）
- `src/aegis/init.py:43-389`（Aegis singleton：_activate→_setup_audit/_setup_guardrails/_setup_cost_tracker/_setup_integrations；_patch_openai/_patch_anthropic）
- `src/aegis/instrument/__init__.py:128,349-355`（auto_instrument + AEGIS_INSTRUMENT 环境变量自动接入）
- `src/aegis/core/policy.py:169-260`（evaluate LRU 缓存 / _is_cacheable / merge / from_yaml）
- `src/aegis/core/action_claim.py:109-236`（validate_monotone_constraint / ClaimVerdict / DeclaredFields·AssessedFields·ChainFields / ActionClaim）
- `src/aegis/core/claim_policy.py:152-220`（evaluate 四步：assess→policy→selection audit→composite decision + 解释组装）
- `src/aegis/core/merkle_audit.py:57-187`（SHA256 规范 JSON + Merkle 树 + 批证明）
- `src/aegis/core/agent_identity.py:96-122`（AgentIdentity frozen dataclass + capabilities frozenset + capability_matches glob）
- `src/aegis/core/autopolicy_llm.py:143-267`（OpenAI/Anthropic 策略生成器 + _validate_policy_dict）
- `src/aegis/core/mcp_shadow.py` / `mcp_escalation.py`（MCP tool shadowing 检测 / 跨 server 升级链检测）
- `src/aegis/core/zero_trust_agent.py:66-155`（AgentCredential / VerificationResult / SessionToken / ZeroTrustPolicy）
- `tests/conformance/test_agp_level1.py`（AGP Level1 conformance：fail-safe BLOCK/DENY）
- `specs/agef/v1/schema.json` + `specs/agp/v1/protocol.md`（自述标准，Status: Draft 2026-03-24）

## 跨项目连接
- **aigis / guardian / ai-protector**（corpus 既有）：同品类（运行时防御）——aegis 多了策略引擎+标准支柱，contrast 锚
- **agent-governance-toolkit**（corpus）：无状态策略引擎 ↔ aegis 的声明式策略 + approval gates（同问题两种实现）
- **Archon**（10-21，corpus PR #50）：控制面治理在**进程内 server**（opt-in gate/401/fail-closed 端点）vs aegis 治理在**进程外中间件/插桩层**（drop-in）——同一目标两种位置
- **deepseek-harness**：sandbox/approval seam ↔ aegis ActionClaim/approval gates
- **事件集群（breach cluster）**：Archon CVE（控制面配置缺失·外因性）vs aegis（运行时注入/PII 防御·中间件侧）——为周五事件精读补防御侧源码证据

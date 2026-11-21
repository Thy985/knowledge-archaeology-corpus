# 02 Engineering Knowledge — agent-aegis（ARCH-2026-10-22-001）

> EK Graph：每条 EK 带证据（文件:行/锚）+ links（6 类边）。前缀 G（Governance），避免与 Archon B / Box b 混用。

## EK-G01 · 惰性 import 保证零成本接入（~30ms）
- **机制**: `__init__.py` 急切加载仅 Aegis/init/shutdown/get/auto_instrument，其余 `_LAZY_IMPORTS` dict 惰性——"import aegis is fast (~30ms) regardless of which features you use"
- **证据**: src/aegis/__init__.py:1-35
- **links**: mechanism↔G02（零代码接入族）；dependency→G04（instrument 需要）

## EK-G02 · 三种接入梯度（env 变量 / 自动插桩 / 显式生命周期）
- **机制**: `AEGIS_INSTRUMENT=1 python my_agent.py`（import 时自动 patch）；`auto_instrument(frameworks=[...], on_block="warn")`（可选框架显式）；`Aegis.init(config)`（完整生命周期：audit/guardrails/cost/integrations）
- **证据**: instrument/__init__.py:128,349-355；init.py:64-131
- **links**: mechanism↔G01；subsystem↔G05（guardrails 接入）

## EK-G03 · 声明 vs 评估字段分离（Agent 无权 author 评估字段）
- **机制**: ActionClaim 三组 frozen dataclass——DeclaredFields（agent 自述：proposed_transition/target/justification/declared_impact）、AssessedFields（Aegis 计算：独立 impact_profile，"Agent MUST NOT author these"）、ChainFields（委托链）；justification_gap = declared vs assessed 距离
- **证据**: core/action_claim.py:137-236
- **links**: mechanism↔G04；constraint→G10（claim_policy 四步）

## EK-G04 · Policy 引擎 LRU 缓存 + 条件规则防遮蔽
- **机制**: `evaluate` 缓存 key=(action.type, action.target, action.agent_id)，LRU 256；`_is_cacheable` 对任何带 conditions 且 regex 匹配的规则 → 该 action 不缓存——"prevents the scenario where a cached unconditional match shadows a conditional rule"
- **证据**: core/policy.py:169-228
- **links**: mechanism↔G03；contrast↔G05（策略数据 vs 策略代码）

## EK-G05 · YAML 声明式策略（策略即数据）+ merge 环境覆盖
- **机制**: `Policy.from_yaml/from_yaml_files`；`defaults`（risk_level/approval）+ `rules`（match type→risk_level/approval + conditions）；`merge`：other 规则追加、defaults 取 base（"environment-specific overrides"）
- **证据**: core/policy.py:256-360；policy.example.yaml（read/navigate/screenshot auto；click/fill/write approve；bulk_update>100 升级）
- **links**: causal→G04；contrast↔G06（声明式 vs LLM 生成）

## EK-G06 · autopolicy LLM 生成策略（自然语言→policy dict）
- **机制**: OpenAIPolicyGenerator/AnthropicPolicyGenerator.generate(description)→dict；`_validate_policy_dict` 校验；Anthropic 侧 tool_def 构建
- **证据**: core/autopolicy_llm.py:143-267
- **links**: contrast↔G05；constraint→G07（生成需校验）

## EK-G07 · 信任委托单调约束（trust 非递增）
- **机制**: `validate_monotone_constraint(chain)`——trust_level 沿委托链非递增；违反 → ClaimPolicyDecision 解释追加 "Monotone constraint violated in delegation chain"
- **证据**: core/action_claim.py:109-121；core/claim_policy.py:186-190
- **links**: mechanism↔G03；causal→G10

## EK-G08 · claim_policy 四步评估 + 复合解释
- **机制**: evaluate 四步——①assess（if needed，独立 assessor 计算 assessed fields）②policy evaluate（Action bridge）③selection audit（可选，selection_context 时惰性构造 SelectionAuditor）④composite decision（含 monotone_valid/justification_gap/congruence_score/selection findings + 人读 explanation）
- **证据**: core/claim_policy.py:152-220
- **links**: dependency←G03/G04/G07；subsystem↔G10

## EK-G09 · GuardrailEngine 多 guardrail 管线 + pack
- **机制**: add/check/check_and_transform/acheck/acheck_and_transform/from_pack——同步+异步双 API；guardrails 链式检查与变换
- **证据**: guardrails/engine.py:16-172
- **links**: subsystem↔G02；constraint→G11（audit 记录）

## EK-G10 · fail-safe 默认（无规则匹配 → BLOCK/DENY）
- **机制**: AGP Level1 conformance："Fail-safe policy (BLOCK/DENY on destructive actions when no rules match)"——conformance 测试以真实引擎验证；policy.py defaults 是 `<default>` 落点
- **证据**: tests/conformance/test_agp_level1.py:11-30；core/policy.py:207-211
- **links**: mechanism↔G07；contrast↔G05（显式规则 vs 默认）

## EK-G11 · Merkle 防篡改审计（规范 JSON 叶哈希）
- **机制**: `_canonical_json` → `_hash_leaf`（SHA256）→ MerkleAuditTree（root_hash/proof/batch proof）——审计事件可证伪
- **证据**: core/merkle_audit.py:57-187
- **links**: dependency→G09（guardrail 事件入审计）；subsystem↔G15

## EK-G12 · AgentIdentity 能力矩阵（capabilities glob 匹配）
- **机制**: AgentIdentity frozen dataclass；capabilities frozenset；`capability_matches(capability, pattern)` glob——能力声明即权限根
- **证据**: core/agent_identity.py:50-122
- **links**: mechanism↔G03（identity 是 claim 的上下文）；constraint→G07

## EK-G13 · MCP tool shadowing 检测（4 类向量）
- **机制**: exact_duplicate / typosquat（Levenshtein+Unicode confusables）/ description_override / capability_claim；trusted_servers 白名单
- **证据**: core/mcp_shadow.py:1-30
- **links**: mechanism↔G14（都是 MCP 攻击面检测）；subsystem↔mcp_* 族

## EK-G14 · MCP 跨 server 升级链检测（source→sink 滑动窗口）
- **机制**: EscalationRule（危险 source→sink 模式）+ ToolCallRecord（不可变）+ EscalationFinding（时序）+ EscalationDetector（滑动窗口逐调用评估）——如"一 server 读凭据 → 另一 server 外发"
- **证据**: core/mcp_escalation.py:1-30
- **links**: mechanism↔G13；constraint→G10（发现→决策）

## EK-G15 · ZeroTrustAgent 会话级验证（每请求验身份）
- **机制**: AgentCredential（credential 类型枚举）/ VerificationResult / SessionToken / ZeroTrustPolicy——零信任每请求验证
- **证据**: core/zero_trust_agent.py:66-155
- **links**: mechanism↔G12（identity 为基础）；subsystem↔G11（审计）

## EK-G16 · init 失败降级（warning 不阻断）
- **机制**: SQLite audit logger 初始化异常 → warning（"Failed to initialise SQLite audit logger"）继续；guardrails/cost 空配置跳过——治理组件可选择性失败
- **证据**: init.py:195-225
- **links**: contrast↔G10（fail-safe 决策 vs 组件降级）；constraint→G17

## EK-G17 · AGEF/AGP 自述标准 + conformance 自我钉住
- **机制**: specs/agef/v1/schema.json（事件格式，自称"AI governance 的 SARIF/CEF"）；specs/agp/v1/protocol.md（治理协议，称"MCP 标准化通信侧，AGP 标准化治理侧"）；conformance 测试（test_agef_schema/test_agp_level1）以**本引擎**验证——自我一致但无第三方采纳证据
- **证据**: specs/agef/v1/README.md:1-40；specs/agp/v1/protocol.md:1-25；tests/conformance/
- **links**: constraint→全部（标准是治理对象的契约面）；contrast↔corpus 外标准（如 OWASP/AGENTiGraph 类）

## EK-G18 · 集成补丁可逆（_patch_openai/_patch_anthropic + teardown）
- **机制**: Aegis._patch_openai/_patch_anthropic（on_block 语义）；_deactivate → _teardown_integrations（撤销补丁）——治理可装卸
- **证据**: init.py:328-389
- **links**: mechanism↔G02；dependency→G01

## EK 图质量门
- 18 条 EK；links 覆盖率 100%；平均出边 ~1.8；游离 0%；全部可回溯仓库实盘（HEAD 134580e）

## EK-G19 · AEGIS_* 环境变量配置面 [reconciled via R-1]
- **机制**: env 配置族 = 第二配置入口：`AEGIS_INSTRUMENT`（1/true/yes/on → import 时自动插桩）+ `AEGIS_ON_BLOCK`（默认 **"raise"**——fail-closed 语义）+ `AEGIS_AUDIT` + `AEGIS_GUARDRAILS`（默认 "default"）；与 YAML 策略配置并存
- **证据**: instrument/__init__.py:349-365（_maybe_auto_instrument）
- **links**: mechanism↔EK-G02（接入梯度族）；causal→EK-G10（fail-safe 默认）

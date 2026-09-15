# 02 — Engineering Knowledge（EK Graph，宽底座）

> 每条 EK 声明 links（6 类边）+ evidence（file:line 可回溯）。36 条，L1/L2 为主。

## EK-01 [L2] govern() 两行集成 API 是 AGT 的单一入口
- **内容**：`govern(fn, policy=..., agent_id="*", audit=True, on_deny=None, approval_handler=None, conflict_strategy="deny_overrides", ring=None, rego_path/content, trace=None)` 包装任意 callable 为 GovernedCallable。2-line integration 是产品核心承诺。
- **evidence**：agentmesh/governance/govern.py:701（签名）+ README Quick Start
- **links**：mechanism→EK-02；causal→EK-03

## EK-02 [L2] GovernedCallable 是核心原语：ring → policy → approval → audit → execute
- **内容**：执行顺序固定——①ring enforcement（denied 不达 policy）②policy evaluate③require_approval 处理④audit log⑤on_deny 回调或 raise GovernanceDenied 或执行原函数。
- **evidence**：govern.py:174（class）+ :231（__call__）
- **links**：mechanism→EK-01；causal→EK-04→EK-07；dependency→EK-08

## EK-03 [L2] PolicyEngine 决策：YAML 优先，多策略收集候选 + 冲突解决
- **内容**：`evaluate()` 收集 ALL matching rules across applicable policies（按 agent_did + stage 过滤），然后 PolicyConflictResolver.resolve。YAML 规则先查，rego 未命中才查。
- **evidence**：policy.py:689（class）+ :1036（evaluate）
- **links**：subsystem→EK-04；contrast→EK-13（ACS 单 manifest 决策）

## EK-04 [L2] 四种冲突解决策略
- **内容**：deny_overrides（govern 默认，任何 deny 胜）/ allow_overrides / priority_first_match（引擎默认，v1 行为）/ most_specific_wins（agent-scoped > tenant > global）。
- **evidence**：policy.py:704-725 + conflict_resolution.py
- **links**：subsystem→EK-03；constraint→EK-05

## EK-05 [L2] fail-closed 无开关：策略评估错误一律 deny
- **内容**：rule evaluate 异常 / 后端超时 / regex 编译失败 → 立即 deny + error:true 审计。无配置项切 fail-open。与 AI Protector"检测层可失败降级"形成对照。
- **evidence**：policy.py:168-172（match = self.action != "allow"，allow 规则失败 → fail-open 危险）+ ADR-0013
- **links**：contrast→AI Protector EK-05（软贡献）；constraint→EK-03

## EK-06 [L2] 策略生命周期 stage 过滤（pre_input/pre_tool/post_tool/pre_output）
- **内容**：每条 rule 声明 stage，evaluate 只跑当前 stage 的规则——策略可作用于 agent 生命周期不同点（非全量扫描）。
- **evidence**：policy.py:1036（stage 参数）+ policy.py:109（Rule.action Literal）
- **links**：subsystem→EK-03；mechanism→AI Protector EK-30（pre_llm_only）

## EK-07 [L2] require_approval：动作绑定审批协议（ADR-0030）
- **内容**：rule action=require_approval + approvers → ApprovalCoordinator；approval 记录绑定 policy version（ADR-0030）与 TTL（默认 300s）与 chain_id。策略版本绑定防止"审批后策略已变仍有效"。
- **evidence**：govern.py:260-290（_handle_approval）+ ADR-0030
- **links**：causal→EK-02；dependency→EK-09

## EK-08 [L1] Audit 默认开启，哈希链 + Merkle 树双结构防篡改
- **内容**：AuditLog（audit=True）记录 policy_evaluation 事件；**双结构**：①哈希链（AuditEntry.previous_hash = 上条 entry_hash，verify_chain 校验链连续性，检测改/删）②Merkle 树（增量构建 _tree/_root_hash，verify_proof 提供单条目存在性证明）。SHA-256，无外部区块链锚定，自包含验证。
- **evidence**：audit.py:57（AuditEntry）+ :268-330（MerkleAuditChain 双结构）+ :443（verify_chain）+ ADR-0017
- **links**：dependency→EK-02；mechanism→EK-09

## EK-09 [L2] Decision BOM 可重构而非预构建（ADR-0018）
- **内容**：不存储决策时 BOM，事后从四类信号源（AuditSource/TrustSource/PolicySource/TraceSource）重构决策上下文。设计原则"less complete but non-invasive > complete but invasive"。
- **evidence**：decision_bom.py:1-30（docstring 设计原则）+ ADR-0018
- **links**：contrast→EK-10；dependency→EK-08

## EK-10 [L1] 事件总线 + OTel 导出为可观测底座
- **内容**：governance 决策经 event_bus/OTel trace 暴露；DecisionBOM 依赖 TraceSource 的 OTel spans 重构。otel_observability.py + trace_sink.py（TRACE v0.1 信任记录，ADR-0032）。
- **evidence**：governance/otel_observability.py + trace_sink.py:185（agentrust_trace 依赖）
- **links**：dependency→EK-09；mechanism→AI Protector EK-29（node timings）

## EK-11 [L2] 信任分 0-1000 四维加权 + 阈值映射能力
- **内容**：trust = 0.35×compliance + 0.25×task_success + 0.25×behavior + 0.15×identity；分档（900+ Verified Partner / 500 默认 / 0-299 Untrusted 阻断）。violation -50 硬惩罚，1000 动作/7 天窗口。
- **evidence**：docs/security/trust-score-calibration.md
- **links**：subsystem→EK-12；mechanism→AI Protector EK-01（加权风险分）

## EK-12 [L2] Trust Ceiling Propagation：委托链信任单调收敛
- **内容**：父代理信任分成为子代理硬上限（initial clamp min(initial, ceiling)，后续更新也受上限约束）；多级委托逐级 min。防"低信任父 spawn 高信任子"提权。类比 capability 系统。
- **evidence**：ADR-0016 + identity/delegation.py
- **links**：subsystem→EK-11；constraint→EK-14

## EK-13 [L2] ACS：Rust 无状态确定性策略决策核心
- **内容**：Agent Control Specification——stateless（无状态影响后续 verdict）、deterministic（同 manifest+snapshot+mode → 同 verdict+transform）、fail-closed（错误→deny + reserved runtime error reason）。宿主在每个干预点传完整 JSON snapshot。
- **evidence**：policy-engine/README.md（Core properties）+ docs/security-model.md
- **links**：contrast→EK-03（Python 引擎）；dependency→EK-15

## EK-14 [L2] 干预点（Intervention Points）：agent_startup/input/pre_model_call/pre_tool_call/tool_result/output/post_*
- **内容**：ACS 把 agent 生命周期切为命名干预点，策略绑定到点。policy_target 用 JSONPath（$.tool_call.args）+ tool_name_from。
- **evidence**：policy-engine/README.md（Intervention points 表 + example manifest）
- **links**：subsystem→EK-13；mechanism→EK-06（stage 过滤对照）

## EK-15 [L2] Verdict 三值模型（v5 简化）
- **内容**：Decision 从五值减为三值：warn→allow+warnings[]，escalate→deny+approval block，transform 是唯一改值的动作面（effects plane 移除）。
- **evidence**：policy-engine/core/src/lib.rs:36-45（deprecation 注释）+ agent_control_spec
- **links**：constraint→EK-13；contrast→AI Protector EK-04（三态 BLOCK/MODIFY/ALLOW）

## EK-16 [L2] ACS 核心外移 crates.io：本地 core 是 deprecation shim
- **内容**：policy-engine/core/lib.rs 现在只是兼容层（re-export agent_control_spec 的旧名）；真实运行时在 crates.io agent-control-spec（agent-hooks 契约）。决策逻辑不在本仓库 src 里。
- **evidence**：policy-engine/core/src/lib.rs:1-20（"Deprecation shim"）+ :36（rename 表）
- **links**：dependency→EK-13；constraint→EK-17

## EK-17 [L1] 安全注意：manifest provenance 门控宿主凭据读取
- **内容**：AGT 曾把宿主环境凭据读取门控在 manifest provenance（网络获取的 manifest 不能碰凭据）；agent-control-spec 0.4.0-alpha.3 因仍支持 URL extends 未带此门——文档警告"不要启用 bundled dispatcher 直到上游恢复"。
- **evidence**：policy-engine/core/src/lib.rs:48-55（Security note）+ docs/acs-retarget.md
- **links**：constraint→EK-16；contrast→AI Protector EK-38（管理面无认证——不同攻击面）

## EK-18 [L2] 四执行环取代 RBAC 为运行时主权限模型（ADR-0002）
- **内容**：Ring 0 ROOT / 1 PRIVILEGED / 2 STANDARD / 3 SANDBOX；每环映射 ResourceConstraints（network/filesystem/subprocess/max_tools）。RBAC 保留人管/合规/scope，但沙箱 live agent 执行主用 rings。
- **evidence**：docs/adr/0002 + hypervisor/rings/enforcer.py（RING_CONSTRAINTS）
- **links**：subsystem→EK-19；mechanism→AI Protector EK-18（RBAC 双门——对照：AGT 用环更细）

## EK-19 [L2] RingEnforcer 资源约束 + 命令 denylist + breach detector
- **内容**：enforcer 把 ring 映射到具体资源约束；DENIED_COMMANDS 子进程命令黑名单；共享 breach detector 按 (agent_id, session_id) 跨 callable 计数违规触发熔断。
- **evidence**：hypervisor/rings/enforcer.py（ResourceConstraints + RING_CONSTRAINTS + DENIED_COMMANDS）+ rings/breach_detector.py
- **links**：subsystem→EK-18；causal→EK-20

## EK-20 [L2] ring 检查先于 policy：被拒 ring 不达策略引擎
- **内容**：GovernedCallable.__call__ 中 ring_denial 检查在 policy evaluate 之前——ring 拒绝是更粗更快的防护层，策略是细粒度授权层。分层防御。
- **evidence**：govern.py:243-253（"Ring enforcement — runs before policy evaluation"）
- **links**：causal→EK-02；subsystem→EK-19

## EK-21 [L2] MCP Security Gateway：工具调用治理（OWASP ASI02）
- **内容**：MCP 客户端↔服务器之间的治理层：tool allow/deny 列表 + 参数净化（危险模式）+ per-agent 速率/预算 + 结构化审计 + HITL 审批 + 响应扫描（注入/凭据泄露/exfil）。
- **evidence**：agent_os/mcp_gateway.py:1-35（docstring 覆盖 ASI02）
- **links**：subsystem→agent-os；mechanism→AI Protector EK-16/17（pre/post-tool 双门——同类工具治理）

## EK-22 [L2] Rego 集成是 YAML DSL 的逃生舱，但输入形状有坑
- **内容**：YAML 规则先查；rego_path/content 补充（需 OPA CLI）。两个坑：①标量 kwarg 被包成 {"value": x}——input.role == "auditor" 永不匹配而 input.role.value 才匹配（静默 false）；②YAML DSL 只支持 field-vs-literal 比较，field-vs-field 规则静默落到 default_action。
- **evidence**：govern.py:720-760（rego_path docstring 详细警告）
- **links**：contrast→EK-03；constraint→EK-22b

## EK-22b [L2] 引擎与 adapters 的"静默不匹配"风险是系统性设计约束
- **内容**：DSL 表达能力受限 + Rego 包装形状差异，都可能让规则"永不匹配但不报错"——fail-closed 只覆盖异常，不覆盖"规则没写对"。测试须显式覆盖规则是否真命中。
- **evidence**：govern.py:720-760（"silently never matches... rather than raising an error"）
- **links**：dependency→EK-22；constraint→EK-05

## EK-23 [L2] policy extends 递归合并（additive-only，父 deny 不可削弱）
- **内容**：load_yaml_file 解析 extends 递归，additive-only 合并语义；父 deny 规则不可被子级削弱（v4 记录）。v5 起 native 用 ACS extends，folder-merge 规则 superseded（保留给 v4→v5 迁移命令）。
- **evidence**：policy.py:750-770（extends 注释）+ docs/adr/0014（superseded）
- **links**：constraint→EK-03；contrast→AI Protector EK-20（阈值热更新——更新策略的两条路）

## EK-24 [L2] 速率限制绑定 winning rule（防同名规则 shadow）
- **内容**：_rate_limits 按 (agent_did, policy_name, rule_name)；winning rule 从 winning policy 解析（非首名匹配），防跨策略同名规则 shadow 限流。
- **evidence**：policy.py:1112-1130（"Resolve the rule from the WINNING policy"）
- **links**：subsystem→EK-03；mechanism→AI Protector EK-19（session budget——同类预算机制）

## EK-25 [L1] policy bundle hash（sha256）+ TRACE sink（ADR-0032）
- **内容**：GovernedCallable 加载时算 policy 字节 SHA-256（两次：一次给 TRACE sink）；trace 配置时 TRACEAuditSink 以 agent_did + policy_bundle_hash 发射信任记录。
- **evidence**：govern.py:213-230（_policy_bundle_hash）+ governance/trace_sink.py:185
- **links**：mechanism→EK-09；dependency→EK-26

## EK-26 [L1] agentrust-trace 是 TRACE 发射的可选硬依赖
- **内容**：trace_sink.emit 内部 import agentrust_trace（TrustRecord/sign_record/load_signing_key），缺失时 raise RuntimeError（"Install agentrust-trace"）。4 个本地测试失败即因此。
- **evidence**：governance/trace_sink.py:185-187
- **links**：dependency→EK-25；constraint→EK-16（外部运行时依赖的又一例）

## EK-27 [L2] 包整合期：agent-os-kernel / agentmesh-platform deprecated
- **内容**：v5 合并到 agent-governance-toolkit-core；旧包 import 即 DeprecationWarning（agent-os/__init__.py:17-24, agentmesh 同）。pyproject 注释记录 #2794 事故：stripped dev extra 导致 CI 全 ModuleNotFoundError，#2969/#2972 修复。
- **evidence**：agent-os/__init__.py:17-24 + agent-mesh/pyproject.toml（dev extra 注释）
- **links**：constraint→EK-16；contrast→AI Protector EK-24（依赖低估修复——同类教训）

## EK-28 [L1] PR #2794 教训：CI 配置退化可静默破坏测试可运行性
- **内容**：package consolidation 时删了 [project.optional-dependencies] 整段，`.[dev]` 变 no-op，测试收集全挂（fastapi/pydantic/cryptography 等 import 失败）。修复是恢复 dev extra 完整清单。测试"能跑"是需要守护的基础设施属性。
- **evidence**：agent-mesh/pyproject.toml dev extra 注释（"PR #2794 stripped... left the test suite unable to import"）
- **links**：mechanism→EK-27；constraint→AGENTS.md（贡献规范）

## EK-29 [L2] 29 条 ADR 支撑的规范驱动开发
- **内容**：每个组件有 RFC 2119 规范 + conformance tests（992）；设计决策全部 ADR 化（identity/rings/IATP/merkle/BOM/fail-closed/extends/approval/TRACE）。"为什么"可回溯。
- **evidence**：docs/adr/index.md + README（992 conformance）
- **links**：subsystem→EK-13；mechanism→AI Protector（issues.md 记录驱动——对照：ADR 是设计侧，issues 是故障侧）

## EK-30 [L2] 供应链与发布治理：SLSA/Scorecard/SBOM/Dependabot/CodeQL
- **内容**：40+ workflows；sbom/sbom-diff/scorecard/supply-chain-check/weekly-security-audit/auto-merge-dependabot/codeql。发布产物有 provenance。
- **evidence**：.github/workflows/ 列表 + README badges
- **links**：subsystem→EK-29；mechanism→AI Protector EK-33（Dependabot+CodeQL 同款）

## EK-31 [L2] 威胁模型显式化：7 类威胁 × 缓解（SECURITY.md）
- **内容**：Policy bypass/Identity spoofing/Audit tampering/Budget evasion/Tool-call injection/Supply chain/Privilege escalation via delegation——每类有确定性缓解。信任边界：Agent(untrusted) → AGT Policy Engine(trust anchor) → Protected Resources + tamper-proof Audit。
- **evidence**：SECURITY.md（Threat Model 段）
- **links**：subsystem→EK-29；mechanism→AI Protector EK-32（三信任边界——同款显式化）

## EK-32 [L2] Budget evasion 防御：IEEE 754 特殊值拒绝
- **内容**：成本输入校验拒绝 NaN/negative 等（预算绕过面）；advisory 机制（AdvisoryCheck）在决策路径外提供风险提示。
- **evidence**：SECURITY.md（Budget evasion 行）+ governance/advisory.py
- **links**：subsystem→EK-03；contrast→AI Protector EK-19（预算——同类）

## EK-33 [L2] 决策确定性分层：确定性层为规范信任边界（独立验证修正）
- **内容**：README/SECURITY 自述"无 LLM 在策略路径"需精确化：**确定性策略层（PolicyEngine/ACS）是规范信任边界**（无 LLM）；advisory 层是非确定性可选防线，但**被剥夺放行权（只能 block，不能 allow），失败默认 allow**——非确定性不参与"批准"，只参与"阻止"。ICO 引用 ICLR 2025 100% ASR 数据划界 prompt 层防御。
- **evidence**：README（The Problem 段）+ SECURITY.md + govern.py:310-323（advisory 位置）+ advisory.py（只能收紧）
- **links**：contrast→AI Protector EK-27（HARM_ML_MODE 权衡）；constraint→EK-37

## EK-34 [L1] 测试布局：e2e_python/redteam/unit/smoke + benchmarks
- **内容**：tests/ 分 e2e_python（scenarios/support）、redteam（benchmark）、unit；benchmarks/prompt-injection 是独立评估 fixture（280 行 smoke：110 attack/170 benign，确定性生成 + hygiene 检查 + Wilson interval 汇总；证据输出不含原始 prompt 文本）。
- **evidence**：tests/ 目录 + benchmarks/prompt-injection/README.md
- **links**：subsystem→EK-29；mechanism→AI Protector EK-25/26（benchmark grader 同族）

## EK-35 [L2] 适配层契约：framework adapters 统一走 govern 原语
- **内容**：LangChain/CrewAI/AutoGen/OpenAI/ADK/smolagents 适配全部建立在 GovernedCallable 之上（"framework-specific wrappers build on it"）——核心原语稳定，适配层薄。
- **evidence**：govern.py:174-180（"core primitive — framework-specific wrappers build on it"）
- **links**：mechanism→EK-01；constraint→AGENTS.md（routing 规则）

## EK-36 [L2] 包结构演进：顶层独立语言 SDK 是 point-in-time 承诺
- **内容**：AGENTS.md 明示"repo layout 是点状态非永久架构承诺"；语言 SDK 未来可能迁到独立仓库。路径演进是 AGT 治理的一部分（非代码冻结）。
- **evidence**：AGENTS.md（Repository Layout Status）
- **links**：constraint→EK-27；contrast→Guardian（单包无此问题）

## EK-37 [L2] Advisory 层：确定性 allow 之后的非确定性防线（独立验证补充）
- **内容**：GovernedCallable.__call__ 在确定性 allow 后、执行前运行可选 advisory 层（分类器式 defense-in-depth）。**只能收紧（block/flag_for_review），不能放宽；失败默认 allow（deterministic layer is canonical）；决策带 `deterministic: false` 审计标记**。含 SSRF 防护（_BLOCKED_HOSTS：cloud metadata 169.254.169.254 / metadata.google.internal / fd00:ec2::254 + URL scheme 白名单）。
- **evidence**：govern.py:310-323（advisory 在 allow 后）+ advisory.py:1-60（docstring 约束 + _BLOCKED_HOSTS）
- **links**：constraint→EK-33（非确定性层被剥夺放行权）；contrast→AI Protector EK-27（ML 在路径内可放行——AGT advisory 只能 block 不能 allow，授权方向相反）

## EK-38 [L2] Authority Resolver：无 YAML 规则匹配时的信任窄化兜底（独立验证补充）
- **内容**：PolicyEngine.evaluate 尾部：无 applicable 策略或无匹配规则时，若有 authority_resolver 则走 trust-based narrowing——DelegationInfo（delegated_capabilities）+ TrustInfo（score 默认 500 / risk_level 默认 medium）+ ActionRequest（action_type/tool_name/resource/requested_spend）合成 AuthorityRequest，按信任窄化授权。这是"YAML 未覆盖时的兜底授权面"，与 fail-closed 形成对照（兜底是窄化而非放行）。
- **evidence**：policy.py:1145-1185（"2. Authority resolution (trust-based narrowing)"）
- **links**：subsystem→EK-03；contrast→EK-05（fail-closed 对异常，resolver 对未覆盖——两个兜底方向）

---

## EK Graph 自检
- 36 条 EK，全部声明 links（平均出边 ≥2）✓
- 六类边覆盖：mechanism（EK-01/02/08/21/24/25...）/ subsystem（03/04/11/18/19/29...）/ causal（01→02→04→07→20）/ dependency（02→08→09→10→25→26）/ constraint（04→05→12→17→22b→27）/ contrast（03↔13、05、15、18、23、27、33）✓
- 与前三项目（Aigis/Guardian/AI Protector）的 contrast 边已建立 ✓

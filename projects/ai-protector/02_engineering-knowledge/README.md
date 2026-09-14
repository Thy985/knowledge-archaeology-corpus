# 02 — Engineering Knowledge（EK Graph，宽底座）

> v3.1：每条 EK 声明 links（6 类边：mechanism/subsystem/causal/dependency/constraint/contrast）。证据 = file:line。Evidence strength：S3 实现 / S4 测试 / S5 运行 / S7 自述。

## A. 编排与决策（核心机制）

- **EK-01 LangGraph 9 节点管线** — `pipeline/graph.py:22`。proxy 防火墙 = StateGraph：parse→intent→rules→scanners→decision；输出侧 llm_call→output_filter→logging。S3。links: [mechanism:EK-31] [causal:EK-02]
- **EK-02 三态决策路由** — `pipeline/graph.py:18 route_after_decision`：BLOCK→logging→END；MODIFY→transform→llm_call→output_filter→logging；ALLOW→llm_call→output_filter→logging。MODIFY 是 Aigis(allow/deny/review)、Guardian(halt/sandbox/log-only) 之外的第三种响应（改写而非阻断）。S4（test_graph 97 用例含三态路由）。links: [causal:EK-01→EK-02] [subsystem:EK-16]
- **EK-03 加权风险分聚合** — `pipeline/nodes/decision.py:9 calculate_risk_score`：intent 类别基础分（jailbreak 0.6 / role_bypass 0.5 / agent_exfiltration 0.5 …14 类）+ denylist 0.8 + jailbreak_ml×0.8 + harm_ml×0.9 + LLM Guard 注入×0.8 + PII 按实体 0.1 封顶 0.5 + NeMo×0.7 + custom boost，cap 1.0。权重全部 policy_config.thresholds 可配（S3）。links: [dependency:EK-04 依赖 EK-03] [subsystem:EK-05]
- **EK-04 决策优先级** — `decision.py decision_node`：denylist 硬 BLOCK → PII block → risk≥max_risk(0.7) BLOCK → PII mask→MODIFY → suspicious_intent BLOCK → secrets BLOCK → ALLOW。确定性、无 LLM。S4（test_decision_transform 通过）。links: [causal:EK-03→EK-04] [mechanism:EK-26]
- **EK-05 NeMo 硬阻断→软贡献修复史** — `decision.py:112-118 注释`："Hard-blocking here caused false positives on benign queries like 'search products laptop' because NeMo embeddings matched attack intents at low similarity thresholds"。语义 rail 从硬阻断改为风险分贡献。**规则层容错**。S3。links: [contrast:EK-20] [subsystem:EK-03]

## B. 检测层（A2 反混淆 + 分类 + 并行扫描）

- **EK-06 A2 反混淆 build_scan_text** — `pipeline/utils/deobfuscate.py:200`：raw 恒在首位 + 去零宽 + leet（仅 digits/symbols 粘连字母，i/l 两变体）+ 去间隔 + homoglyph 折叠 + ROT13（条件） + base64 变体；去重后 raw+"\n"+variants。S3。links: [mechanism:EK-07] [constraint:EK-08] [constraint:EK-12]
- **EK-07 原文第一 + 变体组合的实验证据** — `deobfuscate.py:252-255 注释`："Variants-first was measured to LOWER obfuscation detection ~13 pp — the combined raw+decoded signal scores best with the obfuscated form leading"。**实验驱动设计**。S3。links: [mechanism:EK-06] [causal:EK-22→EK-07]
- **EK-08 反混淆 fail-safe** — `deobfuscate.py:258-260`：整体 try/except 返回 raw——"Detection must never be broken by a decoding edge case"。S3。links: [constraint:EK-06] [mechanism:EK-28]
- **EK-09 ROT13 条件解码** — `deobfuscate.py:212-216`：仅当 raw 不像英语（_english_score 低于阈值）且解码后变英语才应用——避免把良性英文解码成注入样乱码。S3。links: [mechanism:EK-06]
- **EK-10 role-bypass 26 条模式** — `nodes/intent.py:81 AGENT_ROLE_BYPASS_PATTERNS`："i am admin"/"elevate my privileges"/"my boss approved"/"emergency override" 等。S3。links: [subsystem:EK-11] [mechanism:EK-16（agent 门复用同族 intent）]
- **EK-11 tool-abuse 模式 + 分类** — `nodes/intent.py:110 AGENT_TOOL_ABUSE_PATTERNS` + `intent.py:541 classify_intent`：~80 regex/substring → 攻击类别（jailbreak/tool_abuse/role_bypass/agent_exfiltration/social_engineering/harmful_content/system_prompt_extract/rag_poisoning/confused_deputy/template_injection/crescendo…）。S4（test_intent_rules 通过）。links: [subsystem:EK-10]
- **EK-12 scan_text 与 LLM 隔离** — `state.py` 注释："Deobfuscated detection view (raw + decoded variants); LLM never sees this"；`PROXY_FIREWALL_PIPELINE.md`："the original message is what the LLM receives"。检测视图与生成视图分离。S3。links: [constraint:EK-06]
- **EK-13 扫描器并行 + 单失败不崩** — `nodes/scanners.py:12-13`："Each scanner runs as an independent asyncio.gather task. If one scanner fails, the other's results are still merged"。`policy_config.nodes` 控制启用的扫描器（空=fast path）。S3。links: [mechanism:EK-02] [constraint:EK-28]

## C. 输出侧（第三防线）

- **EK-14 输出过滤三防线** — `nodes/output_filter.py`：PII redact（Presidio）+ secrets redact（regex）+ **system-leak 检测**（known fragments → `[SYSTEM_REDACTED]`）。输出侧防护是 Aigis/Guardian 均无的品类差异点。S4（test_output_filter 通过）。links: [subsystem:EK-15] [contrast:Guardian 无输出侧]
- **EK-15 system-leak 检测局限（ISS-007）** — `issues/issues.md:103`：仅 4 条硬编码 fragment，换措辞即可泄露——**已知局限**（S7 记录未修复）。links: [subsystem:EK-14] [causal:EK-15→candidates:多语言绕过]

## D. Agent 门控（动作防护）

- **EK-16 pre-tool gate 5 项** — `agent/nodes/pre_tool_gate.py`：①RBAC scope=read ②args Pydantic schema + 注入扫描 ③上下文风险（会话消息注入模式）④session budget（token/cost）⑤requires_confirmation（RBAC 驱动）。S4（agent-demo test_pre_tool_gate 通过）。links: [dependency:EK-18] [mechanism:EK-02]
- **EK-17 post-tool gate 间接注入检测** — `agent/nodes/post_tool_gate.py:236 scan_injection`：工具输出中的 indirect prompt injection patterns + PII/secrets 脱敏 + 大小限制。**工具输出被视为不可信输入**。S4。links: [subsystem:EK-16]
- **EK-18 RBAC 数据模型** — `agent/rbac/models.py`：ToolDefinition(category data_read|data_write|admin|external × sensitivity low..critical × requires_confirmation × rate_limit) + ToolPermission(role→tool→scopes) + RoleConfig(inherits 继承) + PermissionResult。S4（test_rbac 通过）。links: [dependency:EK-16 依赖 EK-18] [mechanism:Aigis RBAC/Guardian JWT ACL 对照]
- **EK-19 agent 需确认工具暂停** — `agent/graph.py _after_gate`：pending_confirmation → confirmation_response（向用户确认敏感工具调用），非自动放行。S3。links: [causal:EK-16→EK-19]

## E. 失败与修复（认知成本沉淀）

- **EK-20 LLM Guard 阈值热更新（ISS-001 已修）** — `issues/issues.md:9-20`：`_active_thresholds` 追踪 init 时阈值，请求阈值变化自动重建扫描器，免重启。S7（项目记录已修）。links: [contrast:EK-05]
- **EK-21 冷启动预加载（ISS-002 已修）** — `issues/issues.md:22-33` + `main.py lifespan`：ML 模型启动时 `asyncio.create_task(_preload_scanners())` 后台预载，首请求延迟 50s→0.9s。S7。links: [mechanism:EK-34]
- **EK-22 幻影功能（ISS-003/004 未修）** — `issues/issues.md:37-67`：ML Judge chip 仅设 flag 无 ml_judge node；Canary Tokens chip 设 enable_canary 但无任何 node 读取——**设计意图 ≠ 实现现实**。S7。links: [contrast:EK-23] [subsystem:UI 政策编辑器]
- **EK-23 已知绕过（ISS-005/012 未修）** — `issues/issues.md:68-84,158-222`：intent keyword-only 改写即绕；6 场景 xfail（leet/间隔/Caesar/ROT13/Turkish×2）全扫描器绕过（decision=ALLOW, risk≈0.00）。S4（xfail 标记实证）。links: [causal:EK-23→EK-07（A2 部分修复但未覆盖多语言）]
- **EK-24 依赖低估修复史** — `pyproject.toml` 注释：nemoguardrails 0.20.0 未声明 dataclasses-json/fastembed 两个运行时依赖，clean install 时 NeMo 扫描器 ModuleNotFoundError（CI/Docker 曾触发），显式声明后修复。S7。links: [causal:EK-24→EK-01]

## F. Benchmark 与可证明性（品类独有）

- **EK-25 客观真值 grader（no LLM-as-judge）** — `README.md` + `docs/red-team-oracle-calibration.md`：verdict 用 planted canary/secret 精确匹配（mechanical/exact）校准，94%→99% 客观准确率（修复后）；启发式判定标 "needs review" 不冒充证据。**证明 ≠ 自述**。S7。links: [contrast:EK-22] [mechanism:Guardian canary 蜜罐对照]
- **EK-26 Benchmark Hub 归一化** — `README.md`：~5,070 场景（~4,960 来自 6 公开数据集归一化同一 taxonomy + ~110 curated）；种子采样可复现；Quick default ~50。S7。links: [dependency:EK-25] [mechanism:test_scenario_deterministic 358 场景]
- **EK-27 HARM_ML_MODE 三档** — `config.py:106 harm_ml_mode=off` + `BENCHMARKS.md`：off/pre_llm/post_llm 用 2B granite-guardian 换覆盖（genuine-harm 55%→92%，FPR 0→0.7%），延迟 48ms→450ms。**延迟-覆盖显式权衡**。S7。links: [constraint:EK-26]

## G. 配置与运行时保障

- **EK-28 errors[] 非致命错误累积** — `state.py errors` + `runner.py`：节点错误进 errors 列表不使管线崩溃（fail-open，检测层容错）；配合 EK-08/EK-13 构成"检测层错不崩热路径"家族。S3。links: [mechanism:EK-08] [mechanism:EK-13]
- **EK-29 policy 四档 × nodes 列表驱动** — `state.py policy_name` + `scanners.py`：fast/balanced/strict/paranoid 每档一份 JSONB config（nodes 列表决定扫描器组合 + thresholds 加权），配置即管线形态。S3。links: [causal:EK-29→EK-13]
- **EK-30 pre_llm_only 路径** — `runner.py:143 run_pre_llm_pipeline`：streaming/scan 场景先跑 pre-LLM 段拿 ALLOW/BLOCK 决策再决定是否调用 provider。S3。links: [constraint:EK-02]
- **EK-31 API key client-side** — `state.py api_key` 注释 + README：外部 provider key 来自 x-api-key 头，**不落库不记录**。S7。links: [mechanism:EK-24]
- **EK-32 威胁模型三信任边界** — `docs/architecture/THREAT_MODEL.md`：UNTRUSTED(用户输入)→BOUNDARY1(proxy ingress)→SEMI-TRUSTED(防火墙管线)→BOUNDARY2(provider 调用，响应视为不可信过输出过滤)→BOUNDARY3(agent 工具执行，RBAC+双门)；TRUSTED(DB/Redis/config)。**响应与工具输出均不可信**。S7。links: [mechanism:EK-14] [mechanism:EK-17]
- **EK-33 lifespan 治理清理** — `main.py lifespan`：cancel stale benchmark runs + cleanup expired auth secrets（AES-256 SecretStore TTL 兜底，red-team 运行崩溃/重启安全网）。S7。links: [mechanism:EK-21]

## H. 测试揭示的行为

- **EK-34 668 单元测试本地通过（S4 实测）**：proxy 纯逻辑 203（intent/output_filter/decision/graph/rules 等 --noconftest）+ agent-demo 465（含 pre/post gate、RBAC、role separation、limits、trace）。**CI 1900+ 声明包含 DB 依赖部分，本地无 Postgres/Redis 不可复现**。S4（本地实测）。
- **EK-35 358 场景 = 可执行 benchmark** — `tests/test_scenario_deterministic.py`：358 场景（216 playground + 142 agent）pre-LLM 全管线，expectedDecision 断言，6 xfail 标混淆/多语言；**CI-ready ~35s**。S7（项目声明）+ S4（结构可复现，本地因 DB 依赖未全量通过）。links: [mechanism:EK-26]

## EK Graph 自检

- 平均出边：36 条 EK，边数约 45 → 平均 ≥1 ✓
- 游离 EK：无（全部 ≥1 边）✓
- 类别覆盖：核心机制 / 关键实现 / 关键决策 / 失败与修复 / 测试揭示行为 / 重要配置 / 边界与例外 全覆盖 ✓

---

## Reconciliation 增补（阶段 6，独立 Auditor 发现）

> 以下 3 条 EK 由独立验证（blind reconstruction）新增，补覆盖原考古遗漏的"绕过/管理面"攻击面。Evidence：直接代码抽查。

- **EK-36 [MISSING→新增] direct bypass endpoint** — `apps/proxy-service/src/config.py:117`：`enable_direct_endpoint: bool = True  # Set False in production`；`src/routers/direct.py:43 POST /v1/chat/direct`（docstring："Send prompt directly to LLM — NO scanning, NO policy, NO audit log. For Compare demo only"）。**绕过端点默认开启**，生产必须显式关闭。S3。links: [contrast:EK-02（主路径三态 vs 旁路零检测）] [constraint:EK-31]
- **EK-37 [MISSING→新增] streaming 路径无输出过滤** — `src/routers/chat.py:129-164`：stream 请求仅跑 `run_pre_llm_pipeline`（pre-LLM 到 decision），ALLOW/MODIFY 后直接 `sse_stream`（`llm/streaming.py` 无 output_filter/PII/secret/system-leak 引用）。**输出侧三防线（EK-14）只覆盖非流式响应**。S3。links: [constraint:EK-14（范围限定）] [causal:EK-30→EK-37（pre_llm_only 用于 streaming 的直接后果）]
- **EK-38 [MISSING→新增] 管理端点无认证** — `src/main.py` 仅 CORSMiddleware + CorrelationIdMiddleware；`routers/policies.py / rules.py / analytics.py / requests.py` CRUD 端点 `Depends` 仅 get_db（无 auth）。任何可达 :8000 者可改策略/规则/查请求日志。S3。links: [mechanism:EK-18（RBAC 覆盖 agent 工具层，但**不覆盖 proxy 管理面**）]

**Reconciliation 声明**：原 package 02 层未含上述 3 条（Auditor MISSING）；本增补使覆盖完整。KO-08 措辞收紧为"输出侧三线（PII/secrets/system-leak）为 AI Protector 独有；与 Aigis exfil 对照待定"。04 Flow Atlas 补充 bypass flow（direct endpoint 旁路 + streaming 输出旁路）见 04 层。

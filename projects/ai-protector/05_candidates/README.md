# 05 — Candidates（未验证假说 / 待验证模式）

> 不能确认的内容留在候选，不冒充知识。每条标注：类型 / 当前证据 / 缺失证据 / 验证路径。

## C-01 [Cross-project Hypothesis] "可证明安全"存在两条工程路径：Benchmark 客观真值与 TEE 硬件证明
- **陈述**：AI Protector 用"benchmark 客观真值 grader"（no LLM-as-judge）证明安全声明；Proof-of-Guardrail（arXiv 2603.05786，ICML'26）用 TEE 签名 attestation 证明 guardrail 执行。二者同为"可证明"主题的不同实现路径。
- **当前证据**：AI Protector S7（README + oracle calibration）；Proof-of-Guardrail 为雷达 #14 记录（TEE 签名 attestation，已实现于 OpenClaw）——**未直接读论文**。
- **缺失证据**：两路径的威胁模型差异（benchmark 证明"检测率"，TEE 证明"执行不可篡改"）未系统对照。
- **验证路径**：读 arXiv 2603.05786 全文 → 对照 AI Protector 的 provable 声明边界 → 判定是否构成互补模式。

## C-02 [Tentative Pattern] 安全网关的响应谱系正在收敛为三态（阻断 / 降级 / 放行记录）
- **陈述**：Guardian halt/sandbox/log-only ↔ AI Protector BLOCK/MODIFY/ALLOW——两项目独立收敛到三态响应。
- **当前证据**：2 项目（S8 弱：仅 2 个数据点）；Aigis 为 allow/deny/review 三态但语义不同。
- **缺失证据**：第 3 个独立数据点（AGT 待测）；三态的具体降级动作（MODIFY 改写 vs sandbox 隔离）无统一语义。
- **验证路径**：AGT（microsoft/agent-governance-toolkit）考古后对照。

## C-03 [Scope-uncertain] "No LLM in the loop" 的语义边界
- **陈述**：README 宣称 "No LLM in the loop"，但检测依赖本地 ML 模型（DeBERTa/DistilBERT/granite-guardian-2b）。"No LLM"应理解为"无外部 LLM API 调用/无 LLM-as-judge"，而非"无任何模型"。
- **当前证据**：README + config.py（jailbreak_ml_model/harm_ml_model 均为本地模型）。
- **缺失证据**：项目方对"LLM vs 本地判别模型"的明确术语区分文档。
- **验证路径**：查 ARCHITECTURE.md 是否定义术语边界；若否 → 保持 Candidate。

## C-04 [Scope-uncertain] A2 反混淆对多语言的覆盖边界
- **陈述**：A2（leet/间隔/homoglyph/ROT13/base64）覆盖英语混淆，但 ISS-012 记录的 Turkish 场景（AGT-065/066）仍 xfail——多语言绕过是已知缺口。
- **当前证据**：deobfuscate.py 实现 + test_scenario_deterministic 6 xfail（S4）。
- **缺失证据**：多语言检测的 roadmap 状态（issues 无修复计划）。
- **验证路径**：观察后续版本 xfail 移除情况。

## C-05 [Tentative Pattern] 本地模型网关的部署可靠性三件套（预加载 / 显式依赖 / 热更新）
- **陈述**：任何"本地 ML + 网关"服务必须解决：模型冷启动、隐藏运行时依赖、配置热更新。AI Protector 三件都有修复史（ISS-001/002/024）。
- **当前证据**：单项目 S7（修复记录）+ S4（668 测试）。
- **缺失证据**：跨项目对照（Guardian 无 ML 故无此问题——contrast 已记录，但无第二个 ML 网关实例）。
- **验证路径**：AGT 或其它本地模型网关考古后对照。

## C-06 [Unresolved Contradiction] README "5-layer" vs 架构文档 "7 layers / 9 nodes"
- **陈述**：README 开头称 "5-layer proxy firewall"（Quickstart 段），细节表格与 PROXY_FIREWALL_PIPELINE.md 为 7 层（Layer 1-7，Harm ML 可选）。版本演进（v2）措辞不一致。
- **当前证据**：README.md:59（5-layer）vs README.md:128-138（7 层表）+ PROXY_FIREWALL_PIPELINE.md（7 layers in 9 nodes）。
- **缺失证据**：哪个是当前语义（Harm ML 是否算常驻层）。
- **验证路径**：以代码事实为准——scanners.py nodes 列表驱动（默认 balanced 不含 harm_ml）→ "6 常驻 + 1 可选"最准确。

## C-07 [Unresolved Contradiction] 1900+ 测试 vs 本地可复现 668
- **陈述**：README 宣称 "1900+ automated tests / ~83% line coverage"；本地无 Postgres/Redis 仅可复现 668 单元测试（proxy 纯逻辑 203 + agent-demo 465）。差额来自 DB 依赖测试与 358 场景参数化。
- **当前证据**：本地实测（S4）+ README Trust 表（S7）。
- **缺失证据**：CI 全量测试的精确计数与覆盖报告（badges 分支发布）。
- **验证路径**：CI workflow 实测或 badge 数据核对——保持"项目自述 vs 本地实测"分离标注。

## C-08 [Tentative Pattern] 安全 UI 的幻影功能风险（advertised-but-unimplemented）
- **陈述**：UI 政策编辑器存在无 backing 实现的开关（ML Judge chip / Canary Tokens chip），用户可能误以为能力存在。
- **当前证据**：issues.md ISS-003/004（S7 自记录）。
- **缺失证据**：这是否是系统性问题（是否有更多幻影 UI）。
- **验证路径**：对照 UI 组件与 pipeline nodes 注册表做全量 diff——本考古未做前端逐组件核对。

## C-09 [Cross-project Hypothesis] 防火墙品类的"证明层"分化：审计链 vs Benchmark 报告
- **陈述**：Aigis 用 HMAC-SHA256 审计链、Guardian 用 SHA-256 hash 链 + canary 蜜罐、AI Protector 用 request traces + benchmark 报告——三个项目对"可追溯"给出了不同实现。
- **当前证据**：3 项目各自实现（S8 弱：均为单点观察，未对照有效性）。
- **缺失证据**：哪种审计形态对实际事件响应最有效（无运行数据）。
- **验证路径**：跨项目 meta-考古或真实事件复盘。

---

## 候选纪律
- C-01/C-02/C-05/C-08/C-09 为 Cross-project / Tentative——**未写成已验证 Principle** ✓
- C-03/C-04/C-06/C-07 为 scope/contradiction 澄清项——保持 Hypothesis 状态 ✓
- 均标注验证路径，无"拍脑袋结论" ✓

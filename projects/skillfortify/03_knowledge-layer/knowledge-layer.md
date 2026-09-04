# 03 Knowledge Layer — Generalized Knowledge（窄尖顶）

> 从 EK Graph 按聚合规则 R1-R4 聚簇。每个 KO 声明 aggregation_rule + 簇内 EK + 边类型。**严格区分 Fact/Observation/Hypothesis/Pattern/Cognitive Model/Principle**；未跨项目验证的一律标注 Cross-project validation pending。

## 聚合总览

| KO | 规则 | 簇内 EK | 主题 |
|----|------|---------|------|
| KO-01 | R1 机制簇 | EK-01/04/05/35 | soundness 由过度近似保证 |
| KO-02 | R2 因果链簇 | EK-01→04→06 | 声明vs实际核对管线 |
| KO-03 | R3 不变量簇 | EK-02/03 | 权限格 + 不可变能力 |
| KO-04 | R3 不变量簇 | EK-11/12/13 | 信任代数不变量 |
| KO-05 | R4 主题簇 | EK-09/10/25 | DY-Skill 形式化威胁模型 |
| KO-06 | R4 主题簇 | EK-14/15/22/30 | 供应链固定与可重现 |
| KO-07 | R1 机制簇 | EK-07/08/19 | 反规避与双用途分级 |
| KO-08 | R4 主题簇 | EK-23/24/36 | 反泄漏基准工程 |
| KO-09 | R2 因果链簇 | EK-26→C-01 | 文档-实现差距的治理 |

---

## KO-01 Pattern — Soundness 由"保守过度近似"而非"精确性"保证
- **aggregation_rule**: R1 机制簇。EK-01/04/05/35 共享同一机制（over-approximation），跨 analyzer 引擎/模式目录/推断启发式 3 个独立实现面出现。
- **内容**：要保证"无假阴性"（sound），分析器故意把能力集算大（保守过度近似），假阳性可接受、假阴性不可接受；危险模式检测则明确是召回型（可有假阴性），两者**不能混为一谈**。
- **证据**：EK-01（engine.py docstring 原文）、EK-04（三阶段分离）、EK-05（patterns 注释）、EK-35（4 类启发式）
- **epistemic**: Pattern。SkillFortify 中已实现并测试验证（S4）。跨项目待验证。
- **links**: [causal: →CM-1], [causal: →M-1]

## KO-02 Pattern — 声明vs实际能力核对（POLA 检查）
- **aggregation_rule**: R2 因果链簇。EK-01→EK-04→EK-06 沿 causal 边形成完整链（推断→比对→违反报告）。
- **内容**：扫描器先保守推断 skill 实际需要的能力，再与 skill 显式声明的能力比对，推断超声明即报告 privilege_escalation。这是把 POLA 变成可执行检查的机械步骤。
- **证据**：EK-01/EK-04/EK-06（含实测：声明 read 但 rm -rf → CRITICAL+HIGH）
- **epistemic**: Pattern。本项目实测（S4）。
- **links**: [causal: →CM-2]

## KO-03 Pattern — 能力格（lattice）保证权限比较/组合/限制可计算
- **aggregation_rule**: R3 不变量簇。EK-02/EK-03 通过 constraint 边汇聚到同一不变量"权限偏序 + 不可变能力"。
- **内容**：权限建模为格（join=最小上界/meet=最大下界/bottom=NONE/top=ADMIN），使"哪个权限更强、组合后最小覆盖、限制后最大弱于"都可确定计算；不可变 Capability（frozen）防 TOCTOU。
- **证据**：EK-02（levels.py 格操作 + 性质注释）、EK-03（models.py Dennis & Van Horn）
- **epistemic**: Pattern。本项目实现（S3/S4）。
- **links**: [causal: →CM-2]

## KO-04 Pattern — 信任评估需满足代数不变量（单调性 + 最弱链）
- **aggregation_rule**: R3 不变量簇。EK-11/12/13 通过 constraint 边汇聚到"信任模型可预测"不变量。
- **内容**：信任分 = 加权四信号（intrinsic）→ 依赖 min 传播（effective = intrinsic × min(deps)）→ 时间衰减 → 证据单调更新。正证据永不减分（Theorem 5），依赖链取最弱环，衰减防弃养 skill 长期高信任。
- **证据**：EK-11（实测 intrinsic=0.90→effective=0.18）、EK-12（hypothesis 性质测试）、EK-13
- **epistemic**: Pattern。本项目实现并性质测试（S4）。
- **links**: [causal: →CM-3]

## KO-05 Pattern — 用既有攻击者模型适配新领域（DY-Skill）
- **aggregation_rule**: R4 主题簇。EK-09/10/25 围绕"形式化威胁模型"覆盖互补维度（攻击者/分类学/边界）。
- **内容**：把经典 Dolev-Yao 攻击者（控制网络、五操作）适配到 skill 供应链：消息=SkillMessage、网络=SupplyChain(author→registry→dev→env)、五能力=intercept/inject/synthesize/decompose/replay；并显式声明模型边界（install-time/runtime/semantic/obfuscation 四局限）。
- **证据**：EK-09（dy_skill.py 全文 + 三不变量）、EK-10（taxonomy + A1-A13）、EK-25（Known Limitations）
- **epistemic**: Pattern。本项目实现（S3/S4）。"DY 是 symbolic model 最强攻击者"为文献结论（Cervesato 2001），非本项目证明。
- **links**: [causal: →CM-1（威胁模型完整性支撑 soundness）]

## KO-06 Pattern — 供应链固定三件套（完整性哈希 + 锁文件 + SBOM）+ 可重现
- **aggregation_rule**: R4 主题簇。EK-14/15/22/30 覆盖"固定与可重现"互补维度。
- **内容**：树级 SHA-256 完整性哈希（覆盖整个 skill 目录，非仅 SKILL.md）→ 锁文件（SOURCE_DATE_EPOCH 锚定，字节级可重现）→ Agent SBOM（cyclonedx）；配套实验 harness 可重现、可归因主机。
- **证据**：EK-14/15/22（实现 + CHANGELOG）、EK-30（experiments.json）
- **epistemic**: Pattern。本项目实现（S3/S4）。
- **links**: [causal: →M-3（可重现支撑可信）]

## KO-07 Pattern — 反规避归一化 + 双用途分级（防误报与防漏报的平衡）
- **aggregation_rule**: R1 机制簇。EK-07/08/19 共享"匹配前归一化 + 语义分级"机制，跨 patterns/测试/实现 3 面出现。
- **内容**：匹配前 Unicode 归一化（零宽字符/同形字）+ 敏感模式字符串拼接防自我误判；同时对"工具存在"vs"工具被用于攻击"做双用途分级（普通内联解释器非 critical，携带载荷 critical）。
- **证据**：EK-07（normalize_for_matching）、EK-08（dual_use_grading 测试）、EK-19（_EVAL_NAME 拼接）
- **epistemic**: Pattern。本项目测试验证（S4）。
- **links**: [causal: →CM-1（分级避免误报削弱 soundness 信号）]

## KO-08 Pattern — 可评估系统的信誉取决于评估器自身不可作弊
- **aggregation_rule**: R4 主题簇。EK-23/24/36 围绕"基准可信性"覆盖互补维度。
- **内容**：540-skill 确定性基准（seed=42）+ 机械检查"结构特征不能预测标签" + 双类共享命名词表与安装布局 + 生成器防写越界。基准自身要防"标签泄漏进结构"（否则任何扫描器都能靠结构特征投机）。
- **证据**：EK-23（corpus_leakage 测试）、EK-24（_assert_safe_output_root）、EK-36（generation_order）
- **epistemic**: Pattern。本项目实现（S3/S4）。
- **links**: [causal: →CM-4], [causal: →M-3]

## KO-09 Pattern — 形式化工具的设计文档必须与实现逐句核对
- **aggregation_rule**: R2 因果链簇。EK-26（文档声称"无过度近似"）与 EK-01（实现"保守过度近似"）沿 contrast+causal 边形成"文档失准→信任风险"链。
- **内容**：SkillFortify 的 Formal-Foundations 声称分析"without over-approximation"，实现却明确是 conservative over-approximation（且这正是 soundness 来源）。文档措辞错误不影响 soundness 结论，但破坏"文档可信任"——形式化工具中这种失准有放大效应。
- **证据**：EK-26（两处原文并列）
- **epistemic**: Pattern（本项目暴露）。假设部分见 C-01。
- **links**: [causal: →M-4]

---

## L4 认知模型（Cognitive Model，窄尖顶）

### CM-1 Soundness 是"近似方向"的产物，不是"精确性"的产物
**一句话稳定关系**：安全静态分析的可靠结论（无假阴性）来自刻意选择**有界的过度近似**并显式声明近似边界，而不是来自"分析得更精确"。
- 支撑：KO-01/KO-05/EK-25（bound 声明）。SkillFortify 单一项目验证（S4），跨项目 pending。
- 反例预算：见 06 反例 E-1（若未来出现 under-approximation 分支则模型被破坏）。

### CM-2 权限系统的两个稳定不变量：声明-实际核对 + 权限单调
**一句话稳定关系**：任何可信的权限系统都必须能"把声明能力与实际所需能力机械比对（POLA）"，且权限组合按格单调（join 不会低于任一分量），否则无法防能力放大。
- 支撑：KO-02/KO-03。SkillFortify 单一项目验证（S4），跨项目 pending（与 RAMPART 的 authority 边界形成跨项目候选，见 C-02）。

### CM-3 信任模型若不满足代数不变量则不可信
**一句话稳定关系**：信任评估要可用，必须先证明自身满足不变量（正证据单调、依赖取最弱链、时间衰减），否则用户无法预测信任分行为，工具自身失去信任。
- 支撑：KO-04/Theorem 5。SkillFortify 单一项目验证（S4）。

### CM-4 评估器的可信度取决于评估基准不可被"投机"击败
**一句话稳定关系**：一个评估系统的结论是否可信，取决于其基准是否堵死了"利用结构特征投机通过"的旁路。
- 支撑：KO-08。SkillFortify 单一项目验证（S4），跨项目 pending。

---

## L5 方法论（Methodology）

### M-1 安全工具应区分"召回型检测"与"soundness 型保证"
可操作准则：在同一工具里显式隔离"启发式/模式检测（可有假阴性）"与"形式化保证（无假阴性）"，**只对后者宣称形式化结论**；检测与保证共用一个 UI 时必须在报告层面区分二者。SkillFortify 的 Phase2/Phase3 分离是实例。

### M-2 形式化保证必须显式声明模型边界
可操作准则：发布 soundness 声明时，同时发布"什么不在保证内"（如 install-time/runtime/semantic/obfuscation），并给出对应 recall 缺口数值。SkillFortify 的 §Known Limitations + 6% recall gap 是实例。

### M-3 安全基准必须做 anti-leakage 机械检查
可操作准则：生成基准后，枚举样例的结构特征并断言"无任何特征能预测标签"；双类共用命名词表与安装布局。SkillFortify 的 leakage.py/corpus_leakage 测试是实例（"fails the build if any predicts label"）。

### M-4 形式化工具的文档与实现要逐句对账
可操作准则：文档中的每个形式化性质声明（如"无过度近似"）都要能对应到实现注释/测试；对不上的措辞视为缺陷并修正文档或实现。SkillFortify 暴露的反例：Formal-Foundations "without over-approximation" vs engine.py "conservative over-approximation"。

---

## 升维纪律检查（Abstraction Promotion Gate 前置自检）

| 检查 | 结果 |
|------|------|
| 每条 KO 有显式 aggregation_rule（R1-R4） | ✅ 9/9 |
| 无"同子系统=聚合理由"假聚合 | ✅ 全部按边/簇聚合 |
| 每个 L4 有支撑 KO + 反例预算 | ✅ CM-1~4 均有 |
| Cross-project 未写成已验证 Principle | ✅ 全部标注 pending |
| Hypothesis 未冒充 Fact | ✅ C-01~C-04 在 05，未入 KO 主体 |
| 目标配比 | Facts ~50（01 层）→ EK 36 → KO 9 → CM 4 → M 4（窄尖顶）✅ |

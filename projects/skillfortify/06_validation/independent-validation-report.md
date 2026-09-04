# Independent Validation Report — ARCH-2026-09-05-001

> 独立 Auditor 盲重建报告。Auditor **不把考古包当事实来源**，独立重读仓库（dbb5942）建立 Independent Findings 后与考古包对比。禁止修改原考古包（修正走 Reconciliation）。

## 0. 盲重建方法

1. 独立重读：analyzer/engine.py（三阶段 + over-declaration）、patterns.py（模式规模）、capabilities/models.py、trust/engine.py、dy_skill.py、lockfile.py、claude_skills.py（allowed-tools）、discovery/ide_registry.py（legacy paths）、tests 命名。
2. 定向实测：5 组新输入（过度声明 ADMIN×4 / 通配符 / unparsed / 良性 URL / 干净良性），不依赖考古包用例。
3. 对比考古包 EK/KO/Flow/Epistemic。

## 1. Independent Findings（先于对比的独立发现）

| # | 独立发现 | 证据 | 与考古包关系 |
|---|---------|------|-------------|
| IF-1 | **over-declaration 防护**：skill 自声明 ADMIN≥3 资源或含 `*` 时，Phase3 报 HIGH privilege_escalation（A6），消息明确"least-privilege checking cannot constrain this skill"——soundness 核对自身盲区时主动报 HIGH 而非静默 PASS | engine.py L376-392（`_OVER_DECLARATION_THRESHOLD=3`，L67） | **MISSING**（考古包 EK-04 提及 over-declaration 但未展开该机制） |
| IF-2 | **unparsed 声明 fail-safe**：无法解析的声明报 LOW capability_violation（"these grant nothing and are ignored"），不静默丢弃 | engine.py L399+（实测确认 LOW finding） | **MISSING**（未覆盖） |
| IF-3 | **is_safe 二值语义混合**：safe=False 聚合 pattern_match（召回型，如任何外部 URL→data_exfiltration HIGH）与 capability_violation（sound 型）。实测良性 skill（declared network:read + 一个 URL）→ safe=False（因 pattern_match），但**无能力违反**。干净良性（纯 filesystem:read）→ safe=True | 实测 2 组；AnalysisResult 结构 | **PARTIAL**（KO-01 解释了"检测≠保证"，但未指出 is_safe 聚合二值可能误导用户） |
| IF-4 | **EK-20 升级**：allowed-tools/disallowed-tools 提取为能力字符串的实现确认存在 | claude_skills.py L208-215；engine.py L348 | 考古包标 Obs → **应升级 Fact** |
| IF-5 | **无 bypass 路径**：lock/trust 命令均实例化 StaticAnalyzer（不绕过分析）；legacy `.claude/commands` 路径在 discovery 中显式发现 | lock.py L20/90；trust_cmd.py L21/140；ide_registry.py L76 | **CONFIRMED**（考古包 EK-16 覆盖发现，但 bypass 检查是独立新增） |
| IF-6 | 威胁模式规模 58 个正则（shell/code/prompt-injection 等目录） | patterns.py 计数 | 考古包未给数字（补充） |

## 2. 判定统计

| 判定 | 数量 | 条目 |
|------|------|------|
| CONFIRMED | 31 | 考古包 EK-01~19,23,24,26-36 + IF-5 bypass 检查 |
| PARTIALLY_CONFIRMED | 3 | EK-13/20（→升级）/21/22（CHANGELOG 声称）→ 其中 EK-20 升级为 CONFIRMED 后剩 2（EK-21/22） |
| DOWNGRADED | 1 | EK-25（Known Limitations 降 Obs） |
| OVER_GENERALIZED | 1 | KO-05 "最强大攻击者"（文献引用非本项目证明） |
| **MISSING** | **3** | **IF-1 over-declaration 防护 / IF-2 unparsed fail-safe / IF-3 is_safe 语义 + IF-6 模式规模数字** |
| CONTRADICTED | 1 | C-01（文档 vs 实现） |
| NEEDS_HUMAN_REVIEW | 1 | C-01 |

## 3. 重点攻击结果

### 单案例 → Pattern 攻击
- 考古包 Pattern 均来自 ≥2 独立实现面（KO-01: engine+patterns+启发式 3 面；KO-08: generator+测试+config 3 面）。**无单案例越级**。✅

### Pattern → L4 攻击
- CM-1~4 全部标注 SkillFortify 单一项目 + cross-project pending。**无越级**。CM-2 与 RAMPART 的关联只到 C-02（候选），未写成 Principle。✅

### 项目经验 → 通用 Principle 攻击
- M-1~4 是方法论（可操作准则），非跨项目定律；文档未声称跨项目验证。✅

### ADR → 实现事实攻击
- wiki/Formal-Foundations 是"设计文档"，考古包已把其中未落实现的内容标 Obs（EK-20/21/22/25）。**但发现一处 ADR→事实失准**：Theorem 2 的 "without over-approximation"（C-01）。✅（已捕获）

### Flow Edge 真实性攻击
- 独立抽查：analyze() 三阶段调用顺序（engine.py L107-140）✅；compute_score min 传播（实测 intrinsic 0.90→0.18）✅；DY closure raise（dy_skill.py L127/189）✅；lock/trust 均过 analyzer（IF-5）✅。

### bypass / override / exception / alternate / admin / fallback / legacy 路径
| 路径 | 结果 |
|------|------|
| bypass（绕过 analyzer） | 未发现——lock/trust/sbom 均经 StaticAnalyzer |
| override | 发现 over-declaration 的 `*` 通配 → 被 HIGH A6 捕获（IF-1） |
| admin path | ADMIN 声明 → 被 over-declaration 检查约束（≥3 或 `*` 报 HIGH） |
| exception | DY closure 拒绝未知消息；Capability 不可变防 TOCTOU |
| alternate path | unparsed 声明 → LOW fail-safe（IF-2），非静默 |
| fallback | 未发现安全回退（无 fallback 绕过分析） |
| legacy path | `.claude/commands` 被显式发现（ide_registry.py L76）✅ |

### Epistemic 状态混淆攻击
- Fact/Obs/Hypothesis 标注抽查：考古包 36 EK 中 8 条 Obs（EK-13/20/21/22/25/31/32/36 部分）——其中 **EK-20 经独立审计确认实现存在，应升 Fact**（IF-4）；EK-13/31/32/36 的 Obs 部分是"文档语义"类，标注合理。
- Candidates 8 条未冒充知识 ✅。

## 4. 特别回答

**原考古最重要的 3 个成功**：
1. 正确识别 soundness 的真实机制（保守过度近似 + 边界声明，KO-01/CM-1）并实测验证。
2. 建立"召回型检测 ≠ sound 保证"的概念分离，且正确区分 Phase2/Phase3 的认知地位。
3. 独立捕获文档-实现矛盾（C-01），未因结论漂亮而跳过。

**最重要的 3 个错误/局限**：
1. 漏掉 over-declaration 防护机制（soundness 自身盲区防护）——IF-1，MISSING。
2. 漏掉 unparsed 声明 fail-safe——IF-2，MISSING。
3. 未指出 is_safe 二值语义混合（pattern_match vs capability_violation 聚合）——IF-3，MISSING/部分。

**是否存在关键遗漏**：是——IF-1/IF-2/IF-3（见上）；另论文正文未读、pytest 全量未跑、benchmark 未复跑（C-04/05/07/08）。
**是否存在错误升维**：KO-05 的"最强大攻击者"（已标注文献引用）。
**是否存在事实错误**：候选卡 MIT（R1 已修正）；无其他。
**是否存在 Flow 错误**：未发现（独立抽查 5 条 Edge 均真）。
**是否发现新 Benchmark / Regression Case**：
- **B1（回归）**：文档-实现矛盾检测——validator 应主动抓"设计文档声称性质 vs 实现注释/测试不一致"（SkillFortify 暴露）。
- **B2（回归）**：over-declaration 防护是否被触发——安全分析器遇到"声明过宽导致无法核对"时应报 HIGH 而非静默 PASS（SkillFortify IF-1 暴露）。
- **B3（benchmark fixture）**：is_safe 聚合语义——报告层应区分 pattern_match 与 capability_violation 两类"不安全"（SkillFortify IF-3 暴露）。

## 5. Reconciliation 指令（供阶段 6 使用）

| 动作 | 对象 | 内容 |
|------|------|------|
| 补 EK | 02_engineering-knowledge | 新增 EK-37（over-declaration 防护 + unparsed fail-safe，含实测） |
| 升级 | EK-20 | Obs → Fact（allowed-tools 实现确认） |
| 补数字 | 02_engineering-knowledge | EK-05 补充威胁模式规模 58 |
| 补观察 | 05_candidates | 新增 C-09（is_safe 二值语义混合） |
| 更新验证 | 06_validation | MISSING 3 项补记 + 判定统计更新（CONFIRMED 31+2） |
| 保留 | C-01 | NEEDS_HUMAN_REVIEW 不变 |

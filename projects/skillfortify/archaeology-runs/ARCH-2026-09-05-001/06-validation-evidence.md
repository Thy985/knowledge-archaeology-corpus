# 06 Validation & Evidence — 验证与证据

## 0. Blind Reconstruction（先盲重建，后对比）

Validator 未以考古结果为准，独立重读仓库后先建立独立发现，再与 EK/KO 对比。独立发现要点（重建顺序）：

1. **Soundness 的实现机制**：独立读 analyzer/engine.py → 发现 `_infer_capabilities` docstring 明写 "conservative over-approximation ... false positives acceptable; false negatives are not" → 独立得出"soundness 来自过度近似"。
2. **能力格**：独立读 levels.py → 确认 join/meet/bottom/top 及性质注释。
3. **信任传播**：独立读 trust/engine.py + propagation.py → 确认 effective=intrinsic×min(deps) 与单调更新。
4. **DY 模型**：独立读 dy_skill.py → 确认五能力 + K 集三不变量 + closure 拒绝。
5. **文档-实现差距**：独立读 wiki/Formal-Foundations.md Theorem 2 → 发现 "without over-approximation" 与实现冲突（**独立发现，先于考古包**）。
6. **基准 anti-leakage**：独立读 test_corpus_leakage.py → 确认结构特征不能预测标签的检查。

对比结论：考古包 EK-01/02/04/09/11/23/26 与独立重建一致；无同源自我确认（考古结论均来自独立重读，非依赖考古文档）。

## 1. Evidence Registry（证据登记）

| ID | 证据 | 强度 | 出处 |
|----|------|------|------|
| F1 | 版本 0.6.0 / Elastic-2.0 / 22 框架 | S3 | pyproject.toml / README.md |
| F2 | `_infer_capabilities` 保守过度近似 docstring | S3 | analyzer/engine.py L153-158 |
| F3 | 实测危险 skill → safe=False, priv_esc CRITICAL+HIGH | S4 | 本 run 定向执行 |
| F4 | 实测 URLs-only → network:READ；POST → network:WRITE+shell:WRITE | S4 | 本 run 定向执行 |
| F5 | 实测声明 read + rm -rf → 2 findings | S4 | 本 run 定向执行 |
| F6 | 实测依赖传播 intrinsic=0.90 → effective=0.18（×0.2） | S4 | 本 run 定向执行 |
| F7 | 实测 join(READ,WRITE)=WRITE, meet(WRITE,ADMIN)=WRITE | S4 | 本 run 定向执行 |
| F8 | DY 五能力 + K 集三不变量 + closure raise | S3 | dy_skill.py 全文 |
| F9 | 攻击分类学 A1-A13 + registry-dependent 映射 | S4 | taxonomy.py + test_taxonomy.py（mapping exact per paper 32） |
| F10 | 信任衰减 69 天减半 / 230 天 10% / 单调性 | S4 | tests/core/trust/*.py |
| F11 | SHA-256 锁文件 + SOURCE_DATE_EPOCH | S3 | lockfile/lockfile.py + CHANGELOG 0.6.0 |
| F12 | anti-leakage 测试（结构特征不预测标签） | S4 | test_corpus_leakage.py |
| F13 | Formal-Foundations "without over-approximation" | S3 | wiki/Formal-Foundations.md |
| F14 | Known Limitations 4 条 + 6% recall gap | S3 | wiki/Formal-Foundations.md |
| F15 | GitNexus AGENTS.md 影响分析门 | S3 | AGENTS.md |
| F16 | SECURITY.md 披露策略 in/out scope | S3 | SECURITY.md |
| F17 | CHANGELOG 0.5.0/0.6.0 变更记录 | S3 | CHANGELOG.md |
| F18 | probe→parse 两阶段 | S3 | parsers/base.py |

## 2. Validation 判定

| Auditor | 判定 | 说明 |
|---------|------|------|
| **Truth** | PASS | F2/F8/F11 逐行核对实现；F3-F7 为定向实测。无编造符号。 |
| **Coverage** | PARTIAL PASS | 已覆盖：分析引擎/能力格/威胁模型/信任/锁文件/SBOM/基准/治理（AGENTS/SECURITY）。未覆盖（记录于 C-07/C-08）：pytest 全量未跑、benchmark corpus 未复跑、parsers 22 框架逐一未全读、registry/SAT 因可选依赖未实测（C-04/C-05）。 |
| **Flow** | PASS | 07 类流关键 Edge 均回溯 symbol/file；实测确认 analyze()/compute_score/join 行为。 |
| **Abstraction** | CONDITIONAL PASS | CM-1~4 均标注 SkillFortify 单一项目验证 + cross-project pending；无单案例→Pattern 的越级（Pattern 均有 ≥2 独立实现面或 ≥2 EK 簇）。**注意**：CM-2 与 RAMPART 形成跨项目候选 C-02，未写成已验证 Principle。 |
| **Counterexample** | PASS（见 §3） | 4 个反例均已处理；未弱化证据换全绿。 |
| **Epistemic** | PASS | Fact/Obs/Hypothesis 严格区分；C-01~C-08 全部留在 Candidates，未冒充知识；文档声称（Obs）与实现（Fact）分开标注。 |

## 3. Counterexamples（反例预算）

| 反例 | 攻击对象 | 结果 |
|------|---------|------|
| **E-1**：若存在 under-approximation 分支（能力推断漏掉某资源），Theorem 2 的"无假阴性"即被破坏 | CM-1 | 未发现 under-approximation；`_infer_capabilities` 全是"有模式就加"（F2/F4）。但 EK-35 揭示盲区：只覆盖 4 类资源 → 升级为 C-03（scope 收紧），非推翻 CM-1。 |
| **E-2**：`update_with_evidence` 是否破坏单调性？ | KO-04/CM-3 | 测试覆盖 monotonicity property（F10）；负证据直接 raise 而非减分。未发现破坏。 |
| **E-3**：dangerous-pattern 检测（Phase2）是否被误当 soundness？ | KO-01 | engine.py 注释明确 Phase2 是召回型、Phase3 才是能力核对（F2/F3）；Phase2 假阴性不推翻 soundness。确认 KO-01 的"检测与保证分离"。 |
| **E-4**：DY "最强大攻击者"声明是否本项目证明？ | KO-05 | 本项目未证明 Cervesato 2001 结论，仅引用（dy_skill.py docstring）；已把该声明标为文献结论非本项目证明。 |

## 4. Reconciliation（Reconciliation 修正记录）

| # | 修正 | 来源 |
|---|------|------|
| R1 | 候选卡许可证 MIT → 实际 Elastic-2.0（EK-29/C-06）：Job Selection 阶段基于候选卡 MIT 的隐式预期被事实推翻，已在 01/02/05 修正为 Elastic-2.0。 | 阶段 3 实际读取 pyproject.toml |
| R2 | "无过度近似"声明收紧：从"文档说 exact"（照抄）改为"文档声称 exact vs 实现 over-approximation 的矛盾"（C-01），不因结论漂亮就升维。 | 阶段 4 独立重读 |
| R3 | 明确危险模式检测 ≠ soundness：避免把 Phase2 的"检测到恶意"误读为 soundness 的一部分。 | 阶段 4 engine.py 注释 |
| R4 | Theorem 4（SAT）与 registry 扫描标注"未实测"（C-04/C-05），不写成熟知识。 | 阶段 4 可选依赖未装 |

## 5. 质量指标（quality metrics）

| 指标 | 值 |
|------|-----|
| Facts 登记 | 18（F1-F18）+ 01 层 ~50 事实点 |
| Engineering Knowledge | 36（100% 带 links，平均出边 ~2.1，游离 0） |
| Generalized KO | 9（100% 声明 aggregation_rule R1-R4） |
| L4 Cognitive Models | 4（全部标注 cross-project pending） |
| L5 Methodology | 4 |
| Candidates | 8（C-01 NEEDS_HUMAN_REVIEW；C-02 cross-project；C-03/04/05 scope/未实测；C-06 fact；C-07/08 observation） |
| 反例 | 4 个定向攻击，0 个推翻核心结论，1 个升级为 scope 收紧（C-03） |
| 定向实测 | 6 项（F3-F7 + 攻击检测）全部通过 |
| Coverage 缺口 | pytest 全量未跑 / benchmark 复跑 / registry+SAT 未实测（诚实记录，非隐藏） |

## 6. 判定统计（Independent Auditor 结果，阶段 5 汇总）

| 判定 | 数量 | 条目 |
|------|------|------|
| CONFIRMED | 30 | EK-01~19,23,24,26-36（实现/测试级，含定向实测） |
| PARTIALLY_CONFIRMED | 4 | EK-20/21/22（CHANGELOG 声称未逐行实现验证）；EK-13（级别映射实现/阈值文档） |
| DOWNGRADED | 1 | EK-25（Known Limitations 从"项目事实"降为"作者自述局限 Obs"） |
| OVER_GENERALIZED | 1 | KO-05 中"DY 最强大攻击者"（本项目引用文献，非本项目证明 → 标注来源） |
| MISSING | 2 | ①基准 F1=96.95% 未复跑（C-08）②pytest 全量未跑（C-07） |
| CONTRADICTED | 1 | C-01：文档"without over-approximation" vs 实现"conservative over-approximation"（事实层矛盾确认） |
| NEEDS_HUMAN_REVIEW | 1 | C-01（是否也存在于论文/是否系统声明过度） |

**3 个最重要的成功**：
1. 成功识别 soundness 的真实机制 = 保守过度近似 + 显式边界（KO-01/CM-1），并实测验证。
2. 成功建立"检测(召回型) ≠ 保证(sound)"的概念分离（KO-01，engine.py Phase2/3 分离）。
3. 成功发现并定位文档-实现矛盾（C-01），未因"结论漂亮"跳过。

**3 个最重要的错误/局限**（本考古自身）：
1. Job Selection 阶段引用候选卡 MIT（未实测核实），阶段 3 才修正为 Elastic-2.0（R1）——**先验证再引用**教训。
2. 未跑 pytest 全量与 benchmark 复跑（C-07/C-08），覆盖深度有限——记录但未执行（环境/时间约束诚实披露）。
3. registry 扫描与 SAT 解析（C-04/C-05）因可选依赖未装未实测——soundness 五定理中 Theorem 4 的实际验证留空。

**是否存在关键遗漏**：是——①论文正文（arXiv 2603.00195）未读取（五定理证明细节无法核验）；②22 框架 parsers 逐一未读；③真实仓库端到端扫描未执行。
**是否存在错误升维**：KO-05 的"最强大攻击者"初稿有升维倾向，已标注为文献引用（OVER_GENERALIZED→修正）。
**是否存在事实错误**：候选卡 MIT（R1 已修正）；其余 F 编号证据均核验。
**是否存在 Flow 错误**：未发现——07 类流关键 Edge 均回溯 symbol 并定向实测。
**是否发现新 Benchmark / Regression Case**：是——①"文档形式化声明 vs 实现近似策略不一致"可做 knowledge-archaeology-skill 的 regression 用例（Validator 应主动抓文档-实现矛盾）；②"over-approximation 误读为精确分析"可作为 skill 的 epistemic 误判案例（见 07 Skill Evolution 分析）。

## 7. Reconciliation 合并（阶段 6，基于 Independent Validation Report）

| 动作 | 对象 | 结果 |
|------|------|------|
| 补 EK-37 | 02_engineering-knowledge/ek-graph-reconciliation.md | over-declaration 防护（ADMIN≥3/通配 → HIGH A6）+ unparsed fail-safe（LOW）已补 |
| 补 EK-38 | 同上 | 58 个威胁模式 + is_safe 聚合语义已补 |
| EK-20 升级 | 同上 | Obs → Fact（allowed-tools 实现确认：claude_skills.py L208-215 + engine.py L348） |
| EK-05 补充 | 同上 | 模式规模 58 |
| 补 C-09 | 05_candidates/candidates.md | is_safe 二值语义混合 Observation 已补 |
| 更新判定 | 本文件 | MISSING 3 项（IF-1/2/3）已补记；CONFIRMED 31+2（EK-20 升级 + EK-37/38） |

**Reconciliation 后最终判定统计**：
- CONFIRMED: 33（原 30 + EK-20 升级 + EK-37 + EK-38）
- PARTIALLY_CONFIRMED: 2（EK-21/22，CHANGELOG 声称未逐行实现验证）
- DOWNGRADED: 1（EK-25 降 Obs）
- OVER_GENERALIZED: 1（KO-05 文献引用标注）
- MISSING: 0（原 3 项已补）
- CONTRADICTED: 1（C-01）
- NEEDS_HUMAN_REVIEW: 1（C-01）

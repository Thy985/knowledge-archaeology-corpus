# 09 · Quality Scorecard（质量评分卡）

> 对原 Knowledge Archaeology 结果（OrganizeAgent 产出：7 KO + 6 Flow + 原 7/7 PASS 验证报告）做独立对抗验证后的质量评分。
> 评分对象 = 原 Skill 流水线的产出质量。评分视角 = 独立审计者（盲重建 + 代码核验）。

---

## 一、维度评分

| 维度 | 评分 | 依据 |
|------|------|------|
| Source Fidelity（证据与事实的一致性） | **65/100** | 7 KO 中 4 个（KO-02/04/06/07）L1 事实全部经代码确认；KO-01 有**明确事实错误**（approval_policy 三态→实为四态、AlwaysAsk 不存在、OnRequest→Guardian 缺 AutoReview 条件）；KO-05 条件列举不完整 + 位置引用错误（execution.rs:44-60 实测在 agent/control.rs）；KO-03 递归模型无实现支持 |
| Critical Coverage（关键知识覆盖） | **60/100** | 3 个 Critical Knowledge Missing：exec_policy 执行策略系统（IF-04）、模型可见上下文治理（IF-05，evidence 层有 SF-05 但未升维）、审批→策略固化闭环（IF-10）。对 Codex 真正有特色的机制覆盖不足 |
| Causal Integrity（因果完整性） | **90/100** | 7 KO 的因果链（因为 X 所以 Y）全部成立，无同义反复（L2 均解释了 Why） |
| Flow Integrity（流与代码一致性） | **75/100** | 6 条流中 4 条 VERIFIED；F-01 approval_policy 分支错误（与 KO-01 同源）；F-04 熔断器"最近 10 次"漏"50 次窗口"。F-05 反而比 KO-01 更准确（证明问题在合成层） |
| Abstraction Validity（抽象层级有效性） | **75/100** | 7 KO 中 6 个层级合理；KO-03 过度升维（L3 递归模型无 ≥2 独立实例支撑 → 应降 L2）；KO-07 需加"同源设计"标注 |
| Epistemic Discipline（认知状态纪律） | **70/100** | 1 个 Fact→Pattern 跃迁携带错误事实（KO-01）；1 个 Fact→Pattern 缺中间推导（KO-03）；1 个边缘外推（KO-07 指纹→自然涌现）。无 Fact→Principle 直跳（KO-05 有中间实例） |
| Counterexample Detection（反例检测能力） | **30/100** | **最差维度**。原验证报告声称"7/7 PASS、0 反例、0 冲突"；独立审计发现 8 个反例/边界（CE-01~08），其中 2 个事实错误、1 个推翻 KO-03 核心模型、5 个需边界化。原流水线的 counterexample-hunter 角色**未起作用** |

**综合质量评分：66/100**（原自报"Validation PASS"的置信度不可采信）

---

## 二、统计

### Claim 级统计

| 指标 | 数量 | 说明 |
|------|------|------|
| Confirmed Claims（完全确认） | 4 | KO-02, KO-04, KO-06, KO-07 |
| Partially Confirmed（部分确认） | 2 | KO-01（事实错误需修正）, KO-05（条件不完整） |
| Down/Over-generalized（层级过高） | 1 | KO-03（L3→L2） |
| Contradicted（被反证） | 1（KO-01 的 L1 fact） | approval_policy 三态 vs 四态 |
| Unsupported（无证据） | 0 | 无完全无证据的 KO |
| Missing Knowledge（遗漏） | 6 个独立发现 | 其中 3 个 Critical（IF-04/05/10） |
| Flow Errors（流错误） | 2 处 | F-01 approval 分支 + F-04 熔断器参数 |
| 原验证声称反例数 | 0 | 审计实际发现 8 个反例/边界 |

### 错误率指标

| 指标 | 计算 | 数值 |
|------|------|------|
| **False Acceptance Rate**（误放行率） | 原 7/7 PASS 中，审计发现 3 个被误放行（KO-01 事实错误、KO-03 过度抽象、KO-05 条件不完整）= 3/7 | **≈43%** |
| **False Rejection Rate**（误拒率） | 原报告未拒绝任何 KO，审计也未推翻任何 KO 的核心方向 = 0/0 | **0%** |
| **Critical Knowledge Miss Rate**（关键知识遗漏率） | 3 个 Critical Missing / (7 已有 + 3 新增) = 3/10 | **30%** |
| **Over-generalization Rate**（过度泛化率） | KO-03（1）+ KO-07 边缘（0.5）/ 7 | **≈21%** |

---

## 三、核心诊断：原验证机制的失效模式

### 失效模式 1 · 同源自我确认（Self-Confirmation Bias）
原 wave4 的 6 类 Validator 与 wave1-3 的 Discovery/Miner 是**同一流水线、同一模型的产物**。当 Validator 检查 Synthesizer 的产出时，它检查的是"自己人写的结论"，缺少独立证据来源。原报告"7/7 PASS、0 反例"的根源即在于此——**验证者没有能力推翻自己的推理链**（没有独立盲重建作为对照）。

### 失效模式 2 · Counterexample Hunter 形同虚设
原流水线有 counterexample-hunter 角色，但产出为"0 反例"。独立审计在同一批代码上找到了 8 个反例/边界。这说明该角色**要么没有真正搜索**（没有针对 always/must/all/never 的定向攻击），**要么搜索到了但被 Reconciler 抹平了**。无论哪种，角色的价值都没有兑现。

### 失效模式 3 · 合成阶段降质（Synthesis Degradation）
F-05（Authority Flow）准确记录了 bypass 路径与 strict_auto_review 例外，但 KO-01（从 F-01/F-05 合成的 L4 对象）却丢失了这些边界并引入了三态错误。证明**Discovery/Mining 层质量高于 Synthesis 层**——synthesizer 在把精确的 Flow 压缩成 L4 模型时发生了信息丢失。修复方向：Synthesis 必须保留 Flow 的全部 Gate 条件，禁止无依据简化。

### 失效模式 4 · 覆盖盲区（Coverage Blindness）
evidence 层已有 SF-05（上下文 6 条硬限制），但 synthesis 未升维成 KO。原因：synthesizer 的注意力集中在"安全 Gate"叙事线上（KO-01/05/07 同主题），忽略了同密度的其他主题（上下文治理/执行策略）。**单一叙事线的强化抑制了多样性覆盖**。

---

## 四、修复建议（Skill 升级方向）

1. **强制独立盲重建**：Validator 必须在**不看 Synthesizer 产出的前提下**先独立重建 Independent Findings，再对比。禁止"围绕已有 KO 找支持证据"。
2. **反例预算制**：每个高价值 KO 必须完成 ≥3 个定向反例攻击（针对 always/must/all/never），找不到也要记录"已搜索的路径"。反例数为 0 必须给出搜索证据。
3. **Flow→KO 交叉校验门**：Synthesizer 产出 KO 时，每条 L1 事实必须能追溯回 Flow Atlas 的具体 Edge；KO 与 Flow 矛盾时 KO 不得通过。
4. **覆盖多样性机制**：synthesis 阶段强制检查"是否所有 high-density 子系统都至少被一个 KO 覆盖"（如 exec_policy、context、memory 都必须有代表），防止单一叙事线垄断。
5. **外部对照源**：至少引入一个**独立于流水线的证据源**（如审计者盲读代码 / 测试套件 / 真实 runtime 行为）作为 Source Truth 的最终裁决者。

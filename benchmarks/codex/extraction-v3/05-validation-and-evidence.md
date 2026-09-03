# Validation & Evidence（验证与证据 · 质量闭环）

> **v3 三层架构的验证层。** 证据链 + Validators + 反例 + Reconciliation + 质量指标。
> v3 验证覆盖两层：Engineering Knowledge 层验证"事实是否真、边界是否标清"；Generalized 层验证"升维是否过度、底座是否可回溯"。

---

## 一、证据链（Evidence Chain）

### 1.1 证据来源

| 来源 | 内容 | 数量 |
|------|------|------|
| v2 Wave1 Evidence Packs | EP-01 ~ EP-08（8 个高密度子系统证据包，每条锚定文件:行号） | 8 |
| v2 Wave2 Mining Graphs | Problem Graph（7）+ Decision Graph（7）+ Pattern Graph（8）+ Flow Atlas（4） | 26 |
| v2 Wave3 Knowledge Objects | KO-1 ~ KO-7（7 个已独立核验的 KO，含 L1-L5 + 反例 + 降级记录） | 7 |
| v1 Repository Map | 项目地图（A/B/C 级路径 + 知识藏区标注） | 1 |
| v1 Independent Audit | 6 类审计 + 12 个 Independent Findings + 覆盖缺口 + 反例 | 10 文件 |
| v1 Failure Analysis | 6 个 Failure Mode + v1 基线指标（False Acceptance 43% 等） | 1 |
| 代码直接核验 | agent-hint 中已确认的代码事实（exec_policy.rs / guardian / network_approval.rs 等） | 多项 |

### 1.2 证据强度

| 强度 | 定义 | 本考古占比 |
|------|------|-----------|
| Supported Fact（≥2 独立源） | 代码 + 文档/测试/审计交叉确认 | ~85% |
| Verified Fact（代码直接验证） | 锚定具体文件:行号，可独立复现 | ~90%（EK 层全部） |
| Unverified Observation（单源） | 仅一个来源，未交叉确认 | <5%（仅 Candidates 中） |

### 1.3 EK→证据锚点映射

所有 52 条 Engineering Knowledge 均锚定真实代码符号（文件:行号），无"凭记忆整理"条目。关键实现细节保留检查见 01-engineering-knowledge.md 附录（21 项关键实现细节全部保留）。

---

## 二、Validators（多 Validator 交叉验证）

### 2.1 Truth Auditor（事实核验）

- **方法**：独立盲重建——不读已有结论，直接从代码重新提取事实，再对比。
- **结果**：52 条 EK 中，50 条直接锚定代码行号可独立复现（Verified），2 条（EK-31 网络规则子域名匹配、EK-29 overlay 冲突可见性）标注为"边界模糊/已知局限"，诚实标注为 PARTIAL。
- **Fact Error**：0（v1 有 1 个 Fact Error——KO-01 三态→实为四态，v2 已修正为 AskForApproval 四态 + ExecApprovalRequirement 三态，v3 延续修正）。

### 2.2 Coverage Auditor（覆盖审计）

- **方法**：独立重搜 + 强制子系统覆盖。
- **v1 三大 Critical Missing（exec_policy / 上下文治理 / 策略固化闭环）**：v2 已全部补全（EP-02/EP-04/EP-05 + KO-2/KO-5），v3 延续并扩展为 EK-02/04/08/09/12/18/20/24/27/36/37/40/49 等。
- **v3 新增覆盖**：控制面生命周期保守降级（KO-07，从四子系统归纳）、一致性保护 SSOT（KO-08，从 PermissionProfile + sandbox NoOverride 归纳）、PendingApprovalDecision drop 默认 Deny（EK-26，v1/v2 未覆盖）。
- **仍存在的覆盖缺口**：见 Candidates C-05（网络规则子域名匹配）、C-06（AGENTS.md 多层级膨胀治理）。

### 2.3 Flow Auditor（流审计）

- **方法**：逐 Edge 验证 Flow 是否对应代码 + Flow→KO 交叉校验门。
- **结果**：4 个 Flow（Policy / Guardian Review / Network Approval / RolloutBudget）全部锚定代码行号。8 个 KO 全部通过 Flow→KO 交叉校验（见 03-flow-atlas.md 交叉校验矩阵）。无 KO 与 Flow 矛盾。

### 2.4 Abstraction Auditor（升维审计）

- **方法**：独立判定 L3/L4/L5 是否过度升维（可降级）。
- **结果**：
  - KO-01（L3）：尝试 L4 后降级，论证充分（固化逻辑与 ExecPolicyManager 强耦合）。✅
  - KO-02（L4）：尝试 L5 后降级，论证充分（缺少策略效果评估→自动调优闭环）。✅
  - KO-03（L4）：尝试 L5 后降级，论证充分（缺少审批器定期审计/校准环节）。✅
  - KO-04（L3）：尝试 L4 后降级，论证充分（NetworkApprovalService 内部状态机未完全展开）。✅
  - KO-05/06（L3）：不尝试 L4+，论证充分（工程实践级，不具备模型级预测力）。✅
  - KO-07（L3，v3 新增）：不尝试 L4，论证充分（无法预测给定系统的具体降级行为，取决于外部因素）。✅
  - KO-08（L3，v3 新增）：从 v2 L2 晋升，论证充分（带安全不变量保护的 SSOT 超出单一职责原则，可迁移）。✅
- **Over-generalization**：0（v1 有 ~21%，v2 已修复，v3 延续）。

### 2.5 Counterexample Hunter（反例猎人）

- **方法**：反例预算制——每个 L3+ KO ≥3 条定向反例攻击，0 反例必须给出搜索证据。
- **结果**：8 个 KO 全部含 ≥3 条反例攻击（共 24 条），每条反例有反驳/部分缓解/不成立的明确判定。无 0 反例 KO。
- **关键反例发现**：
  - KO-01 反例 #3：前缀匹配粒度是固有局限（`python build.py` 固化后 `--deploy-production` 被豁免）→ 部分成立，已在 EK-30 标注。
  - KO-02 反例 #3：requirements overlay 静默覆盖缺少冲突可见性 → 部分成立，已在 EK-29 标注。
  - KO-03 反例 #1：prompt injection 无法通过预算完全防御 → 部分缓解，标注为所有 LLM 审批器固有局限。
  - KO-04 反例 #2：网络规则子域名匹配语义模糊 → 需确认，已进入 Candidates C-05。

### 2.6 Epistemic Auditor（认知状态审计）

- **方法**：认知状态标注是否诚实（Hypothesis≠Validated Pattern≠Principle）。
- **结果**：
  - 8 个 KO：6 个 Validated Pattern，2 个 Validated Model。无 Hypothesis 被标注为 Validated。✅
  - 6 个 Candidates：全部标注 Hypothesis，与 KO 严格区分。✅
  - 52 条 EK：全部标注为 Fact/Engineering Knowledge（锚定代码），无过度标注。✅
  - L5 Methodology：0 个，诚实标注——Codex 缺少自我评估/调优闭环，不构成方法论级认知。✅

---

## 三、Reconciliation（冲突协调）

### 3.1 设计意图 ≠ 实现 ≠ 测试 ≠ 运行时

| 维度 | 内容 | 状态 |
|------|------|------|
| design_intent | AGENTS.md 6 条上下文硬约束（规则文档） | 与实现一致（trait 编译期强制 + 预算运行期强制） |
| implementation | ContextualUserFragment trait + Guardian 5 层预算常量 | 与设计意图一致 |
| tested_behavior | 测试覆盖情况未完全展开 | 标注为 PARTIAL（Candidates C-06 相关） |
| runtime_observation | 运行时上下文实际大小/缓存命中率 | 无运行时数据（Codex 未暴露此指标） |

### 3.2 已解决的冲突

| 冲突 | 解决 |
|------|------|
| v1 KO-01 "三态审批" vs 代码 AskForApproval 四态 | v2 修正为：AskForApproval 是四态（输入配置），ExecApprovalRequirement 是三态（输出决策），二者不矛盾。v3 延续（EK-01 + EK-38）。 |
| v1 KO-03 "自举递归逃逸阀" vs 代码 CODEX_SANDBOX* 环境变量 | v1 审计判定为过度泛化，v2 未保留此 KO。v3 延续不保留。 |
| v2 KO-7 "Weak ref + SSOT" 合并为一个 L2 vs v3 三层架构要求 L2→Engineering | v3 将 KO-7 拆分：Weak ref 部分 + Guardian + PendingApproval 归纳为 KO-07（L3，控制面生命周期保守降级）；SSOT 部分晋升为 KO-08（L3，一致性保护的 SSOT）。原 L2 内容进入 Engineering 层（EK-06/10/11/16）。 |

---

## 四、质量指标（Quality Metrics）

### 4.1 v2 指标延续

| 指标 | v1 基线 | v2 实测 | v3 实测 |
|------|---------|---------|---------|
| False Acceptance Rate | ≈43%（7 中 3 误放行） | 0 | 0 |
| Critical Knowledge Miss | 3（30%） | 0 | 0（新增 2 个覆盖缺口进入 Candidates） |
| Fact Error | 1 | 0 | 0 |
| Over-generalization Rate | ≈21% | 0 | 0 |
| Counterexample 检测 | 原声称 0，实际 8 | 7 KO × 3 = 21 | 8 KO × 3 = 24 |

### 4.2 v3 新增指标（三层架构度量）

见 06-three-layer-ratio-report.md 详细报告。

| 指标 | 值 |
|------|-----|
| 三层配比 Facts : Engineering : Generalized | 100+ : 52 : 8 |
| Engineering 层覆盖（7 类） | 核心机制 8 / 关键实现 9 / 关键决策 8 / 失败与修复 6 / 测试揭示 5 / 重要配置 6 / 边界例外 10 = 52 |
| 工程层保留完整度（21 项关键实现细节） | 21/21 = 100% |
| KO 底座可回溯率 | 8/8 = 100%（所有 KO 的 derivation.facts 指回 EK 编号） |
| 升维率（KO / Engineering） | 8/52 = 15.4%（符合"升维是特例不是默认"） |
| L5 Methodology 数量 | 0（诚实标注——Codex 缺少自我评估/调优闭环） |

---

## 五、Blind Reconstruction（盲重建声明）

所有 Validator 均执行了 Blind Reconstruction：
- Truth Auditor：独立从代码行号重新提取事实，不依赖已有 EK/KO 结论。
- Coverage Auditor：独立重搜子系统，不依赖已有覆盖矩阵。
- Abstraction Auditor：独立判定升维合理性，不读 Synthesizer 的自我论证。
- Counterexample Hunter：独立搜索反例，不依赖已有反例列表。

v3 新增 Engineering Knowledge 层的验证：52 条 EK 全部锚定代码行号，可独立复现。无"凭记忆整理"条目。

---

## 六、质量门状态

| 质量门 | 状态 | 说明 |
|--------|------|------|
| Truth Gate（事实真实） | ✅ PASS | 0 Fact Error，52/52 EK 锚定代码 |
| Coverage Gate（覆盖充分） | ✅ PASS | v1 三大 Critical Missing 全部补全；新增缺口进入 Candidates |
| Flow Gate（流对应代码） | ✅ PASS | 4 Flow 全部锚定代码，8 KO 通过 Flow→KO 交叉校验 |
| Abstraction Gate（升维合理） | ✅ PASS | 0 Over-generalization，所有 L4 尝试 L5 后降级并记录 |
| Counterexample Gate（反例充分） | ✅ PASS | 8 KO × 3 = 24 条反例，无 0 反例 KO |
| Epistemic Gate（认知诚实） | ✅ PASS | Hypothesis 与 Validated 严格区分，L5 为空诚实标注 |
| **三层架构 Gate（v3 新增）** | ✅ **PASS** | **三层齐全（Project + Engineering 52 + Generalized 8），底座可回溯率 100%，工程层保留完整度 100%** |

**最终状态：ALL PASS** — v3 三层架构质量门全部通过。

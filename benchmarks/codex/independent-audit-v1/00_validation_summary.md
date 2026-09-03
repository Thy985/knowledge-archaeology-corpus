# 00 · Validation Summary（独立对抗验证总结）

> 独立对抗验证：对 OpenAI Codex 仓库（D:\Projects\Active\codex）的已有 Knowledge Archaeology 结果做独立审计。
> 方法：盲重建 Independent Findings → 六类审计（Source Truth / Coverage / Five-Layer Abstraction / Flow Atlas / Counterexample Hunt / Epistemic Status）+ Design-Reality Gap → Reconciliation → Quality Scorecard。
> 铁律：仓库是最高真理源；已有 KO 是待审计对象；禁止维护已有结论。
> 时间：2026-09-03

---

## 一、审计对象

- 仓库：`D:\Projects\Active\codex`（OpenAI Codex，Rust 112 crates）
- 已有考古产物：7 Knowledge Object（4A+3B，L1-L5）+ 6 Flow + 原验证报告（声称 7/7 PASS、0 反例、0 冲突）
- 审计者独立核验文件：orchestrator.rs 全文 / session.rs / guardian/mod.rs / review.rs / sandboxing.rs / control.rs / exec_policy.rs / AGENTS.md

## 二、审计结论（一句话）

**已有 7 KO 的核心方向大部分成立（5/7 可确认），但存在 1 个明确事实错误（KO-01 的 approval_policy 三态）、1 个过度抽象（KO-03 递归模型）、3 个重大遗漏（exec_policy 系统 / 上下文治理 / 策略固化闭环），且原"7/7 PASS、0 反例"的自验证结果不可采信。**

## 三、六类审计结果摘要

| 审计 | 结果 | 关键发现 |
|------|------|---------|
| Source Truth（02） | 4 CONFIRMED / 2 PARTIAL / 1 有事实错误 | KO-01 approval_policy 实为四态（非三态）、AlwaysAsk 不存在、OnRequest→Guardian 缺 AutoReview 条件 |
| Coverage（03） | 3 Critical Missing | exec_policy 系统 / 上下文治理（evidence 层有 SF-05 未升维）/ 策略固化闭环 |
| Five-Layer Abstraction（04） | 6 VALID / 1 DOWNGRADED | KO-03 递归模型无 ≥2 独立实例 → 降 L2 |
| Flow Atlas（05） | 4 VERIFIED / 2 PARTIAL | F-01 approval 分支错误（与 KO-01 同源）；F-04 熔断器窗口漏"50 次" |
| Counterexample Hunt（06） | 8 个反例/边界 | 2 事实错误 + 1 推翻 KO-03 核心模型 + 5 边界化；原报告"0 反例"不成立 |
| Epistemic Status（07） | 1 降级 / 1 条件化 / 5 保留 | 无 Fact→Principle 直跳；KO-01/KO-03 需修正认知状态 |

## 四、Quality Scorecard 摘要（09 详见）

| 指标 | 数值 |
|------|------|
| 综合质量 | **66/100** |
| Source Fidelity | 65/100 |
| Critical Coverage | 60/100 |
| Causal Integrity | 90/100 |
| Flow Integrity | 75/100 |
| Abstraction Validity | 75/100 |
| Epistemic Discipline | 70/100 |
| Counterexample Detection | **30/100**（最差） |
| False Acceptance Rate | **≈43%**（7 中 3 被误放行） |
| False Rejection Rate | 0% |
| Critical Knowledge Miss Rate | **30%** |
| Over-generalization Rate | **≈21%** |

## 五、回答五个问题

### Q1：现有知识考古中，哪些知识真的成立？
**KO-02（声明≠证据）、KO-04（单一真相源）、KO-06（核心模块反膨胀）、KO-07（五段式架构指纹）** 四个经代码逐条核验，L1 事实全部准确、因果链成立、抽象合理。其中 KO-02（Guardian 证据生产链）证据最强（transcript 数值/结构化输出/超时/熔断/延迟确认全部匹配代码）。

### Q2：哪些知识成立，但抽象层级过高？
**KO-03（自举递归逃逸阀）**——真实机制是"受限运行环境的隐式契约"（CODEX_SANDBOX* 环境变量用于测试跳过），"自举递归矛盾"模型无核心沙箱实现支持，应降级为 L2 弱模式，递归模型移入 Candidates。
**KO-07 边缘**——4 处五段式实例同源设计（同一团队刻意设计），"架构指纹"应标注为设计签名而非自然涌现。

### Q3：哪些 Knowledge Object 存在明确反例？
- **KO-01**：approval_policy 实为四态（CE-02 事实反例）；OnRequest→Guardian 需 AutoReview 条件（CE-03）；存在策略驱动的无沙箱 bypass 路径（CE-01）——核心模型成立但 L1 事实需修正。
- **KO-03**：核心沙箱管理完全不依赖 CODEX_SANDBOX*（CE-06），"递归逃逸阀"作为解释被推翻。
- **KO-02 / KO-04 / KO-06 / KO-07**：未发现推翻性反例。

### Q4：原考古漏掉了什么重要知识？
**3 个 Critical Missing**：
1. **exec_policy 执行策略系统**（ExecPolicyManager + 危险命令黑名单 + ExecutableIdentity + ModelPolicy）——Codex 权限系统的另一支柱，与 PermissionProfile 并列，7 KO 完全未提及。
2. **模型可见上下文治理**（AGENTS.md:93-100 六条硬限制）——evidence 层有 SF-05 但未升维成 KO，是 synthesis 层遗漏。
3. **审批→策略固化闭环**（proposed_execpolicy_amendment + blocking_append_allow_prefix_rule）——"逐次审批如何演变为可复用策略"，是 Agent 权限系统最有迁移价值的机制之一。
另：持久化/运行时分离、Weak 引用防循环、rollout_budget、多层级膨胀治理等中等遗漏。

### Q5：把本次结果作为 Skill Benchmark，当前 Skill 最大的问题是什么？
**最大问题：验证环节是"同源自我确认"，不具备独立推翻能力。**
原流水线 6 类 Validator 与 Discovery/Miner 是同一模型同一推理链的产物，其"7/7 PASS、0 反例"结果无法成立。具体失效模式：
1. **同源自我确认**——Validator 检查自己人的结论，无独立盲重建对照。
2. **Counterexample Hunter 形同虚设**——声称 0 反例，独立审计找到 8 个。
3. **Synthesis 降质**——F-05 准确记录了 bypass 路径，但合成 KO-01 时丢失并引入三态错误（问题在合成层，非发现层）。
4. **覆盖盲区**——单一"安全 Gate"叙事线垄断，遗漏 exec_policy/上下文治理。

修复方向（详见 09 §四）：强制独立盲重建、反例预算制、Flow→KO 交叉校验门、覆盖多样性机制、外部对照源。

## 六、对已有 Knowledge Package 的处置建议

| 对象 | 建议 |
|------|------|
| KO-01 | 修正 L1 事实（四态 + AutoReview 条件 + bypass 边界）后保留（A 级） |
| KO-02 | 保留（A 级）；补 CyberModel 触发条件 |
| KO-03 | 降级改写为"环境隐式契约"（B→C 级）；递归模型进 Candidates |
| KO-04 | 保留（B 级）；补持久化/运行时分离 |
| KO-05 | 保留（A 级）；补全升级条件清单 + 位置引用 |
| KO-06 | 保留（B 级）；扩展为多层级膨胀治理 |
| KO-07 | 保留（A 级）；加"同源设计"标注 |
| **新增 KO-08** | exec_policy 执行策略系统（建议 A 级） |
| **新增 KO-09** | 模型可见上下文治理（建议 A/B 级） |
| **新增 KO-10** | 审批→策略固化闭环（建议并入 KO-08） |

## 七、交付物清单

```
validation/
├── 00_validation_summary.md      ← 本文件
├── 01_independent_findings.md    独立盲重建（审计者核验 + 盲审 SubAgent 待并入）
├── 02_ko_audit.md                Source Truth Audit（7 KO 逐条）
├── 03_coverage_gaps.md           Coverage Audit（6 独立发现 + 3 Critical Missing）
├── 04_abstraction_audit.md       Five-Layer Abstraction Audit
├── 05_flow_audit.md              Flow Atlas Audit（6 条流逐条）
├── 06_counterexamples.md         Counterexample Hunt（8 个反例/边界）
├── 07_epistemic_audit.md         Epistemic Status Audit
├── 08_reconciliation.md          Reconciliation（合并裁决）
└── 09_quality_scorecard.md       Quality Scorecard + Skill Benchmark 诊断
```

> **盲审说明**：独立盲审 SubAgent（o_0000PxPWaTb，完全隔离于已有考古结论）正在执行完全独立的盲重建。本 summary 的 01 目前基于审计者独立代码核验；盲审 agent 产出到达后将作为第二独立来源并入 01 并交叉验证。仓库与已有产物全程只读，未做任何修改。

# Failure Analysis — knowledge-archaeology v1 · Benchmark: Codex

> Status: **FAILED QUALITY GATE**
> 本文件是 knowledge-archaeology Skill v1 在真实项目（OpenAI Codex）上的第一次独立对抗验证结果。
> 它不是"Validation Failed"的简单宣告，而是对 Skill 自身失败模式的精确定位。
> 原则：保留全部痕迹，不修订已有产物。原始提取产物见 `extraction-v1/`，独立审计产物见 `independent-audit-v1/`。

---

## 一、Benchmark 基本信息

| 项 | 值 |
|----|-----|
| Skill 版本 | v1（Multi-Agent 顺序提取：Discovery → Mining → Synthesis → Validator） |
| 目标项目 | OpenAI Codex（OpenAI Codex 仓库（本地只读克隆，Rust 112 crates）） |
| 提取产物 | 7 Knowledge Object（4A+3B，L1-L5）+ 6 Flow + 原验证报告（声称 7/7 PASS、0 反例、0 冲突） |
| 审计方法 | 独立盲重建 + 六类审计（Source Truth / Coverage / Five-Layer Abstraction / Flow Atlas / Counterexample Hunt / Epistemic Status）+ Reconciliation + Scorecard |
| 审计日期 | 2026-09-03 |

## 二、基准指标（v1 基线，供 v2 对比）

| 指标 | v1 实测 | 目标（v2） |
|------|---------|-----------|
| False Acceptance Rate（误放行） | **≈43%**（7 中 3 被误放行） | → ? |
| Critical Knowledge Miss Rate | **30%**（3 个 Critical Missing / 10） | → ? |
| Over-generalization Rate | **≈21%**（KO-03 + KO-07 边缘） | → ? |
| Fact Error | **1 个**（KO-01 approval_policy 三态→实为四态） | → 0 |
| Counterexample 检测 | 原声称 0 反例，实际 8 个 | → ? |
| Source Fidelity / Coverage / Flow / Abstraction / Epistemic / Counterexample | 65 / 60 / 75 / 75 / 70 / 30 | → ? |

## 三、核心结论

### 结论 1 · Discovery 基本有效

5/7 KO 可确认（KO-02/04/06/07 完全成立），另外 2 个（KO-01/KO-03）不是"发现失败"，而是分别存在**事实错误**和**抽象层级问题**。

这意味着：
```
Repository → Code/Docs/Test/Failure analysis → Knowledge Candidates
```
这条路径**没有根本失效**。此前担心的"是不是整理得太少、漏了很多"——部分答案已给出：至少不是因为 Discovery 完全失灵。

### 结论 2 · 真正瓶颈在 Synthesis

```
Discovery     ↓ Evidence     ↓ Synthesis   ← 主要缺陷     ↓ Validation
```

典型案例：KO-03。原始证据（CODEX_SANDBOX* 环境变量用于测试跳过）是**正确**的，但 Synthesizer 把它**包装成更高阶的"自举递归逃逸阀" Pattern**。独立 Auditor 重新看证据后发现：它只能被解释为**"环境隐式契约"**。

这是典型的：
```
Evidence 正确 → Pattern 过度泛化
```
而非：
```
Evidence 错误
```

**推论**：下一版 Skill 最值得改的**不是增加更多 Discovery Agent**，而是建立真正的 `Evidence → Abstraction Promotion Gate`——每一次 `L1→L2 / L2→L3 / L3→L4 / L4→L5` 都必须经过独立的晋升判定（解释范围是否真的扩大，而非文字更抽象）。

### 结论 3 · False Acceptance ≈43% 暴露验证体系的结构性缺陷

原验证体系结构：
```
Archaeologist → Synthesis → Validator
```
若 Validator 只是读取 Archaeologist 产生的证据与结论，它会退化为"请证明这个答案没问题"，而非"重新做一遍，然后判断这个答案有没有问题"。

**同源自我确认**（Same-source Confirmation）是根因：Validator 与 Miner 共享同一推理链，没有独立盲重建作为对照，因此不具备推翻自身结论的能力。原报告"7/7 PASS、0 反例"即源于此。

## 四、三个重大遗漏及其深层含义

| 遗漏 | 性质 |
|------|------|
| exec_policy 执行策略系统 | 完整子系统（命令级审批策略 + 危险命令黑名单 + 策略固化），7 KO 完全未提及 |
| 模型可见上下文治理 | evidence 层有 SF-05 但未升维成 KO（synthesis 层遗漏） |
| 审批→策略固化闭环 | proposed_execpolicy_amendment + blocking_append_allow_prefix_rule |

它们共同揭示一个**更深层的问题**：

> 当前 Archaeology 对"显式结构 / 显式机制 / 显式架构"敏感（代码里有什么），
> 但对"系统通过什么机制**持续约束自己**"不敏感（Policy Governance / Feedback Loop / Context Control）。

即：它对"代码里有什么"敏感，对"系统如何自我治理"不够敏感。

## 五、Failure Mode 清单（设计需求来源）

| ID | Failure Mode | 证据 | 对应 v2 设计需求 |
|----|-------------|------|-----------------|
| FM-1 | Fact hallucination（Synthesis 引入不存在的事实） | KO-01 三态→四态；F-05 正确但合成 KO-01 时丢失 | F1 · Source Truth Gate |
| FM-2 | Over-abstraction（证据正确但过度泛化） | KO-03 递归模型 | F2 · Abstraction Promotion Gate |
| FM-3 | Discovery omission（高密度子系统未被覆盖） | exec_policy / 上下文治理 / 策略固化 | F3 · Independent Coverage Agent |
| FM-4 | Same-source validation（验证器无独立制衡） | 原 7/7 PASS、0 反例 vs 实际 8 反例 | F4 · Blind Reconstruction |
| FM-5 | Missing governance knowledge（对自我约束机制不敏感） | 三个遗漏均为 Policy/Governance/Context 类 | F5 · Policy/Governance Analyst |
| FM-6 | Weak contradiction discovery（反例搜索形同虚设） | 声称 0 反例 vs 实际 8 个 | F6 · Counterexample Hunter 强化 |

## 六、结论

**knowledge-archaeology 的问题不是"不会发现知识"，而是"发现之后过度综合、过度升维，且原有验证器没有形成独立制衡"。**

本次审计的价值在于：它把失败转化成了可执行的 Skill 设计需求（F1-F6），并为 v1→v2 提供了可度量的基线（False Acceptance 43% → 0？Critical Missing 3 → 0？Fact Error 1 → 0？）。

> 以后升级到 Multi-Agent v2 后，再跑一次 Codex，最有价值的不是"新报告更漂亮"，而是：
> ```
> v1 → v2
> False Acceptance 43% → ?
> Critical Missing 3 → ?
> Over-abstraction ? → ?
> Fact Error 1 → 0
> ```
> 这才是真正的 Skill Engineering。

---

*本文件与 extraction-v1/、independent-audit-v1/ 构成 v1 完整痕迹。不修订、不覆盖，作为 v2 对比基线。*

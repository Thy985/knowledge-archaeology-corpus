# Lessons for v2 — knowledge-archaeology 升级蓝图

> 来源：Codex v1 Benchmark 的独立对抗验证（见 `failure-analysis.md`）。
> 本文件把 v1 的六个 Failure Mode 转化为 v2 的显式设计需求（F1-F6），并引入第七类流（Policy Flow）。
> 目标不是"优化 Prompt"，而是用真实项目反向发现 Knowledge Archaeology Runtime 的 Failure Modes 并修复。

---

## 一、v2 架构原则

### 原则 1 · 晋升必须经过独立判定

```
L1 Candidate → L1 Validator → L2 Deriver → L2 Validator → L3 Pattern Miner → L3 Validator
→ L4 Model Builder → L4 Validator → L5 Methodology Candidate → L5 Validator
```

每次晋升（L1→L2→L3→L4→L5）都必须证明：
1. **new explanatory power**（解释范围确实扩大，不是文字更抽象）
2. **supporting evidence**（有新的独立证据支撑）
3. **scope validity**（适用范围与反例已知）

无法证明 → **降级**（L4 candidate → demote to L3），而不是保留高层级。

### 原则 2 · 验证必须独立于生产

Validator 不得读取 Synthesizer 产生的证据与结论来"确认"它。
Validator 必须：**先独立盲重建，再对比判断**。

```
❌ 请证明这个答案没问题（Same-source Confirmation）
✅ 重新做一遍，然后判断这个答案有没有问题（Blind Reconstruction）
```

### 原则 3 · 覆盖必须有多样性强制

synthesis 阶段强制检查：所有 high-density 子系统（exec_policy / context / memory / sandboxing / approval...）**必须至少被一个 KO 覆盖**，防止单一叙事线（如"安全 Gate"）垄断注意力。

---

## 二、F1-F6 设计需求

### F1 · Fact Hallucination → Source Truth Gate

- **问题**：Synthesis 阶段引入不存在的事实（KO-01 把四态写三态；"AlwaysAsk" 不存在）。
- **Gate 设计**：每条 KO 的 L1 事实必须可追溯到仓库 symbol（文件:行号）；Synthesizer 产出 KO 时，每条 L1 事实必须能回溯到 Flow Atlas 的具体 Edge；KO 与 Flow 矛盾时 KO 不得通过。
- **反制**：Counterexample Hunter 定向攻击 always/must/all/never 类表述。
- **验收**：v2 中 Fact Error = 0。

### F2 · Over-abstraction → Abstraction Promotion Gate

- **问题**：证据正确但过度泛化（KO-03 递归模型，实际只是"环境隐式契约"）。
- **Gate 设计**：每次 Ln→L(n+1) 晋升单独判定，判据为"解释范围是否真的扩大"（≥2 独立实例 → Pattern；解释多机制 → Model；跨上下文 + 例外清单 → Principle）。单实例只能标 Observed Structure。
- **强制**：Promotion 必须有"为什么这一级成立"的显式论证，否则默认不晋升。
- **验收**：v2 中 Over-generalization Rate 显著下降；无"项目局部经验被写成通用原则"。

### F3 · Discovery Omission → Independent Coverage Agent

- **问题**：高密度子系统未被覆盖（exec_policy / 上下文治理 / 策略固化）。
- **设计**：新增 Coverage Agent，**独立于 Discovery** 从仓库重新搜索，产出"高密度子系统清单"，强制每个子系统至少一个 KO 代表；若 KO 缺失，标记 Critical Knowledge Missing。
- **关键**：Coverage Agent 不能从已有 Knowledge Inventory 猜，必须独立重新搜索。
- **验收**：v2 中 Critical Knowledge Miss Rate 显著下降。

### F4 · Same-source Validation → Blind Reconstruction

- **问题**：Validator 与 Miner 同源，无独立制衡（False Acceptance 43%）。
- **设计**：Validator 执行前**先独立盲重建** Independent Findings（不看 Synthesizer 产出），再与已有 KO 对比。
- **机制**：用隔离 SubAgent（无 Miner 上下文的独立实例）执行盲重建，保证 epistemic 隔离。
- **验收**：v2 中 False Acceptance Rate 显著下降；Validator 能独立发现 Miner 的事实错误与遗漏。

### F5 · Missing Governance Knowledge → Policy / Governance Analyst

- **问题**：对"系统如何持续约束自己"不敏感（Policy / Governance / Feedback Loop / Context Control）。
- **设计**：新增 Policy/Governance Analyst 角色，专门调查：
  - 系统如何把一次性决策固化为持久策略？（审批→策略固化）
  - 系统如何治理自身的上下文/资源/执行？（上下文硬限制、并发限制、预算）
  - 系统如何自我约束（规则文档、guard、policy 文件）？
- **工具**：引入 **Policy Flow**（见下）。
- **验收**：v2 对"治理/反馈闭环"类知识的覆盖显著提升。

### F6 · Weak Contradiction Discovery → Counterexample Hunter 强化

- **问题**：声称 0 反例，实际 8 个。
- **设计**：
  - **反例预算制**：每个高价值 KO 必须完成 ≥3 个定向反例攻击（针对 always/must/all/never），找不到也要记录"已搜索的路径"。
  - 反例数为 0 必须给出搜索证据（搜了什么 bypass/override/admin/fallback/direct call/test-only path）。
  - Counterexample Hunter 不负责证明"是的"，只负责寻找"哪里不是这样"。
- **验收**：v2 中反例检测从 0 提升到与独立审计相当水平。

---

## 三、Policy Flow — 第七类流

### 为什么需要

针对 Agent / AI 系统，现有六类流（Control/State/Data/Evidence/Authority/Memory）对"治理闭环"表达不足。Codex 的**审批→策略固化闭环**（proposed_execpolicy_amendment + blocking_append_allow_prefix_rule）是典型实例：一次审批决策，固化成持久策略，影响未来所有决策。

### 定义

```
Decision → Approval → Policy → Enforcement → Future Decision
```

| 阶段 | 含义 | Codex 实例 |
|------|------|-----------|
| Decision | 产生一次判断 | 用户/Guardian 批准一条命令 |
| Approval | 判断生效为授权 | ExecApprovalRequirement → allow |
| Policy | 授权固化为持久规则 | proposed_execpolicy_amendment → blocking_append_allow_prefix_rule 落盘 |
| Enforcement | 规则约束后续行为 | ExecPolicyManager 对后续命令匹配 rules |
| Future Decision | 规则改变未来判断 | 同类命令免审 / 危险命令拒绝 |

### 与其他流的关系

- **Control Flow** 回答"下一步做什么"；**Policy Flow** 回答"系统如何改变自己对'下一步'的规则"。
- 它是 **Feedback Loop**（系统自我修正）的载体，与普通单向执行流本质不同。
- Authority Flow 描述"谁有权执行"；Policy Flow 描述"执行规则如何被固化与演进"。

### 纳入 v2

Flow Atlas 升级为七类流：Control / State / Data / Evidence / Authority / Memory / **Policy**。
flow-miner 增加 Policy Flow 构建责任，必须锚定真实 symbol（policy 文件、rule 追加、enforcement 点）。

---

## 四、v1 → v2 度量对比框架

升级后重跑 Codex，必须对比：

```
指标                    v1 基线    v2 目标
──────────────────────────────────────────
False Acceptance         43%       → ?
Critical Missing          3        → ?
Over-generalization      ~21%      → ?
Fact Error                1        → 0
Counterexample Found      0(声称)  → ?
Policy/Governance KO      0        → ≥1
Source Fidelity          65        → ?
Coverage                 60        → ?
Flow Integrity           75        → ?
Abstraction Validity     75        → ?
Epistemic Discipline     70        → ?
Counterexample Detection 30        → ?
```

> 有 v1 基线才有 v2 的"进步证明"。升级后最有价值的不是"新报告更漂亮"，而是这组指标的真实变化。

---

## 五、v2 落地清单

- [ ] Skill 增加 `Abstraction Promotion Gate`（每次晋升独立判定 + 降级机制）
- [ ] Validator 改为 Blind Reconstruction（独立 SubAgent 隔离执行）
- [ ] 新增 Independent Coverage Agent（独立重搜，强制子系统覆盖）
- [ ] 新增 Policy/Governance Analyst + Policy Flow（第七类流）
- [ ] Counterexample Hunter 反例预算制（≥3 个定向攻击/高价值 KO）
- [ ] Flow→KO 交叉校验门（L1 事实必须回溯 Flow Edge）
- [ ] benchmark 目录保留 v1 痕迹（已完成：benchmarks/）
- [ ] 用 Codex 重跑一次 v2，对比 v1→v2 指标

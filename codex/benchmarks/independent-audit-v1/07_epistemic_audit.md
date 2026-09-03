# 07 · Epistemic Status Audit（认知状态审计）

> 检查每个结论的认知状态是否与其证据强度匹配。禁止无中间推导的跃迁：
> Fact → Principle、单例 → Pattern、ADR 意图 → 实现事实、项目局部 → 通用原则。

---

## 状态标尺

```
FACT       可验证的单一事实（代码/文档中存在）
OBSERVATION  直接观察到的行为/现象（可重复）
HYPOTHESIS   解释现象的可能机制（未验证）
PATTERN      在 ≥2 独立实例中重复出现的结构
PRINCIPLE    跨上下文成立的通用规律（需例外清单）
```

## 逐项审计

### KO-01 · 原 Validated Pattern → 审计：Pattern（条件化）

- **问题**：KO-01 标 "Validated Pattern（codex 内多源验证）"，但其 L1 包含**未验证/错误的事实**（approval_policy 三态、AlwaysAsk 命名、OnRequest→Guardian 无条件）。带错误事实的 Pattern 其 epistemic status 必须降级修正。
- **跃迁检查**：Fact → Pattern 的路径有中间推导（分离机制在 3+ 机制中重复出现），不是 Fact→Principle 跃迁。✅
- **修正后状态**：**Observation/Pattern（条件化）**——claim 限定在受管默认路径；四态事实修正后恢复 Pattern。

### KO-02 · 原 Validated Pattern → 审计：Validated Pattern ✅

- 证据链完整（transcript 数值 / 结构化输出 / 超时 / 熔断 / 延迟确认全部代码确认），多源代码独立支持。
- **跃迁检查**：Fact → Pattern 有中间推导（5 个机制指向同一"证据生产链"），证据充分。✅
- **边界**：Guardian 路由条件是 OnRequest/Granular+AutoReview，claim 需注明适用范围（非全审批路径）。

### KO-03 · 原 Validated Pattern → 审计：Observation（弱）

- **核心跃迁问题**：从"CODEX_SANDBOX* 环境变量用于测试跳过"（Fact/Observation）直接跳到"自举系统的递归矛盾 + 显式逃逸阀"（Pattern/模型）——**缺少中间推导**。
  - 真实机制（环境隐式契约）只是 Observation 级：运行时设置只读信号 → 代码据此降级/跳过。
  - "递归矛盾"是 HYPOTHESIS（提出了一种解释，但 codex 代码不支持其作为核心结构）。
- **修正后状态**：**Observation（环境隐式契约）**；"自举递归逃逸阀"作为 **HYPOTHESIS** 移入 Candidates。

### KO-04 · 原 Validated Pattern → 审计：Validated Pattern ✅

- SSOT 证据充分（3 个派生函数 + 同步注释 + 环境覆盖）。无跃迁问题。✅

### KO-05 · 原 Principle → 审计：条件化 Principle

- **问题**：KO-05 标 "Principle"，且 L1 升级条件列举不完整（缺 network_approval_context / should_bypass_approval）。原则本身的跨上下文逻辑成立（fail closed 是安全默认），但：
  - 升级条件的**完整清单**需补全（否则 Principle 的适用范围表述不精确）。
  - fail closed 是"默认"而非"绝对"——Never 白名单 / ExecPolicy Allow(bypass) 是已知例外，Principle 必须带例外清单。
- **跃迁检查**：项目 5 实例 → Principle，有 Fail Closed 的工程通用性支撑（防火墙/权限系统跨领域对照），非无依据跃迁。✅ 但需条件化。
- **修正后状态**：**条件化 Principle**（默认 fail closed + 显式白名单例外 + 完整条件清单）。

### KO-06 · 原 Validated Pattern → 审计：Validated Pattern（扩展）

- 治理模式证据充分。无跃迁问题。**覆盖缺口**（模块级/变更级/API 级治理未纳入）不影响 epistemic status，但影响完整性。

### KO-07 · 原 Validated Pattern → 审计：Validated Pattern（同源标注）

- 4 实例证据充分。**epistemic nuance**：4 处五段式**同源设计**（同一团队刻意设计），其"跨项目重现性"未验证——这是 pattern 的**来源同质性**问题，不影响"项目内 Pattern"地位，但**不能**据此宣称"普遍架构指纹"（跨项目验证 pending 应保留）。
- **跃迁检查**：Pattern → 通用"架构指纹"是**过度外推边缘**——"指纹"暗示自然涌现，实际是设计签名。跨项目重现性必须标注 UNVERIFIED。

---

## 高风险跃迁清单（本审计发现的）

| 跃迁 | 位置 | 判定 | 处置 |
|------|------|------|------|
| Fact(三态) → Pattern | KO-01 L1 | **错误事实** | 修正为四态 |
| Fact(测试跳过) → Pattern(递归逃逸阀) | KO-03 | **缺少中间推导** | 降为 Observation；递归模型进 Candidates |
| Pattern(4 同源实例) → 通用"架构指纹" | KO-07 L4 | **过度外推边缘** | 标注同源设计 + 跨项目 UNVERIFIED |
| 项目 5 实例 → Principle | KO-05 | 有条件成立 | 补全条件 + 例外清单后保留 |
| ADR/注释意图 → 实现事实 | 全 KO | 无此跃迁 | 全部经代码核验 ✅ |

---

## Epistemic 审计汇总

| KO | 原状态 | 审计后状态 | 判定 |
|----|--------|-----------|------|
| KO-01 | Validated Pattern | Observation/Pattern（条件化） | **降级修正**（事实错误） |
| KO-02 | Validated Pattern | Validated Pattern | 保留 |
| KO-03 | Validated Pattern | Observation（弱）+ HYPOTHESIS(递归) | **降级** |
| KO-04 | Validated Pattern | Validated Pattern | 保留 |
| KO-05 | Principle | 条件化 Principle | 保留（补条件） |
| KO-06 | Validated Pattern | Validated Pattern | 保留（补覆盖） |
| KO-07 | Validated Pattern | Validated Pattern（同源标注） | 保留（加标注） |

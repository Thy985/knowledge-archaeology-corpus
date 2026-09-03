# 08 · Reconciliation（独立发现与已有 Knowledge 的合并裁决）

> 将 Independent Findings（IF）与已有 7 KO 合并。判定：CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / MISSING / CONTRADICTED / DUPLICATE / NEEDS_HUMAN_REVIEW。

---

## 合并裁决表

| 对象 | 裁决 | 说明 |
|------|------|------|
| KO-01 Intelligence ≠ Authority | **PARTIALLY_CONFIRMED**（需修正 L1 事实） | 核心模型成立（多 Gate 分离被代码强支持）；但 L1 事实错误（approval_policy 三态→四态、AlwaysAsk 不存在、OnRequest→Guardian 需 AutoReview）必须修正；bypass 路径需边界化 |
| KO-02 声明 ≠ 证据 | **CONFIRMED** | 全部 L1 事实经代码确认（transcript 数值/结构化/超时/熔断/延迟确认）；仅需补 CyberModel 触发条件与适用范围 |
| KO-03 自举递归逃逸阀 | **DOWNGRADED** | 真实机制为"环境隐式契约"（L2 弱模式）；"递归矛盾"模型过度演绎，无核心沙箱实现支持；移入 Candidates 待跨项目验证 |
| KO-04 单一真相源 | **CONFIRMED** | 三策略派生 + 三字段同步 + 环境覆盖全部代码确认；可补"持久化与运行时状态分离"证据 |
| KO-05 Fail Closed | **PARTIALLY_CONFIRMED** | 多实例成立；但升级条件列举不完整（缺 network_approval_context/should_bypass_approval/owner_network_policy/wants_no_sandbox_approval）；位置引用（execution.rs:44-60）与实测（agent/control.rs）不符；fail closed 为默认非绝对 |
| KO-06 核心模块反膨胀 | **CONFIRMED**（有覆盖缺口） | crate 级治理成立；遗漏模块级（500/800 LoC）/变更级（800/500 行）/API surface 三同族机制，建议扩展为"多层级膨胀治理体系" |
| KO-07 五段式架构指纹 | **CONFIRMED**（需加同源标注） | 4 处实例代码确认；但同源设计（同一团队刻意设计）——"架构指纹"应标注为设计签名，跨项目重现性 UNVERIFIED |
| IF-04 exec_policy 执行策略系统 | **MISSING**（Critical） | 7 KO 完全未覆盖；建议新增 KO-08 |
| IF-05 模型可见上下文治理 | **MISSING**（synthesis 层） | evidence 层有 SF-05，但未升维成 KO；建议新增 KO-09 |
| IF-10 审批→策略固化闭环 | **MISSING**（Critical） | proposed_execpolicy_amendment + blocking_append_allow_prefix_rule；建议并入 KO-08 |
| IF-06 持久化/运行时分离 | MISSING（中等） | 可并入 KO-04 |
| IF-07 Weak 引用 | MISSING（中等） | 单列 L2/L3 候选 |
| IF-08 rollout_budget | MISSING（中等） | 单列 L2/L3 候选 |
| IF-09 多层级膨胀治理 | PARTIAL | KO-06 扩展 |

---

## NEEDS_HUMAN_REVIEW 项

1. **KO-03 是否应完全删除或降级改写**：递归模型无实现支持是明确的，但"环境隐式契约"是否值得保留为独立 KO 需人工判断其价值（它确实解释了 AGENTS.md 的强制约束背后的工程动机）。
2. **exec_policy 系统（IF-04）是否升为最高价值 KO**：证据充分（完整子系统 + 策略固化闭环），但它是"权限系统的另一支柱"这一判断需要人工确认优先级——审计者建议 A 级。
3. **五段式架构指纹（KO-07）的跨项目验证计划**：是否纳入个人知识库的跨项目验证清单（FormulaFix/TeamMind/GrowthOS）。

---

## 裁决后状态汇总

| 对象 | 原状态 | 裁决后状态 |
|------|--------|-----------|
| KO-01 | Validated Pattern (L4, A) | Pattern（条件化）(L4, A) —— 修正事实错误 |
| KO-02 | Validated Pattern (L4, A) | Validated Pattern (L4, A) —— CONFIRMED |
| KO-03 | Validated Pattern (L3, B) | Observation/弱Pattern (L2) —— DOWNGRADED |
| KO-04 | Validated Pattern (L3, B) | Validated Pattern (L3, B) —— CONFIRMED |
| KO-05 | Principle (L4, A) | 条件化 Principle (L4, A) —— 补条件 |
| KO-06 | Validated Pattern (L3, B) | Validated Pattern (L3, B) —— 扩展覆盖 |
| KO-07 | Validated Pattern (L4, A) | Validated Pattern (L4, A) —— 加同源标注 |
| 新增 | — | KO-08 exec_policy 执行策略系统（建议 A） |
| 新增 | — | KO-09 模型可见上下文治理（建议 A/B） |

**净效果**：7 KO → 2 个降级修正（KO-01 事实错误、KO-03 抽象降级）、2 个需补条件（KO-05）、3 个确认+扩展（KO-02/04/06/07）、2 个新增（KO-08/09）。**确认了 5 个高价值 KO 的核心方向，修正了 1 个事实错误，降级了 1 个过度抽象，补上了 2 个重大遗漏。**

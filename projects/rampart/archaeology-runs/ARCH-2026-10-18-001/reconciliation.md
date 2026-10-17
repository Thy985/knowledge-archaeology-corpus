# Reconciliation — ARCH-2026-10-18-001（RAMPART refresh）

> 阶段⑤独立 Auditor 盲重建修正的合并记录。**只修正 Corpus Artifact，不修改生产 Skill**（无系统性 skill 缺陷证据，未触发 Skill Evolution）。

## 判定统计（独立验证）
CONFIRMED 13 / PARTIALLY_CONFIRMED 1 / DOWNGRADED 0 / OVER_GENERALIZED 0 / MISSING 3 / CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 0

## 应用的修正（3 项）
| # | 修正 | 来源 | 落点 |
|---|---|---|---|
| 1 | 06 层测试清单补充 `tests/unit/pytest_plugin/test_trial.py` + `test_trial_integration.py`（trial 解析/遮蔽/CLI override 专项覆盖） | Auditor F10（MISSING） | corpus 06-validation-evidence.md |
| 2 | EK-R10 scope 收紧：strict JSON-mode 指字段值与 JSON domain 校验，**信封层 `additionalProperties: True`（开放）** | Auditor F8（PARTIALLY_CONFIRMED） | corpus 02-engineering-knowledge.md |
| 3 | 新增 EK-R16（eval_purpose requires eval_result 不变量）+ EK-R17（population allow_empty 双路径：聚合可空 / 执行强制 ≥1） | Auditor F7/F9（MISSING） | corpus 02-engineering-knowledge.md |

## Candidates 追加
- C-R07 · method marker 遮蔽（shadow）语义是否为 pytest get_closest_marker 通用行为（Cross-project Hypothesis，Auditor F6）

## 保留项（不弱化证据标准）
- 全部 CONTRADICTED = 0：主考古无事实级冲突
- Counterexample 预算 5 KO × ≥1 攻击全保留（06-validation-evidence.md §5）
- C-R01~R07 全部保持 Hypothesis/Observation 标注，不升层

## 未触发 Skill Evolution 的理由
Auditor 捕获的 3 处 MISSING 属**单项目覆盖广度问题**（测试清单穷举、边界参数搜索），非 skill 系统性缺陷；"Auditor 应独立列测试清单 + 搜索 additionalProperties/allow_* 边界参数"可作为工程惯例沉淀，但不足以构成 P0/P1 Proposal。观察后续 2-3 个项目是否复现同类遗漏后再评估。

# Independent Validation — 盲重建审计报告（AProver）

> Auditor 独立重读仓库建立 Independent Findings，不与考古包对比前先盲重建。
> 判定: CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / MISSING / CONTRADICTED / NEEDS_HUMAN_REVIEW

## 判定统计
| 判定 | 数量 |
|---|---|
| CONFIRMED | 15 |
| PARTIALLY_CONFIRMED | 4 |
| DOWNGRADED | 1 |
| OVER_GENERALIZED | 1 |
| MISSING | 1 |
| CONTRADICTED | 0 |
| NEEDS_HUMAN_REVIEW | 0 |
| 合计 | 22 |

## 3 个成功判定（独立重建证据）

### #1 CONFIRMED — 删除不对称性（KO-01/EK-01）
盲重建: 独立读 soundness_policy.py docstring, 独立推导出 '排除=集合收窄, 错误排除隐藏 bug' 结论 → 与考古包一致
证据: soundness_policy.py L1-31

### #2 CONFIRMED — 五档分级与 realism 免疫（KO-05/EK-04）
盲重建: 从 bug_reporter.py create_report 独立追踪 tier 赋值链, 确认 confirmed_dynamic 免疫 realism 降级（main.assertion.N 例外）
证据: bug_reporter.py L156-171; PIPELINE.md Tier assignment

### #3 CONFIRMED — LLM 判定只降级不删除（KO-01/KO-06）
盲重建: 独立检查所有可能 drop finding 的决策点（realism 降档/UNRESOLVED 保留/refiner 双条件）, 确认 AGENTIC_JUDGMENT 无删除权
证据: soundness_policy.py resolve_action; realism_checker.py 降档路径; cex_validator.py UNRESOLVED

## 3 个错误/降级判定（盲重建攻击点）

### #16 DOWNGRADED — EK-16 漂移的性质
攻击: 初稿把 EK-16 标为 '实现缺陷'; 盲重建发现实现符合 README(agentic realism 默认 OFF), 是测试滞后 → 修正为三方漂移(Observation)
证据: README §Agentic mode vs tests/test_agentic_components.py:29 vs config 默认 False

### #21 OVER_GENERALIZED — KO-06 曾标 Principle
攻击: 单项目(甚至三项目)证据不足以声明 Principle; 领域实例不足 → 收紧为 L4 Cognitive Model(3 项目收敛, Cross-project validation pending)
证据: 03 层 epistemic_status 修正记录

### #12 MISSING — 反例具体化细节
攻击: 初稿未覆盖 concretization 失败→UNRESOLVED 的具体路径与 reproducer agent 尝试链; 盲重建发现 _reproducer_agent_attempt/_generate_system_entry_reproducer/_try_dynamic_validation 存在 → 补入 04 Flow Atlas 状态流
证据: cex_validator.py L2037/L2081/L2212

## 遗漏（盲重建发现的初稿缺口）
- MISSING-1: spec_generator_v2.py / spec_refiner.py 的具体精化策略（初稿只覆盖 v1 路径）→ 记为待补（不影响核心 KO）
- MISSING-2: per-role provider 混合(anthropic+claude-code+codex) 的跨提供者一致性风险 → 补入 KO-02 反例
- MISSING-3: evaluation/ 基线的具体指标口径 → 未在核心知识面, 如实披露

## Benchmark case（供 skill CI 使用）
- 名称: AProver-soundness-policy
- 输入: soundness_policy.py + PIPELINE.md + bug_reporter.py
- 期望产物: 识别出 (1) 删除不对称性 (2) 三判定类型→权限映射 (3) 五档分级+realism 免疫 (4) LLM 判定无删除权
- Gold: KO-01/KO-02/KO-05/KO-06 + EK-01..04

## 判定明细（22 条全表）
| # | 判定 | 对象 | 摘要 |
|---|---|---|---|
| 1 | CONFIRMED | EK-01/KO-01 | 删除不对称性 |
| 2 | CONFIRMED | EK-02 | 三判定类型→权限 |
| 3 | CONFIRMED | EK-03 | refiner 双条件授权 |
| 4 | CONFIRMED | EK-04/KO-05 | 五档分级+免疫 |
| 5 | CONFIRMED | EK-05 | UNRESOLVED 不丢弃 |
| 6 | CONFIRMED | EK-06 | 动态验证确定性事实 |
| 7 | CONFIRMED | EK-07 | 隔离验证+stub |
| 8 | CONFIRMED | EK-08 | CEGAR+Soundness Guard |
| 9 | CONFIRMED | EK-09 | SCC 分层生成序 |
| 10 | CONFIRMED | EK-10 | dual-spec+分歧 |
| 11 | CONFIRMED | EK-11 | DSL 直接翻译 |
| 12 | MISSING | concretize 链 | reproducer 尝试链 |
| 13 | CONFIRMED | EK-13 | realism witness 短路 |
| 14 | CONFIRMED | EK-14 | 确定性优先模式 |
| 15 | PARTIALLY | EK-15 | LLM 审计+降档保留（pass-through 细节存疑） |
| 16 | DOWNGRADED | EK-16 | 漂移性质修正 |
| 17 | CONFIRMED | EK-17 | 属性类免疫 |
| 18 | CONFIRMED | EK-18 | CEx 去重 |
| 19 | PARTIALLY | EK-19 | spec_evidence（v2 路径使用率待核） |
| 20 | CONFIRMED | EK-20 | ML 内核验证里程碑 |
| 21 | OVER_GENERALIZED | KO-06 | Principle→L4 收紧 |
| 22 | PARTIALLY | KO-02 | 信任链（solver 元验证缺失=反例） |

## 结论
核心认知（删除不对称性/智能≠权威/证据分级/确定性短路）全部 CONFIRMED；无 CONTRADICTED；2 处降级/收紧已 reconcile（禁止修改原考古产物, 修正并入 06 报告与 03 层）。

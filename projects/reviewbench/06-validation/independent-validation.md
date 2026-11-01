# Independent Validation Report — ReviewBench（ARCH-2026-10-17-001）

> Auditor 盲重建：不把考古产物当事实来源，独立重读仓库（matcher.ts/scorer.ts/eval.ts/input-validation.ts/classify.ts/classify-adapter.ts/HARNESS.md/METHODOLOGY.md/JUDGING_INPUT.md/OFFLINE_EVAL.md/try-agent.sh/package.json）建立 Independent Findings，再与考古产物（02/03/04/05/06）逐条对照。

## A. Independent Findings（盲重建，共 24 条）
| IF | 内容 | 证据 |
|---|---|---|
| IF-01 | 双指标公式：grounded 只用 golden 标签；augmented 纳入 unmatched classifier TP | scorer.ts `grounded_precision/recall`、`augmented_precision/recall`；METHODOLOGY §7.1/7.2 |
| IF-02 | 匹配判据 "fixing one would fix (or substantially address) the other"；同位置不同问题不匹配 | matcher.ts:43 |
| IF-03 | 文件内 chunking，CHUNK_SIZE=10 | matcher.ts `const CHUNK_SIZE = 10` |
| IF-04 | precision 认领规则：candidate 按序认领第一个未认领 golden | METHODOLOGY §6.2.4；scorer.ts matched Map |
| IF-05 | recall 覆盖去重：golden 计一次 | scorer.ts `covered_golden` Set；§6.2.4 |
| IF-06 | classifier 双角色（corpus labeling + unmatched classification）+ pi session per PR+SHA + 只读工具（Read/grep/find/ls） | classify.ts 头部注释；METHODOLOGY §5.3/6.3 |
| IF-07 | eval.ts 有 manifest/golden identity guard（repo/pr_number/head/base 全等才继续） | eval.ts:437-447 |
| IF-08 | judge timeout 随 unmatched findings 数增长：`JUDGE_CALL_TIMEOUT_MS + PER_FINDING*count` | eval.ts:449；judge-call.ts:9-12 |
| IF-09 | input-validation：malformed-json/invalid-shape 跳过；strict 模式默认 true 硬失败，--no-strict 警告继续 | input-validation.ts:195-222；eval.ts:23-25,123 |
| IF-10 | package.json 依赖 `@earendil-works/pi-coding-agent ^0.74.0`（README 写 badlogic/pi-mono） | package.json；README pi 文档链接 |
| IF-11 | classify-adapter.ts 注释 "Fetches PR context from GitHub" vs eval.ts 注释 "never the upstream API"（表面矛盾） | classify-adapter.ts:29-32；eval.ts:429 |
| IF-12 | HARNESS 双 adapter：GitHub adapter（App/fork）与 Local adapter（diff/repo/API），统一 normalized output | HARNESS.md §1:17-22 |
| IF-13 | duration 回退：`review_ms` 缺失时用 `duration_seconds*1000`（legacy 含 retry） | METHODOLOGY §7.5:651 |
| IF-14 | 镜像 fallback：mirror 缺 commit → upstream（MIRROR_ORG 可换 org） | try-agent.sh:45,69-85 |
| IF-15 | 全量数据：4632 findings / 2623 TP（56.6%）；test set 565/352（62.3%）；producer 差异（claude-sonnet-4.6 81.8% vs gpt-4o 4.6% vs gemini-2.5-pro 21.1%） | METHODOLOGY §5.5 表 |
| IF-16 | classifier 校准：hill-climb human dev set；TP/FP 96.6%；severity 98.7% within-one/62.9% exact；category 79.7% exact；47 findings 人工修正 | METHODOLOGY §5.3/5.4 |
| IF-17 | offline 结果不可提交 leaderboard；仅 portal 官方 run | OFFLINE_EVAL.md；JUDGING.md |
| IF-18 | fresh container per PR、无跨 PR 状态、镜像 digest pin、RB_* env、agent 不知是 benchmark | AGENT_CONTRACT.md |
| IF-19 | 3 runs 取均值 + run 级保留可审计 | METHODOLOGY §7.7 |
| IF-20 | §8 六类 Threats：golden incompleteness / classifier dependence / offline-online / matcher dependence / PR snapshot / producer bias | METHODOLOGY §8 |
| IF-21 | §9 八项 reproducibility artifacts | METHODOLOGY §9 |
| IF-22 | manifest 治理面：secrets{env,files}/egress[]/pr_timeout_seconds；schema 校验 | schema/agent-manifest.schema.json；example-reviewer.json |
| IF-23 | finding 字段含 3 justification + producer + source{type,alignment} | golden/axios_axios_10836 样本 |
| IF-24 | TP/FP 定义三维：true（factually correct）+ relevant + within scope；FP failure modes 枚举 | METHODOLOGY §5.2 |

## B. 对照判定（Auditor 判定 vs 考古产物）
| 考古条目 | Auditor 判定 | 说明 |
|---|---|---|
| EK-01/02/04/05/25/26/27 | **CONFIRMED** | 公式与规则逐字对应（IF-01/02/03/04/05） |
| EK-03/06/07/10 | **CONFIRMED** | classifier 双角色/校准/文化披露/dedup（IF-06/16） |
| EK-08/09/11/12/13/29 | **CONFIRMED** | producer mix/PR 快照/justification（IF-15/23/24） |
| EK-14/15/16/17/18/19/22/23/24 | **CONFIRMED** | 隔离/治理/镜像/复现（IF-17/18/19/21/22） |
| EK-20/21 | **CONFIRMED** | 宏微/分层（§7.6/7.3） |
| EK-30（judge-call） | **PARTIALLY_CONFIRMED** | 考古只写超时常量；遗漏"timeout 随 finding 数增长"（IF-08）——补 EK-30 细节 |
| 01 层外部依赖"pi（badlogic/pi-mono SDK）" | **PARTIALLY_CONFIRMED / CORRECTED** | package.json 实际为 `@earendil-works/pi-coding-agent`；README 的 badlogic/pi-mono 链接可能是文档过时或包改名——01 层修正 |
| 04 Flow C2 | **MISSING** | 遗漏 eval.ts manifest identity guard（IF-07）与 strict 默认 true（IF-09）——补 04 |
| 01 层核心模块 | **MISSING** | 遗漏 HARNESS 双 adapter（GitHub/Local）（IF-12）——补 01 |
| 02 EK-19 | **CONFIRMED** | 镜像 fallback 已写（IF-14） |
| 03 KO-01~06 | **CONFIRMED（无 OVER_GENERALIZED）** | 全部标 Cross-project validation pending；无 L4/Principle 跳级 |
| 05 C-01~07 | **CONFIRMED（Epistemic 诚实）** | Hypothesis/Observation 未冒充知识 |
| 06 Validation | **CONFIRMED（基线匹配）** | 独立审计未发现新的事实错误 |

**判定统计**：CONFIRMED 20 / PARTIALLY_CONFIRMED 2 / DOWNGRADED 0 / OVER_GENERALIZED 0 / MISSING 3 / CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 1。

## C. 3 成功（Auditor 独立确证的核心）
1. **双指标分离**（EK-01）：盲重建 scorer.ts 独立推导出 grounded/augmented 四公式与考古产物一致——grounded 是"人类验证标签"通道、augmented 是"classifier 补充"通道，golden 非穷尽被显式承认（§5.4/§8）。
2. **语义匹配判据**（EK-25）：盲重建 matcher.ts 独立发现 "fixing one would fix (or substantially address) the other"——考古产物 EK-25 与其逐字一致。
3. **评测隔离**（EK-14/15/17）：盲重建 AGENT_CONTRACT + OFFLINE_EVAL 独立确认 fresh container/digest pin/offline 不可提交——考古 KO-03 的"权威分离不变量"与之相符。

## D. 3 错误（Auditor 发现考古产物偏差）
1. **pi SDK 命名偏差**（PARTIALLY_CONFIRMED/CORRECTED）：考古 01 写"pi（badlogic/pi-mono SDK）"——package.json 实际依赖 `@earendil-works/pi-coding-agent ^0.74.0`；README 的 provider 文档链接指向 badlogic/pi-mono。判定：实现以 package.json 为准，文档链接疑似过时；01 层已修正，留 NEEDS_HUMAN_REVIEW（Owner 可确认 pi 包归属）。
2. **identity guard 遗漏**（MISSING）：eval.ts 在 classification 前有 manifest/golden 身份全等检查（防 PR 身份错配），考古 04 Flow C2 未写——已补。
3. **双 adapter 遗漏**（MISSING）：HARNESS 的 GitHub adapter（App/fork）与 Local adapter 是 agent 接入的两条 parallel path（alternate path），考古 01 未写——已补。

## E. 遗漏（Auditor 独立发现但考古未覆盖）
- classifier 的实现形态：pi session per PR+SHA（共享上下文）+ 只读工具（Read/grep/find/ls）+ Full pi resources（extensions/skills/context files）——考古只写"prompt 公开"，未写"classifier 是有工具能力的 agent session"。已补入 02（EK-03 细节）。
- strict/--no-strict/--allow-empty 输入开关（admin/exception 路径）：已补 04。
- judge timeout 增长机制（exception 路径）：已补 02 EK-30。

## F. Benchmark case（用本项目反观 skill 评测层）
ReviewBench 是对 knowledge-archaeology skill"自建基准方法论"的最佳对照物：
- **judge 校准约束落地**：ReviewBench 的 classifier hill-climb + agreement 报告 + 47 人工修正，正是引擎 evaluation 层"程序化断言优先、LLM-judge 次级信号、LLM-judge 必须校准"的完整工程化实例。
- **golden set 非穷尽 → augmented 指标**：与三环方法论"独立验证"互补——golden 是验证环，augmented 是发现环。
- **防自我验证**：fresh container + offline 隔离 + 3 runs——与"评测者与参赛者分离"一致。
- **可迁移结论（跨项目 Candidate）**：C-01（双指标可迁移）、C-04（classifier 文化依赖）对 skill 自建基准方法论有直接指导。

## G. 结论
- 考古产物质量：无 CONTRADICTED、无 OVER_GENERALIZED、Epistemic 标注诚实；20/24 IF 被考古覆盖。
- 修正：3 处 MISSING（identity guard、双 adapter、strict 开关）+ 1 处 CORRECTED（pi SDK 命名）+ 2 处 PARTIALLY_CONFIRMED（timeout 增长、classifier session 形态）——全部并入 Reconciliation。
- 不需要 Skill Evolution（无系统性 skill 缺陷暴露；考古遗漏均属本项目覆盖广度问题，非 skill 机制缺陷）。

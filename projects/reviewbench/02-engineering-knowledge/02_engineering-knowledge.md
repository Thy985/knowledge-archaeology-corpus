# 02 · Engineering Knowledge（EK Graph）— ReviewBench（ARCH-2026-10-17-001）

> 六类边：mechanism(same-mech) / subsystem(same-subsys) / causal(A→B) / dependency(A→B) / constraint(A→B) / contrast(A↔B)。证据均为仓库实际内容。

## A. 评测可信度（grounded/augmented 双指标族）
**EK-01 双指标分离：grounded 只用 golden 标签，augmented 纳入 classifier 对 unmatched 的判定（Fact）**
证据：docs/METHODOLOGY.md §7.1-7.2（"The first pair uses only golden-set labels; the second pair augments with classifier labels on unmatched findings"）；scorer.ts §7.1 grounded_precision/recall、§7.2 augmented_precision/recall（`(matched_tp+unmatched_tp)/candidate_count`、`(golden_tp_covered+unmatched_tp)/(golden_tp+unmatched_tp)`）。
links: causal(EK-01→EK-04); mechanism(EK-01,EK-02); subsystem(A)

**EK-02 golden 非穷尽被显式承认并设计补偿（Fact）**
证据：§5.4 "The golden set is not assumed to be exhaustive: there may be valid findings on a PR that no producer surfaced. This is a known limitation and motivates the augmented metrics defined in Section 7."；§8 "Golden set incompleteness"。
links: causal(EK-02→EK-01); contrast(EK-02↔EK-08)

**EK-03 同一 classifier 双角色：corpus labeling + unmatched classification（Fact）**
证据：§5.3 "The same classifier is used for corpus labeling and for unmatched candidate findings during evaluation"；§6.3 "classified using the same Claude Sonnet 5 classifier described in Section 5.3"；classify.ts（pi session per PR+SHA 共享上下文，Read/grep/find/ls 只读工具，Full pi resources）。
links: mechanism(EK-03,EK-14); causal(EK-03→EK-06); constraint(EK-03→EK-07)

**EK-04 precision 认领规则：candidate 按序认领第一个未认领的 golden match，防多候选抢 credit（Fact）**
证据：§6.2.4 "Each candidate claims at most one golden finding: the first unclaimed match in golden-set order… Only that golden finding is claimed; the candidate's other matches remain available for later candidates to claim."；scorer.ts matched_candidates Map。
links: causal(EK-04←EK-01); constraint(EK-04→EK-05); subsystem(A)

**EK-05 recall 覆盖去重：多 candidate 匹配同一 golden 只计一次，不膨胀 recall（Fact）**
证据：§6.2.4 "All golden findings matched by one candidate can contribute to recall. Multiple candidates matching the same golden finding do not inflate recall — the golden finding is counted once."
links: constraint(EK-05→EK-04); mechanism(EK-05,EK-11); subsystem(A)

**EK-06 classifier 校准纪律：hill-climb against human dev set + 校准后重算下游指标（Fact）**
证据：§5.3 "hill-climbed against a development set of human-authored findings… with prompt changes evaluated against labeled examples not used to motivate that iteration… When the classifier is recalibrated, affected labels and downstream metrics are recomputed."；TP/FP 96.6% agreement；47 findings 人工修正（§5.4）。
links: causal(EK-06→EK-10); dependency(EK-06 依赖 EK-13); subsystem(B)

**EK-07 classifier 文化依赖被披露：prompt 编码的是作者团队 taste（Observation）**
证据：§8 "The classifier prompt encodes *our* working group's taste; another team running this methodology could configure the classifier differently and reach different scores for the same agents."
links: constraint(EK-07→EK-06); contrast(EK-07↔EK-12); subsystem(B)

## B. golden 构建（多源 producer + 反偏差）
**EK-08 多源 producer：human review + ccr + LLM review agents + deterministic tools（Semgrep）（Fact）**
证据：EXTRACTION.md §1（"Human comments and inferred author actions form the backbone of the golden set; LLM passes and deterministic tools supplement them"）；§5.5 producer 表（llm_review/ccr/review_comment/tool 四类 source type）。
links: mechanism(EK-08,EK-11); causal(EK-08→EK-09); subsystem(B)

**EK-09 producer 间 TP rate 差异巨大且被公开（Fact）**
证据：§5.5 全量表——claude-sonnet-4.6 81.8% TP rate vs gpt-4o 4.6% vs gemini-2.5-pro 21.1% vs human_review 20.4% vs semgrep 5.5%；Total 56.6%。
links: causal(EK-09→EK-12); subsystem(B)

**EK-10 dedup 用同一 matcher：findings 跨 producer 去重防 recall 分母膨胀（Fact）**
证据：§5.4 "findings from different producers that refer to the same underlying issue are deduplicated using the same matching procedure described in Section 6.2… This prevents the recall denominator from being inflated by redundant findings across producers."
links: mechanism(EK-10,EK-16); causal(EK-10→EK-08); constraint(EK-10→EK-05)

**EK-11 producer bias 显式披露：candidate 与 producer 同模型可能获益（Observation）**
证据：§5.5 "A candidate reviewer whose earlier output contributed golden findings may be graded partly against its own prior output… The composition is disclosed so that results can be read with this in mind."；§8 "Producer bias"。
links: contrast(EK-11↔EK-08); constraint(EK-11→EK-01); subsystem(B)

**EK-12 PR 快照语义：base/head 锁定在 review-time，head 后 commit 对 agent 不可见（Fact）**
证据：§3.2 "The head commit is the state of the code at the time review feedback would have been given — before any follow-up commits… it is the diff that candidate agents will review"；HARNESS.md §2 "Agents must not see commits after head."；§8 "PR-level snapshot"。
links: constraint(EK-12→EK-01); dependency(EK-12 依赖 EK-19); subsystem(C)

**EK-13 分布代表性：test set 25/219 按语言/规模/类别/severity 抽样（Fact）**
证据：README "The 25 test set tasks were selected as a representative sample of the full set… preserving its mix of major languages, change sizes, and repository diversity while also covering every finding category and severity level."；§3.1 仓库选择四标准。
links: constraint(EK-13→EK-08); subsystem(B)

## C. 评测隔离与权威（AGENT_CONTRACT）
**EK-14 fresh container per PR + 无跨 PR 状态（Fact）**
证据：AGENT_CONTRACT.md "We start a fresh container from your image for every pull request… Nothing carries over between pull requests, and your agent never learns it is part of a benchmark."
links: mechanism(EK-14,EK-15); constraint(EK-14→EK-01); subsystem(C)

**EK-15 镜像 digest pin + RB_* env 契约（Fact）**
证据：AGENT_CONTRACT.md 调用示例（`your-image@sha256:...`、`-e RB_NWO/RB_PR_NUMBER/RB_BASE/RB_HEAD/RB_AGENT/RB_DIFF/RB_PR_JSON/RB_OUT`）；schema required `image`。
links: dependency(EK-15→EK-14); subsystem(C)

**EK-16 secrets/egress 治理：manifest 声明最小访问（Fact）**
证据：schema `secrets{env,files}` + `egress[]` + `pr_timeout_seconds`；agents/example-reviewer.json（`"secrets":{"env":["EXAMPLE_API_KEY"],"files":["/root/.config/example/creds.json"]}`, `"egress":["api.example.com"]`）。
links: constraint(EK-16→EK-14); subsystem(C)

**EK-17 offline 结果不可提交 leaderboard：官方评审隔离（Fact）**
证据：OFFLINE_EVAL.md "Offline results are for development and tuning. Only a final run started from the self-service portal can appear on the leaderboard."；JUDGING.md "Results produced locally cannot be submitted or published directly to the ReviewBench leaderboard."
links: constraint(EK-17→EK-01); contrast(EK-17↔EK-01); subsystem(D)

**EK-18 三次重复运行取均值 + run 级保留可审计（Fact）**
证据：§7.7 "Each candidate evaluation is run three times… The final score… is the arithmetic mean of the corresponding scores from the three runs. The three component run scores are retained so the published average can be audited and run-to-run variance can be inspected."
links: constraint(EK-18→EK-17); subsystem(D)

**EK-19 仓库镜像机制：review-bench/<owner>_<repo> 防上游删除/重写（Fact）**
证据：README "Each source repository in the corpus has a mirror… holding every task's base and head commits… the benchmark still runs if an upstream repository is deleted or rewritten."；RUNNER.md "If the mirror lacks a commit, it falls back to the upstream repository."
links: dependency(EK-19→EK-12); mechanism(EK-19,EK-22); subsystem(C)

## D. 计分与报告（可复现）
**EK-20 宏/微平均双报（Fact）**
证据：§7.6 "macro-averaging (mean over PRs)… and micro-averaging (pool counts across PRs before computing the ratio)… they answer different questions: macro captures behavior on the typical PR; micro captures behavior on the typical finding."
links: contrast(EK-20↔EK-21); subsystem(D)

**EK-21 severity/category 分层报告默认开启（Fact）**
证据：§7.3 "We report severity- and category-stratified metrics by default… because aggregate scores can hide important behavior."（security-focused vs high-impact agent 两种用例）。
links: contrast(EK-21↔EK-20); subsystem(D)

**EK-22 review duration 单独报告不入分数（Fact）**
证据：§7.5 "Duration is reported alongside the quality metrics rather than incorporated into the precision, recall, or F_beta calculations."；`review_ms` 记录规则（排除 retry/prep/matching）。
links: mechanism(EK-22,EK-19); subsystem(D)

**EK-23 八项 reproducibility artifacts + checkpoint fingerprint（Fact）**
证据：§9 八项清单（PR list/producers/classifier version/matcher version/candidate version/severity provenance/aggregation choice/three run results）；scripts/eval/checkpoint-fingerprint.ts。
links: constraint(EK-23→EK-18); dependency(EK-23 依赖 EK-01); subsystem(D)

**EK-24 input-validation：candidate findings schema 校验（Fact）**
证据：scripts/eval/input-validation.ts；tests/eval-input-validation.test.ts。
links: constraint(EK-24→EK-14); subsystem(D)

## E. 匹配与 LLM-judge 机制
**EK-25 匹配判据 = 语义问题等价而非位置等价（Fact）**
证据：matcher.ts "Two findings match if fixing one would fix (or substantially address) the other. They do NOT match if they happen to be at the same location but describe different issues."
links: mechanism(EK-25,EK-16); causal(EK-25→EK-04); subsystem(E)

**EK-26 many-to-many 匹配 + raw correspondences 保留审计（Fact）**
证据：§6.2.3 "The raw match correspondences are preserved for audit"；matcher.ts 输出 `matchedGoldenIndices` 数组。
links: causal(EK-26→EK-04); subsystem(E)

**EK-27 chunking 纯操作优化不影响匹配语义（Fact）**
证据：§6.2.1 "Chunking is purely an operational concern; it does not affect the matching semantics."
links: contrast(EK-27↔EK-25); subsystem(E)

**EK-28 离线/在线收敛是经验问题，在线信号应回喂（Observation）**
证据：§8 "Offline metrics use the classifier as a surrogate for that human reaction… Where online signal is available (comment reactions, resolution rates…) it should be compared against offline TP/FP labels… any systematic divergence should be fed back into the labeling guidelines."
links: causal(EK-28→EK-06); contrast(EK-28↔EK-01); subsystem(B)

**EK-29 每条 finding 自带 justification 证据链（Fact）**
证据：golden/axios_axios_10836 样本（tp_fp_justification/severity_justification/category_justification 字段）；METHODOLOGY.md §2 finding 定义。
links: constraint(EK-29→EK-01); subsystem(B)

**EK-30 judge-call 超时/并发与 usage 追踪（Fact）**
证据：scripts/eval/judge-call.ts（`JUDGE_CALL_TIMEOUT_MS`=10min 默认、`JUDGE_CALL_PER_FINDING_MS`=2min）；eval.ts 按 unmatched findings 数增长超时（`JUDGE_CALL_TIMEOUT_MS + PER_FINDING_MS*count`）；scripts/lib/usage.ts（`aggregateCandidateUsage`）。
links: constraint(EK-30→EK-14); subsystem(E)

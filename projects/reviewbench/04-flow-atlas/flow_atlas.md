# 04 · Flow Atlas · ReviewBench（七类流）

> 全部 Edge 从仓库实际代码导出，可回溯到 symbol/file/condition/state transition。commit `c17b05b`。

## 1. Control Flow（控制流）
**评测主管道**（scripts/eval/eval.ts）：
```
parseArgs(CLI) → loadCandidateFindings(--candidate)        [input-validation.ts L77-163]
      ↓ 异常分支: strictInputProblems 非空 → 列问题（strict 默认开）[L193-235]
GoldenStore.init(--golden)                                 [L194-236]
      ↓ 分支: RB_DELETE_GOLDEN_AFTER_LOAD=1 → deleteFromDisk()（store 继续服务）[L225-235]
buildEvalInputSummary → overlap keys（candidate ∩ golden）  [L165-191]
      ↓ 分支: --limit 截断 overlap
per-PR evalPR（并发 --concurrency）:                        [L364+]
  dedup(matcher) → resolveForScoring → classify unmatched → scorePR
      ↓ 每完成 PR → saveCheckpoint（fingerprint+completed+details sidecar）[L288-302]
aggregate(macro+micro) → printSummary → (--ingest: writeGoldenSet + removeCheckpoint) [L238-242, L304-309]
```
- 条件 Edge: `strictInputProblems.length>0 && strict` → fail（input-validation.ts L193-235）
- 条件 Edge: `loadCheckpoint: data.fingerprint !== expectedFingerprint` → 丢弃，重评（eval.ts L272-278）
- 条件 Edge: `goldenStore.get(prKey)==null` → evalPR 返回 null（跳过，L371-372）
- 条件 Edge: `candidateIdentityProblem != null` → throw（L373-376）
- 条件 Edge: `unmatchedFindings.length>0` → 才调 classifier（L405-406）；否则 scorePR 直接算（unmatched=∅）

**容器 run 控制流**（scripts/try-agent.sh + AGENT_CONTRACT.md L83-92）：
```
per PR: init repo → mirror fetch(base,head) → fallback upstream → detach checkout head → diff base...head → docker run
    → 分支: exit!=0 / 无 valid JSON / pr.head≠RB_HEAD / pr.pr_number≠RB_PR_NUMBER / findings 字段非法 → FAIL
    → FAIL 后: retry（RB_ATTEMPT 递增）→ 仍 FAIL → 整 run 失败（全有或全无）
```

## 2. State Flow（状态流）
**评测状态机**：
```
candidate_loaded ──overlap>0──▶ eval_ready ──per-PR──▶ scored ──aggregate──▶ reported
      │  overlap==0(allowEmpty=off) → FAIL（strictInputProblems）
      ▼
scored ──checkpoint 指纹匹配──▶ resumed（跳过已完成 PR）
      │  指纹不匹配 → 全部重评
      ▼
ingest 后：golden 集合变化 → removeCheckpoint → 下一次评分强制重评
```
**checkout 缓存状态机**（git.ts L103-126）：
```
exists(.git)?
  ├─ no → mkdir + clone --filter=blob:none + checkout sha
  └─ yes → rev-parse HEAD==sha AND status --porcelain 空?
       ├─ yes → clean -ffdx → 复用
       └─ no  → rmSync → 重建
```
**状态转换证据**：`assertHexRef(sha); parseNwo(nwo)` 在任何 fs 操作前（git.ts L93-100）；镜像引用状态 = digest（不可变）。

## 3. Data Flow（数据流）
```
candidate findings.json ──loadCandidateFindings──▶ byPR: Map<prKey, PRFindings>
golden/*.json ──GoldenStore──▶ parsed: Map<prKey, GoldenSet>（驻内存）
matcher: Finding[] × GoldenEntry[] ──file 分组/chunk──▶ RawCorrespondence[]（many-to-many）
resolveForScoring ──▶ ScoringInput{matched, matchedToGolden, unmatched, coveredGolden}
classifier（unmatched 仅）──▶ ClassifiedFinding[] ──unmatched-classifications──▶ Map<candidateIndex, cls>
scorePR ──▶ MetricSet{grounded/augmented P/R, novel_tp_count, ...} + by_severity/by_category
aggregate ──▶ {macro, micro, pr_count, per_pr}
usage: candidate usage ──aggregateCandidateUsage──▶ run 级 token/cost 统计
```
- Edge 回溯: `coveredGolden` 由 matcher raw matches 直接注入（match-types.ts L50-52）；`matchedToGolden` 第一个未认领 golden（L54-61）。
- 数据一致性守卫：`candidateIdentityProblem` 比对 4 字段（input-validation.ts L64-75）；usage 字段非负有限数校验（L244-262）。[Reconciliation 修正] 第三处 identity 守卫：classify 前 manifest entry ↔ golden 四字段比对（repo/pr_number/head/base，不等 throw，eval.ts L415-425）——防"manifest 缓存与 golden 权威错配时误用 manifest 文本作 judge 上下文"。

## 4. Evidence Flow（证据流）
**golden 证据的产生链**（docs/EXTRACTION.md + METHODOLOGY §4-5）：
```
human review comments + inferred author actions（PR 历史）   → review_comment
LLM review passes（claude-sonnet-4.6/gemini-2.5-pro/gpt-4o）→ llm_review
CCR system（gpt-5.5/claude-opus-4.7）                       → ccr
deterministic tools（Semgrep）                               → tool
      ↓ 语义去重（matcher，§5.4）→ 多源 findings 融合
      ↓ 统一 rubric 分类（Claude Sonnet 5 LLM grader）→ tp_fp/severity/category/scope
      ↓ 资深工程师独立复标验证（96.6% 一致，Blog 报告口径）→ golden set
      ↓ 版本化（golden 文件 + manifest 冻结）
```
**judge 的判定证据流**（classify.ts）：
```
classifier session 收到 PR 上下文（diff≤100k/SHA/title/body）
      ↓ 工具验证（read/grep/find/ls）实际代码
      ↓ 输出 JSON: tp_fp_justification/severity_justification/category_justification
      ↓ session 持久化到磁盘（sessionDir）→ 可 `pi --resume/--export` 审计
```
- 权威性说明：golden 标签的"真值"来自方法论定义（§5），LLM grader 是执行者，human 复标是抽样校验——三层证据权威不同。

## 5. Authority Flow（权威流）
```
GitHub 账号/org ──▶ manifest contacts（≤8 人：可 start runs/publish/rotate credentials）
      ↓ schema/agent-manifest.schema.json contacts
Portal 验证 manifest → 批准 → 授权 run
      ↓
Runner 权限面：egress 白名单（网络权威）[AGENT_CONTRACT.md L102-112]
      + model_api_url 主机自动授权
      + secrets 只读注入（env/文件，write-only 存储）
      ↓
Judge 判定权威：matcher/classifier 模型是评分唯一权威（provider/model CLI 指定）
      ↓
被测 agent：无 GitHub API、无上游网络路径、无其它分支（冻结）
```
- Edge 回溯: egress 被拒 → 首 PR 失败 → 停 run（AGENT_CONTRACT.md L105-106）；label 未被 agent 输出消费 → run 点名（L38）。
- 反例 Edge: `RB_DELETE_GOLDEN_AFTER_LOAD` 是"runner 本地文件权威"的撤销（删盘后 store 仍权威）。

## 6. Memory Flow（记忆流）
```
持久记忆：
  - golden/（219 文件）：长期真值库（GoldenStore 驻内存后 disk 可删=运行期无记忆）
  - corpus/manifest.json + test/：基准 PR 库
  - checkpoint sidecar（scoring/results.checkpoint.json）：已完成 PR 分数（指纹隔离）
  - classifier session 日志（sessionDir）：judge 行为审计
  - mirror 仓库（review-bench/<owner>_<repo>）：冻结代码快照
  - findings/ 输出：每次 run 独立目录（OFFLINE_EVAL: run=$root/runs/...）
运行期记忆：
  - manifestCaches：manifest → Map<prKey, entry>（进程内缓存）[eval.ts L340-356]
  - GoldenStore.raw/parsed：golden 双份驻内存 [L194-236]
  - classifier session 内上下文：一个 PR 内 findings 间共享（避免重复探索）[classify.ts L93-150]
不记忆：
  - 容器间零状态（每 PR 新容器，nothing carries over）[AGENT_CONTRACT.md L3-6]
  - matcher session 单次性（abort/dispose 后无残留）[matcher.ts L373-377]
```

## 7. Policy Flow（策略流/治理闭环）
```
策略定义：AGENT_CONTRACT.md（运行契约）+ agent-manifest.schema.json（治理 schema）+ METHODOLOGY.md（评分策略）
      ↓
策略固化：manifest 强制（image digest / egress / secrets shape / contacts / configuration）
      ↓
策略执行（Enforcement）：
  - run 期：docker 挂载/环境变量/网络 proxy/超时/输出校验（try-agent.sh + runner）
  - 评分期：strict 门 / identity 校验 / checkpoint 指纹 / golden 保密
      ↓
策略审计与反馈：session 日志 / usage / producers 报告（npm run report:producers）
      ↓
未来决策（Feedback Loop）：METHODOLOGY §11（Feedback and Labeling Improvements）
  + §8 Threats→标注→方法论演进
```
- Edge 回溯: `report:producers` 由 scripts/golden-producers.ts + lib/producer-mix.ts 实现（package.json scripts）；§11 在 docs/METHODOLOGY.md L772 起。
- Policy→Flow 交叉校验：所有 enforcement Edge 都能在代码/契约文档回溯（无一悬空）。

## Flow→KO 交叉校验门
| KO | 依赖 Flow Edge | 冲突？ |
|---|---|---|
| KO-01 | matcher 校验 → throw（Data Flow） | 无冲突 |
| KO-05 | checkpoint 指纹 / egress / digest（State+Authority Flow） | 无冲突 |
| KO-07 | classifier 只读工具 + 脏 checkout 重建（Evidence+State Flow） | 无冲突 |
| KO-08 | Threats→反馈闭环（Policy Flow） | 无冲突 |

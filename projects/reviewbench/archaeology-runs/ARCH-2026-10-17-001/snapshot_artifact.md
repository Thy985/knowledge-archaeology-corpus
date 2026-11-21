# Repository Snapshot Artifact — ARCH-2026-10-17-001 · ReviewBench

## 快照表
| 项 | 值 |
|---|---|
| repository | https://github.com/review-bench/ReviewBench.git |
| HEAD | `6e37127defb2da675d4b59da61cf38d12ded1ad8` |
| branch | main |
| commit time | 2026-10-16 10:39:23 -0400（"Onboard diffox ultra (diffox-ultra) (#87)"） |
| version | ReviewBench v1（GitHub 官方 2026-10-05 发布；仓库 2026-09-04 创建） |
| 规模 | 307 文件（含 golden 219 + test 25 + scripts + docs + tests）；TypeScript/Node |
| license | MIT |

## 项目基础地图
1. **语言/运行时**：TypeScript（Node ≥20）；npm；无 Go/Rust。TypeScript 文件集中在 scripts/ 与 schema/。
2. **入口**：`scripts/try-agent.sh`（本地跑 agent on test/full set）、`scripts/eval/eval.ts`（judging CLI）、`scripts/golden/pipeline.ts`（golden 构建管线）、`scripts/golden-producers.ts`（producer 分布报告）。
3. **核心模块**：
   - `golden/`（219 个 PR 的 ground-truth findings JSON）
   - `corpus/manifest.json`（219 PR manifest）+ `corpus/test/`（25 PR 测试集）
   - `scripts/eval/`（eval.ts/matcher.ts/scorer.ts/judge-call.ts/input-validation.ts/output-paths.ts/prompt-format.ts/print-summary.ts/unmatched-classifications.ts/checkpoint-fingerprint.ts）
   - `scripts/golden/`（pipeline.ts/classify-adapter.ts）+ `scripts/classifier/`（classify.ts/prompts.ts/types.ts）
   - `schema/agent-manifest.schema.json` + `validate-manifest.mjs`
   - `agents/*.json`（18 个已 onboard agent manifest）
4. **核心数据结构**：
   - Finding：`{file, start_line, end_line, message, tp_fp, severity, category, scope, difficulty, context_required, tp_fp_justification, severity_justification, category_justification, producer, source:{type,alignment}}`
   - Golden set：`{pr_key, pr:{repo,pr_number,base,head}, findings[]}`
   - Agent manifest：`{name, display_name, provider, image, configuration[], secrets{env,files}, egress[], pr_timeout_seconds, contacts[], notes, image_visibility}`
   - Scorer 指标：grounded_precision/recall、augmented_precision/recall、novel_tp_count/yield、matched/unmatched 计数
5. **状态/生命周期**：PR 快照（base/head 锁定在 review-time，head 后 commit 不可见）→ extraction（human comments + LLM passes + deterministic tools）→ labeling（Claude Sonnet 5 classifier + human audit）→ dedup（用 matcher）→ golden → 候选 agent 评测（fresh container per PR）→ matching → classification of unmatched → scoring。
6. **测试体系**：tests/ 10 个 .test.ts（checkpoint-fingerprint / eval-input-validation / judge-call / matcher-response / model-resolution / output-paths / producer-mix / prompt-format / scorer / unmatched-classifications）；CI 在 .github/。
7. **配置**：package.json（scripts）+ tsconfig；provider/model 通过 `--provider/--model` 选 LLM judge；API key 走环境变量（pi model registry）。
8. **权限与治理机制**：AGENT_CONTRACT（fresh container per PR、无跨 PR 状态、RB_* env 契约、secrets 经 env/files 注入、egress 白名单、pr_timeout_seconds、image digest pin、agent 不知自己是 benchmark）；schema 校验 agent-manifest；镜像 digest 引用。
9. **外部依赖**：Node、docker（runner）、git、jq、pi（badlogic/pi-mono SDK 用于 provider auth + LLM calls + read-only classifier）、LLM providers（Anthropic/OpenAI/Azure/Gemini/DeepSeek/OpenRouter）、GitHub API（extraction）、仓库镜像（review-bench/<owner>_<repo> 防上游删除）。

## 考古焦点（6 条）
1. **grounded vs augmented 双指标**（docs/METHODOLOGY.md §7.1-7.2 + scorer.ts）——grounded 只算匹配到 golden 的 TP；augmented 纳入 classifier 判定的 novel TP；golden 非穷尽的诚实承认。
2. **多源 golden 构建**（§4.1/4.2/5.5 + EXTRACTION.md）——human review + ccr + LLM review agents + deterministic tools（Semgrep）；TP rate 差异巨大（claude-sonnet-4.6 81.8% vs gpt-4o 4.6%）；dedup 防 recall 分母膨胀。
3. **同一 classifier 双角色**（§5.3/5.4 + scripts/classifier/prompts.ts）——Sonnet 5 classifier 同时用于 corpus labeling 与 unmatched candidate classification；hill-climbed against human dev set；TP/FP 96.6% agreement；47 findings 人工修正；classifier 校准需重算下游指标。
4. **LLM matcher 规则**（§6.2 + matcher.ts）——many-to-many；判据"fixing one would fix (or substantially address) the other"；同位置不同问题不算匹配；代码上下文注入 prompt。
5. **AGENT_CONTRACT 隔离设计**（AGENT_CONTRACT.md + RUNNER.md + schema）——fresh container per PR、镜像 digest pin、secrets env/files、egress 白名单、RB_* env 契约、git safe.directory、agent 不知道自己在 benchmark 中。
6. **Threats to Validity 显式清单**（§8）——golden incompleteness / classifier dependence / matcher dependence / producer bias / offline-online convergence / PR-level snapshot；未弱化证据标准。

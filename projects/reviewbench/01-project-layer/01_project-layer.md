# 01 · Project Layer — ReviewBench（项目地图）

## 定位
GitHub 官方（Copilot 团队）发布的 AI 代码审查离线评测基准；仓库 review-bench/ReviewBench（MIT）。以 219 个公开真实 PR 的 golden findings 为 ground truth，衡量候选 agent 的 grounded/augmented precision & recall。

## 架构（数据→证据→判定管线）
```
真实 PR（219）→ manifest（base/head 快照）
        → extraction（human comments + LLM passes + deterministic tools）→ raw findings
        → labeling（Claude Sonnet 5 classifier + human audit + dedup）→ golden set
        → 候选 agent（fresh container per PR）→ candidate findings
        → matching（LLM matcher, many-to-many）→ classification of unmatched（同 classifier）
        → scoring（grounded/augmented × precision/recall + 分层）→ leaderboard（3 runs 均值）
```

## 核心模块
| 模块 | 路径 | 职责 |
|---|---|---|
| golden/ | `golden/*.json`（219） | 每 PR 的 ground-truth findings（含 justification 证据链） |
| corpus/ | `corpus/manifest.json` + `corpus/test/` | 219 PR manifest + 25 PR 测试集 |
| scripts/eval/ | `eval.ts/matcher.ts/scorer.ts/judge-call.ts/input-validation.ts/output-paths.ts/prompt-format.ts/checkpoint-fingerprint.ts/print-summary.ts/unmatched-classifications.ts` | 评测管线（匹配/分类/计分） |
| scripts/golden/ | `pipeline.ts/classify-adapter.ts` + `scripts/golden-producers.ts` | golden 构建 + producer 分布报告 |
| scripts/classifier/ | `classify.ts/prompts.ts/types.ts` | 分类器（TP/FP/severity/category 标注） |
| schema/ | `agent-manifest.schema.json` + `validate-manifest.mjs` | agent manifest 校验 |
| agents/ | `*.json`（18 个 onboard manifest） | 已注册审查 agent 的声明式配置 |
| docs/ | `METHODOLOGY.md/HARNESS.md/EXTRACTION.md/JUDGING.md/OFFLINE_EVAL.md/ONBOARDING.md/RUNNER.md/JUDGING_INPUT.md` | 方法论与流程 |
| tests/ | 10 个 `.test.ts` | 单元测试（scorer/matcher/judge-call/validation/fingerprint 等） |

## 核心数据结构
- **Finding**：`{file, start_line, end_line, message, tp_fp, severity, category, scope, difficulty, context_required, tp_fp_justification, severity_justification, category_justification, producer, source:{type,alignment}}`
- **GoldenSet**：`{pr_key, pr:{repo,pr_number,base,head}, findings[]}`
- **AgentManifest**：`{name, display_name, provider, image(digest), configuration[], secrets{env,files}, egress[], pr_timeout_seconds, contacts[], notes, image_visibility}`
- **ScoreResult**：grounded/augmented × precision/recall + novel_tp_count/yield + matched/unmatched 计数

## Agent 接入双路径（HARNESS.md §1）
- **GitHub adapter**：agent 作为 GitHub App/integration 运行，需要真实 PR on GitHub（fork per source repo）。
- **Local adapter**：agent 接受 diff/repo path/API call 并返回结构化输出，无需 GitHub。
- 两条路径产出同一 normalized output，下游（matching/classification/scoring）完全一致。

## 生命周期
corpus 构建（extraction→labeling→dedup）→ golden 发布 → 候选 agent 三阶段（manifest 校验→test run→official 3-runs）→ leaderboard；分类器校准→重算下游指标；golden 可经 human adjudication 修订（§11）。

## 状态机（评测侧）
candidate findings → input-validation → raw match（many-to-many）→ claimed/unmatched 分离 → unmatched 分类 → scorer 汇总。

## 测试体系
10 个测试覆盖：scorer 公式、matcher response 解析、judge-call 超时、input-validation、output-paths、checkpoint-fingerprint、producer-mix、prompt-format、model-resolution、unmatched-classifications。

## 配置与治理
- 配置：package.json scripts + tsconfig；`--provider/--model` 选 judge LLM；API key 环境变量（pi registry）。
- 权限/治理：AGENT_CONTRACT（fresh container、RB_* env、secrets env/files、egress 白名单、pr_timeout、image digest pin、agent 不知自己在 benchmark）；offline 结果不可提交 leaderboard；manifest schema 校验。
- 外部依赖：Node≥20、docker、git、jq、`@earendil-works/pi-coding-agent`（^0.74.0，pi SDK 的 npm 包名；README provider 文档链接仍指向 badlogic/pi-mono，疑似文档过时，见 06 NEEDS_HUMAN_REVIEW）、LLM providers、GitHub API、仓库镜像（review-bench/<owner>_<repo>）。

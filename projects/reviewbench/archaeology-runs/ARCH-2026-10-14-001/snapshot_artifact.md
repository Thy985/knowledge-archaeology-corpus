# Repository Snapshot Artifact · ARCH-2026-10-14-001 · ReviewBench

## 快照事实表

| 项 | 值 | 证据 |
|---|---|---|
| run_id | ARCH-2026-10-14-001 | job_manifest.md |
| project | ReviewBench（GitHub 官方 AI 代码审查离线基准） | README.md L5 |
| repository | https://github.com/review-bench/ReviewBench.git | git remote |
| branch | main | `git branch --show-current` |
| HEAD commit | `c17b05be9c90b85a1fc012ce39e455e2d09a7255` | `git log -1` |
| commit message | "Onboard block/review (block-review) (#71)" | `git log -1 --format=%s` |
| commit timestamp | 2026-10-13 10:08:07 -0700 | `git log -1 --format=%ci` |
| 浅克隆方式 | `git clone --depth 1` | 阶段③执行记录 |
| 体积 / 文件数 | 12M / 303 files（不含 .git） | `du -sh` / `find | wc -l` |
| 语言分布 | 240 json / 33 ts / 16 md / 5 yml / 3 mjs / 1 sh / 1 svg | `find | sed | uniq -c` |
| TypeScript 总量 | 5758 行（33 文件） | `xargs wc -l` |
| 版本 | package.json `version: 1.0.0`，name `reviewbench`，type `module` | package.json |
| 许可证 | MIT | LICENSE / package.json / GitHub API |
| GitHub 元数据 | created 2026-09-04、pushed 2026-10-13、50★、8 forks、5 open issues、size 2716KB、homepage https://review-bench.ai、topics [ai, ai-code-review, benchmark, code-review] | GitHub API repos/review-bench/ReviewBench |

## 项目基础地图

### 定位
ReviewBench 是 GitHub 官方开发的开源、可复现的 AI 代码审查评测基准：对每个 PR 提供人工审校的 golden findings 集合作为 ground truth，把候选 review agent 产出的 findings 与之匹配、分类、评分，产出 grounded/augmented precision/recall 指标。（README.md L5-L9）

### 目录结构（顶层，源自 `ls` 实证）
```
AGENT_CONTRACT.md     # Agent 契约：docker 调用协议、RB_* 环境变量、输出 schema、exit code 语义
agents/               # 14 个 agent manifest JSON（codex/codex-2/codex-test/deepseek-harness-reviewer/example-reviewer/hermes-reviewer/block-review 等）
corpus/               # manifest.json（219 PR 全量）+ test/（test.json 25 PR + findings.json + picks.md）
golden/               # 219 个 golden findings JSON（每 PR 一个，文件名 <owner>_<repo>_<pr>-<head8>）
docs/                 # 8 md：METHODOLOGY/EXTRACTION/HARNESS/JUDGING/JUDGING_INPUT/RUNNER/ONBOARDING/OFFLINE_EVAL
examples/codex-cli/   # 完整 vendor agent 示例（agent.mjs + parse.mjs + Dockerfile + README）
schema/               # agent-manifest.schema.json（JSON Schema 2020-12）+ validate-manifest.mjs
scripts/              # 25 ts + try-agent.sh
tests/                # 10 个 *.test.ts
package.json          # scripts: judge/sync:test-set/check:test-set/check:golden/report:producers/typecheck/test:judge/test
```

### 核心模块（scripts/，行数实证）
| 模块 | 文件 | 行数 | 职责 |
|---|---|---|---|
| 评测 CLI 主入口 | scripts/eval/eval.ts | 885 | candidate 加载、golden 加载、校验、checkpoint、并发、匹配→分类→评分管道、ingest |
| 评分器 | scripts/eval/scorer.ts | 595 | §7 grounded/augmented 四指标 + severity/category 分层 + macro/micro 聚合 |
| Classifier 会话 | scripts/classifier/classify.ts | 412 | 每 PR+SHA 一个 pi session，read/grep/find/ls 工具验证，逐个 findings 分类 |
| LLM 匹配器 | scripts/eval/matcher.ts | 380 | 同文件分组 + chunk(10) 交叉比较，many-to-many 对应，JSON 契约校验 |
| Test set 同步 | scripts/sync-test-set.ts | 366 | test set 生成/校验（--check） |
| Git 快照操作 | scripts/extraction/git.ts | 330 | bare clone / checkout@SHA / diff（3-dot）/ 行区域 unchanged 检测 / 代码区域提取；全 argv、hex-ref/nwo 校验、LFS skip |
| 输入校验 | scripts/eval/input-validation.ts | 296 | candidate JSON 形状校验、PR identity 校验、strict 门 |
| Classifier prompts | scripts/classifier/prompts.ts | 219 | CLASSIFIER_SYSTEM_PROMPT v3.0.0（TP/FP/severity/category/scope/difficulty/context 契约 + 校准例） |
| Golden 管道 | scripts/golden/pipeline.ts | 209 | dedup（new vs golden 用 matcher）+ merge（novel 回灌） |
| 用量统计 | scripts/lib/usage.ts | 190 | candidate usage 聚合 |
| Golden 分类适配 | scripts/golden/classify-adapter.ts | 131 | 把 unmatched findings 送给 classifier |
| 详情输出 | scripts/eval/print-summary.ts | 102 | 汇总打印 |
| Producer 混合 | scripts/lib/producer-mix.ts | 84 | golden producer 构成分析（report:producers） |
| 类型 | scripts/lib/types.ts (72) / match-types.ts (68) | | PR/Finding/ManifestEntry 类型、prKey、resolveForScoring |
| 未匹配分类 | scripts/eval/unmatched-classifications.ts | 63 | unmatched→classification map + novel TP 选择 |
| Judge prompt 渲染 | scripts/eval/prompt-format.ts | 62 | findings 注入防护（escape + data-not-instructions） |
| Judge 超时 | scripts/eval/judge-call.ts | 57 | AbortController 超时（10min 基础 + 2min/finding） |
| checkpoint 指纹 | scripts/eval/checkpoint-fingerprint.ts | — | judge/prompts/PRs/candidates/golden/manifest 指纹 |
| 输出路径 | scripts/eval/output-paths.ts | — | results/details/checkpoint sidecar |
| Golden 校验 | scripts/validate-golden.ts | — | golden 文件 schema 校验 |

### 数据集规模（仓库实际内容实证）
- `corpus/manifest.json`：**219 条 PR 记录**（repo/pr_number/pr_url/base/head/title/body/nwo 等）
- `corpus/test/test.json`：**25 条** test set PR
- `golden/`：**219 个文件**（`ls golden | wc -l`）
- 样例 golden（AA-Factory_aafactory-prototype_17-a1978cb7.json）：17 findings，含 file/start_line/end_line/message/severity/category/scope 等
- 样例 test findings（corpus/test/findings.json）：示例含 severity=high/category=security 的路径穿越 findings（TechDebtMCP PR）
- 方法论报告数据（docs/METHODOLOGY.md §5.5，非仓库统计、标注来源）：full set 4632 findings / 2623 TP / 56.6% TP rate；test set 565 / 352 / 62.3%

### Agent 契约要点（AGENT_CONTRACT.md）
- 每个 PR 起全新 docker 容器：`-v <checkout@head>:/work/repo -v <pr metadata>:/work/pr:ro -v <output>:/work/out` + 环境变量 RB_NWO/RB_PR_NUMBER/RB_BASE/RB_HEAD/RB_AGENT/RB_DIFF/RB_PR_JSON/RB_OUT/RB_CONFIG_*/RB_MODEL_BASE_URL/RB_ATTEMPT
- 输出：一个 JSON 文件（pr + agent + findings[]），field rules：file 相对路径、行号指向 RB_HEAD、message 说明"错在哪为什么"
- Exit codes：0+有效文件=Accepted；0+空 findings=Accepted（看了没发现）；非 0/缺文件/坏 JSON/head 或 PR 号不符=Fail（重试后失败 run）
- Run 全有或全无：一个 PR 失败 → 整 run 失败
- 限制：linux/amd64；30min/PR 默认（可 1h）；网络白名单（manifest egress + model_api_url 主机自动加）；无 GPU；标准 runner
- "会吓到你的"：repo 冻结（minimised clone、shallow log、无其他分支、无 upstream 网络路径）、LFS 大文件是 pointer、无 GitHub API、截断输出是常见失败（dev 时丢过一个 PR）

### 配置 / 权限与治理机制
- agent-manifest.schema.json：name 模式 `^[a-z0-9][a-z0-9._-]{1,63}$`（producer 标识）；image 必须 **GHCR digest-pinned**（`@sha256:64hex`，可复现、字节不可变）；configuration ≤6 项（→RB_CONFIG_*）；secrets 只声明 **SHAPE 不存值**（write-only）；egress ≤32 主机白名单；model_api_url 自动入白名单；pr_timeout_seconds 60–3600 默认 1800；contacts ≤8（操作权限名单：start runs/publish/rotate credentials）；image_visibility public/private
- 评测保密：golden 加载后可 `RB_DELETE_GOLDEN_AFTER_LOAD=1` 删盘（防后续 job 读到标签）
- checkpoint 指纹：judge 模型/prompt hash/PR 集/candidates/golden/manifest 任何变化 → 丢弃 checkpoint 重评

### 测试体系
- tests/ 10 个文件：checkpoint-fingerprint / eval-input-validation / judge-call / matcher-response / model-resolution / output-paths / producer-mix / prompt-format / scorer / unmatched-classifications
- `npm test` = `npm run typecheck`（tsc --noEmit）+ `node --import tsx --test tests/**/*.test.ts`
- 无集成测试（需要 LLM/网络）

### 外部依赖
- dependencies: `@earendil-works/pi-coding-agent ^0.74.0`（matcher/classifier 的 agent session 运行时：ModelRegistry/AuthStorage/SessionManager/SettingsManager）
- devDependencies: tsx ^4.21.0 / typescript ^6.0.3 / @types/node ^25.6.0
- 运行时外部：docker（run agent image）、git、LLM provider API（judge/model）

### 边界与例外（考古标注）
- 代码库不含官方 leaderboard 后端 / 门户（仅在 review-bench.ai 站点）；仓库是公开的 benchmark 数据集 + judge CLI + 本地 runner
- `corpus/test/findings.json` 是**示例 golden findings**（test set 25 PR 的金标样例），不是 candidate 输出
- METHODOLOGY.md 中 103.9M PR / 96.6% 等数字为 GitHub 官方报告口径，仓库内无原始计算脚本（docs 声明 + Blog 佐证）
- scripts/golden-producers.ts + lib/producer-mix.ts 可复现 producer 构成表（`npm run report:producers`）

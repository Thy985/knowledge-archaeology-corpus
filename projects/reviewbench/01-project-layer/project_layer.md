# 01 · Project Layer · ReviewBench

> 项目地图（L0 事实底座）。所有条目均可回溯到仓库实际内容；来源列给出文件路径。

## 1.1 是什么
开源 AI 代码审查评测基准（v1.0.0, MIT）。对一个 PR 提供人工审校的 golden findings，把候选 review agent 的 findings 与之匹配、分类、评分。（README.md；package.json）

## 1.2 怎么运行（用户可见流程）
1. **候选方**：把 review agent 打包成 docker 镜像（遵循 AGENT_CONTRACT.md），在 portal 注册 manifest（image digest、egress、secrets shape、configuration、contacts）。
2. **运行**：对每个 benchmark PR 起全新容器（mount checkout@head + diff.patch + pr.json），agent 写 findings.json；检查契约合规（head/pr_number 匹配、findings 字段规则），单 PR 失败→retry→run 失败（全有或全无）。
3. **评分**：`npm run judge -- --candidate <dir> --provider <p> --model <m>`：candidate 与 golden 做 LLM 语义匹配 → 未匹配项 LLM 分类 → grounded/augmented precision/recall → macro/micro 聚合。
4. **榜单**：官方结果只能由 review-bench.ai portal 的 final run 产生；离线 `npm run judge` 仅供开发调优。（docs/OFFLINE_EVAL.md L7-11；docs/JUDGING.md L7-12）

## 1.3 核心模块与职责（证据：行数实证）
| 模块 | 文件 | 职责 |
|---|---|---|
| 评测主管道 | scripts/eval/eval.ts (885) | candidate 加载→golden 加载→identity 校验→matcher→resolveForScoring→classifier→scorePR→aggregate→（可选 ingest）→checkpoint/resume |
| 评分器 | scripts/eval/scorer.ts (595) | §7 四指标 + severity/category 分层 + macro/micro |
| LLM 匹配器 | scripts/eval/matcher.ts (380) | 同文件分组 + chunk 交叉 LLM 匹配（many-to-many） |
| Classifier | scripts/classifier/classify.ts (412) + prompts.ts (219) | 每 PR 一个 pi session，read/grep/find/ls 工具验证，TP/FP+severity/category/scope/difficulty/context 分类 |
| Golden 管道 | scripts/golden/pipeline.ts (209) + classify-adapter.ts (131) | dedup（新 findings vs golden）+ merge（novel TP 回灌） |
| Git 快照层 | scripts/extraction/git.ts (330) | bare clone / checkout@SHA / 3-dot diff / region-unchanged / code region |
| 输入校验 | scripts/eval/input-validation.ts (296) | candidate JSON shape / PR identity / strict 门 |
| 用例/契约 | AGENT_CONTRACT.md；schema/agent-manifest.schema.json | agent 调用协议与 manifest 治理 schema |
| 数据集 | corpus/manifest.json (219) + corpus/test/test.json (25) + corpus/test/findings.json；golden/*.json (219) | 基准 PR 集 + golden findings |
| 示例 agent | examples/codex-cli/（agent.mjs+parse.mjs+Dockerfile） | vendor agent 实现模板 |
| 测试 | tests/ 10 *.test.ts | scorer/matcher-response/prompt-format/judge-call/input-validation 等纯逻辑单测 |

## 1.4 核心数据结构
- **ManifestEntry**（scripts/lib/types.ts）：`{ repo, pr_number, pr_url, base, head, title, body, nwo }`；`prKey(entry) = <nwo 斜杠转下划线>_<pr_number>-<head8>`
- **Finding**：`{ producer, file, start_line, end_line, message, source? }`
- **GoldenEntry**：Finding + `tp_fp, severity, category, scope?, difficulty?, context_required?, justifications, producer, source`
- **RawCorrespondence**：`{ candidate_index, matched_golden_indices[] }`（many-to-many）
- **ScoringInput**：`{ matched, matchedToGolden, unmatched, coveredGolden, raw }`（resolveForScoring 输出）
- **PRScoreResult / MetricSet / AggregateMetrics**（scorer.ts）：per-PR 指标 + macro/micro
- **Checkpoint**：`{ fingerprint, completed: Record<prKey, PRScoreResult>, details }`

## 1.5 生命周期与状态
- **评测管道状态**：candidate 加载 → 与 golden overlap 计算 → per-PR `evalPR`（dedup→scoring→classify unmatched→scorePR）→ aggregate →（可选 ingest novel TPs 写回 golden/）
- **Checkpoint 生命周期**：saveCheckpoint（每完成 PR 落盘 sidecar）→ resume 时指纹匹配才复用，否则丢弃重评；ingest 后 removeCheckpoint（eval.ts L288-309, 260-286）
- **Container run 生命周期**：每 PR 一个容器（--rm），attempt 编号 RB_ATTEMPT；失败 retry；run 全有或全无（AGENT_CONTRACT.md L91-92）

## 1.6 配置体系
| 配置面 | 机制 |
|---|---|
| CLI | `npm run judge -- --candidate/--golden/--manifest/--output/--ingest/--concurrency/--limit/--strict/--no-strict/--allow-empty/--provider/--model/--repo-dir`（eval.ts L113-187） |
| 环境变量 | `REVIEW_BENCH_JUDGE_TIMEOUT_MS`（judge 超时，默认 10min）；`MIRROR_ORG`（mirror org，默认 review-bench）；`RB_DELETE_GOLDEN_AFTER_LOAD`（保密）；`TRY_AGENT_WORK`（try-agent 工作目录） |
| Agent manifest | name/display_name/image(digest-pinned)/configuration(≤6)/secrets(shape only)/egress(≤32)/model_api_url/pr_timeout_seconds(60-3600)/contacts(≤8)/image_visibility（schema/agent-manifest.schema.json） |
| 评测运行 env | RB_NWO/RB_PR_NUMBER/RB_BASE/RB_HEAD/RB_AGENT/RB_DIFF/RB_PR_JSON/RB_OUT/RB_CONFIG_*/RB_MODEL_BASE_URL/RB_ATTEMPT（AGENT_CONTRACT.md） |

## 1.7 权限与治理机制
- **manifest 治理**：secrets 只声明 SHAPE 不存值（write-only 收集）；contacts 是"谁能代表 vendor 行动（start runs/publish/rotate credentials）"的授权名单（schema agent-manifest.schema.json secrets/contacts 字段）
- **网络治理**：egress 白名单 + forward proxy（HTTP_PROXY/HTTPS_PROXY + NODE_USE_ENV_PROXY=1）；model_api_url 主机自动加入；拒绝其他主机 → 首 PR 失败即停 run 并点名（AGENT_CONTRACT.md L102-112）
- **保密治理**：golden 可加载后删盘（RB_DELETE_GOLDEN_AFTER_LOAD=1）；GoldenStore 驻内存后 disk 可删（eval.ts L189-236）
- **可复现治理**：image digest-pinned（字节不可变）；manifest/PR/diff 用 mirror 冻结快照（upstream rot 不影响）；checkpoint 指纹（judge/prompts/PRs/candidates/golden/manifest）
- **git 安全治理**：argv 执行 + hex-ref 校验 + nwo 校验（防命令/选项注入与路径穿越）

## 1.8 外部依赖
- `@earendil-works/pi-coding-agent ^0.74.0`（matcher/classifier 的 LLM session 运行时：ModelRegistry/AuthStorage/DefaultResourceLoader/SessionManager/SettingsManager）
- devDeps: tsx / typescript ^6.0.3 / @types/node ^25.6.0
- 运行时外部：docker（run agent image）、git（checkout/diff）、LLM provider API

## 1.9 边界与例外
- 本仓库不含 leaderboard 后端/门户代码（仅在 review-bench.ai）；仓库 = 公开数据集 + judge CLI + 本地 runner + 契约文档
- corpus/test/findings.json 是示例 golden findings（非 candidate 输出）
- METHODOLOGY.md 的 103.9M PR / 96.6% 复标一致率等为官方报告口径，仓库内无原始计算脚本（Blog + docs 声明）
- 无集成测试（judge 需真实 LLM/网络）；tests/ 10 个为纯逻辑单测

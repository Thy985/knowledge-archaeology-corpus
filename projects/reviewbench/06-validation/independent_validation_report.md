# Independent Validation Report · ReviewBench（ARCH-2026-10-14-001）

> 独立 Auditor 盲重建报告。Auditor 不把考古产物当事实来源：独立重读仓库（重点：scorer.ts / checkpoint-fingerprint.ts / types.ts / eval.ts main 段 / input-validation.ts / tests/ 清单）建立 Independent Findings 后与产物对比。**未修改任何考古产物**；修正项将在阶段⑥ Reconciliation 合并。Auditor 与考古者共享仓库但独立取证，以下判定均附盲重建证据。

## 1. 判定统计
| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 10 | 见 §2 |
| PARTIALLY_CONFIRMED | 1 | EK-22（指纹细节） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 2 | manifest↔golden identity 比对；RB_DELETE_GOLDEN×ingest 互斥 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 2. 3+ 成功（CONFIRMED 抽查——盲重建独立取证）
1. **computeMetrics 的 null 保护（EK-06）**：盲重建 scorer.ts L150-241：`matched_count > 0 ? matched_tp_count / matched_count : null`；`augmented_recall` 分母 `golden_tp_count + unmatched_tp_count`（agent-dependent yardstick 与 METHODOLOGY §7.2 一致）；`novel_tp_yield` 同式。→ CONFIRMED。
2. **checkpoint 指纹的输入构成（EK-22 主体）**：盲重建 checkpoint-fingerprint.ts 全文：SHA-256 + canonicalize（key 排序、undefined 过滤）+ `provider/modelId | hash(classifierPrompt) | hash(matcherPrompt) | hash({pr_keys,candidates,golden_sets,manifest_entries})`。→ 指纹确实隔离 judge 配置与全部输入；"配置/输入变化 → 指纹变化 → 重评"主体成立 → CONFIRMED（细节偏差见 §3.1）。
3. **prKey 格式（01 文件）**：盲重建 types.ts L22-28：`repo.replace("https://github.com/","").replace("/","_")` + `_${pr_number}-${head.slice(0,8)}`，与 snapshot/01 所述 head8 一致 → CONFIRMED。
4. **classifier 调用条件（04 Flow Atlas）**：盲重建 eval.ts L405-406：`if (unmatchedFindings.length > 0)` 才分类；超时预算 `JUDGE_CALL_TIMEOUT_MS + JUDGE_CALL_PER_FINDING_MS * unmatched.length`（EK-14 的 2min/finding 语义）→ CONFIRMED。
5. **golden 驻内存/删盘（EK-23）**：盲重建 eval.ts L689 区：`RB_DELETE_GOLDEN_AFTER_LOAD=1` → `deleteFromDisk()`，注释 "labels stay in memory only, so nothing that runs later in the job can read them from the filesystem" → CONFIRMED。
6. **strict 输入门（EK-20）**：盲重建 input-validation.ts L193-235 行为与产物一致 → CONFIRMED。
7. **测试体系规模（01 文件）**：盲重建 `ls tests/` = 10 个 *.test.ts（checkpoint-fingerprint/eval-input-validation/judge-call/matcher-response/model-resolution/output-paths/producer-mix/prompt-format/scorer/unmatched-classifications）→ CONFIRMED（与 01 所述 10 个一致）。
8. **评测主流程分支（04 Control Flow）**：盲重建 eval.ts main：loadCandidateFindings → GoldenStore → buildEvalInputSummary → per-PR → aggregate 顺序一致 → CONFIRMED。
9. **EK-30 golden 数字**：盲重建 METHODOLOGY §5.5 数字一致（4632/2623/56.6%）→ CONFIRMED。
10. **matcher 空集短路（04 Flow）**：盲重建 matcher.ts L248-256 空候选/空 golden → 短路 → CONFIRMED。

## 3. 3 错误/偏差（Auditor 独立发现，禁改原产物）
### 3.1 EK-22 指纹构成轻微过度 → PARTIALLY_CONFIRMED（细节修正）
- **盲重建证据**：checkpoint-fingerprint.ts 参数只有 `provider/modelId, classifierPrompt, matcherPrompt, prKeys, candidates, goldenSets, manifestEntries`。**`renderJudgeSystemPrompt` 不在指纹参数中**。
- **产物声称**：EK-22"对 judge 模型、prompt hash（MATCHER + CLASSIFIER + renderJudgeSystemPrompt）、PR 集、candidates、golden、manifest 求指纹"。
- **差异**：renderJudgeSystemPrompt 变化（若 modelId 不变）不会使指纹变化。影响：judge 系统提示单独改动不触发重评（但 judge system 与 modelId 通常绑定，实际影响低）。
- **修正建议（Reconciliation）**：EK-22 措辞改为"provider/modelId + classifier/matcher prompt hash + 输入集（PR/candidates/golden/manifest）hash"；renderJudgeSystemPrompt 不单独进指纹。主体结论（配置/输入变化→重评）不变。

### 3.2 产物遗漏：classify 前的 manifest↔golden PR identity 四字段比对 → MISSING
- **盲重建证据**：eval.ts L415-425：classify 前 `if (manifestEntry.repo !== golden.pr.repo || manifestEntry.pr_number !== golden.pr.pr_number || manifestEntry.head !== golden.pr.head || manifestEntry.base !== golden.pr.base) throw new Error("manifest PR identity does not match golden")`——防"manifest 与 golden 错配时把 manifest 的 title/body 当作 judge 上下文"。
- **产物缺失**：02 EK-21/EK-24 与 04 Flow 均未记录此 Edge（01 的 identity 校验只写了 candidate 侧；这是 **golden/manifest 侧** 的第三处 identity 校验）。
- **意义**：manifest 是缓存数据（EK-24），golden 是权威数据——两者必须对齐才能把 manifest 的文本喂给 judge；这是一条 Authority+Data 流防误用 Edge。应补入 Reconciliation（04 Authority Flow + EK-24 links）。
- **影响**：低（不改变已产结论，但属"评测输入一致性"关键防护，遗漏不诚实）。

### 3.3 产物遗漏：RB_DELETE_GOLDEN_AFTER_LOAD 与 --ingest 互斥 → MISSING
- **盲重建证据**：eval.ts L689-691：`if (opts.ingest) throw new Error("RB_DELETE_GOLDEN_AFTER_LOAD=1 cannot be combined with --ingest")`——原因注释：ingest 要重写 golden，删盘后无法写回。
- **产物缺失**：EK-23 只写了"可删盘保密"，未记录与 ingest 的组合约束。
- **修正建议**：EK-23 补边界"deleteFromDisk 与 --ingest 互斥"（组合约束属于 EK 边界与例外）。

## 4. 遗漏（Coverage 视角，独立重搜后对比）
- §3.2 / §3.3 两处（已计入 MISSING）。
- 另确认产物已诚实自列的 5 项 coverage 缺口（sync-test-set.ts、golden-producers.ts/print-summary.ts/checkpoint-fingerprint.ts 细节、tests 断言细节、examples/parse.mjs、METHODOLOGY §3/§6）在盲重建后维持"非阻塞"判断——其中 checkpoint-fingerprint.ts 已在本轮 Auditor 独立读毕并触发 §3.1 修正，属"产物缺口 → Auditor 补证 → 修正项"的良性闭环。

## 5. 重点攻击面结论（Auditor 定向攻击）
| 攻击面 | 盲重建结论 |
|---|---|
| 单案例→Pattern | KO-01/02/03/04/06 均有 ≥2 EK 簇 + 同类对照，无单案例升维 |
| Pattern→L4 | KO-05/07/08 的三不变量/最小权限/自证其偏均为稳定关系表述，无"项目经验冒充定律" |
| 项目经验→通用 Principle | 全部 L3+ 标 `Cross-project validation pending`，无越界 |
| ADR→实现事实 | 本仓库无 ADR 文档；"设计意图"均来自代码注释与 METHODOLOGY（已按文档/实现分层标注） |
| Flow Edge 真实性 | 抽查 6 条全 ✔（§2.4/2.8/2.10 等）；补证 1 条新 Edge（§3.2） |
| bypass/override/exception/alternate/direct call/admin/fallback/legacy | 盲重建确认：upstream 回退（git.ts L139-145 / try-agent.sh）、空集短路、checkpoint 异常→empty、RB_ATTEMPT 重试——全部在产物中显式覆盖；**无隐藏捷径进入计分** |
| Epistemic 混淆 | 产物区分 Fact/Observation/Hypothesis；官方报告口径（96.6%/103.9M/A-B）三处标注非仓库可复现；无混淆 |

## 6. Benchmark Case（供 knowledge-archaeology-skill CI 复用）
- **case**：reviewbench / ARCH-2026-10-14-001
- **学习点 A（评测可信度）**：三不变量模型（可复现×独立×防操纵）——skill 的 Validation 流可对照"评测系统自身可信性审计"；
- **学习点 B（不可信输入防御）**：git argv + hex-ref + nwo + literal-pathspecs 四层——可作 skill 的 authority-analyst 反例模板；
- **学习点 C（匹配计分去重）**：covered/claimed 分离——可迁移到 skill 自身"证据匹配"设计；
- **质量基线**：EK 28 / 边 42 / 0 游离；KO 8 全带 R1-R4；Truth 12/12；Abstraction 0 DOWNGRADED；Epistemic 全绿——作为 skill v3.2 在"评测类项目"上的 Gold Record 参考。

## 7. 结论
- 独立验证判定：**10 CONFIRMED / 1 PARTIALLY_CONFIRMED / 2 MISSING / 0 CONTRADICTED / 0 DOWNGRADED**；
- 3 处偏差均为细节级（1 措辞、2 遗漏 Edge/约束），不推翻任何 KO 与 EK 主体；
- 修正项将在阶段⑥ Reconciliation 合并进 corpus 产物（不修改考古原始文件）。

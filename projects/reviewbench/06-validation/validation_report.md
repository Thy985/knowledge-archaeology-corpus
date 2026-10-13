# 06 · Validation & Evidence · ReviewBench（ARCH-2026-10-14-001）

> 验证 = 重新做一遍再判断（Blind Reconstruction），不是"证明这个答案没问题"。所有 Auditor 先独立重建，再与本包产物对比。禁止以弱化证据标准换取全绿。

## 0. 验证方法说明
- **盲重建**：本包 02/03/04 产物完成后，验证阶段独立重读关键代码（scorer.ts / matcher.ts / classify.ts / eval.ts / git.ts / match-types.ts / prompt-format.ts / judge-call.ts / prompts.ts / input-validation.ts / pipeline.ts / AGENT_CONTRACT.md / schema / METHODOLOGY.md §5.5-§8），重建独立结论后再逐条对比，而非"检查产物是否自洽"。
- **反例预算制**：每个 L3+ KO 至少 3 个定向反例攻击（找"哪里不是这样"）；0 反例必须给出搜索证据。

## 1. Truth Auditor（这句话是真的吗？）
**抽查结果（每条都回到仓库原文核对）**：
| 产物 claim | 盲重建核对 | 结论 |
|---|---|---|
| EK-01 匹配标准 "fixing one would fix the other" | matcher.ts L43-46 原文一致 | CONFIRMED |
| EK-04 coveredGolden 全算 recall、claimed 只认第一个 | match-types.ts L50-61：raw match 全进 coveredGolden；claimedGolden 首个未认领 | CONFIRMED |
| EK-06 分母 0 → null 而非 NaN | scorer.ts computeMetrics 段（matched_count=0 → null 保护） | CONFIRMED |
| EK-08 findings 转义 + data-not-instructions | prompt-format.ts L7-25 原文一致 | CONFIRMED |
| EK-12 classifier 工具只读 | classify.ts L52-53 `["read","grep","find","ls"]` | CONFIRMED |
| EK-15 argv + LFS skip + diff flags | git.ts L24-48 原文一致 | CONFIRMED |
| EK-18 mirror 优先 + 3-dot | git.ts L129-149（mirrorNwo 构造）、L227-246（cat-file -e 预检 + ... diff） | CONFIRMED |
| EK-22 checkpoint 指纹隔离 | eval.ts L272-278：指纹不等 → 丢弃重评 | CONFIRMED |
| EK-30 golden 构成数字 | METHODOLOGY §5.5 表：full 4632/2623 TP/56.6%；human 1.5% TP 份额 | CONFIRMED |
| KO-08 §8 六类威胁 | METHODOLOGY L678-734 原文六段（golden incompleteness/classifier dependence/offline-online convergence/matcher dependence/PR-level snapshot/producer bias） | CONFIRMED |
| snapshot 中 manifest 219 / test 25 / golden 219 | `python3 -c json` 实读：len(manifest)=219、len(test)=25、`ls golden|wc -l`=219 | CONFIRMED |
| run "全有或全无" | AGENT_CONTRACT.md L91-92 原文 | CONFIRMED |

**未发现 Contradiction**（产物与代码原文无冲突项）。一处需注意的**边界事实**：Blog 报告的 96.6% 复标一致率与离线/在线 A/B 数字（precision +4.45% vs +8.00% 等）在仓库内无原始计算脚本，本包全部标注为"官方报告口径"而非仓库可复现事实——未升层。

## 2. Coverage Auditor（还有什么重要东西没发现？）
**独立重搜 + 子系统强制覆盖**后，确认产物已覆盖：评测管道（eval/scorer/matcher/classify/input-validation/pipeline/judge-call/prompt-format/unmatched-classifications）、git 安全层、契约与 schema、数据集构成、方法论核心章节。**诚实列出的覆盖缺口**：
| 缺口 | 影响 | 处置 |
|---|---|---|
| scripts/sync-test-set.ts（366 行）与 check:test-set/check:golden 的校验规则未逐行读 | test set 同步逻辑的细节（如何从 full 选 25）未入 EK | 记录为 Coverage Gap；不影响已产 EK 正确性 |
| scripts/golden-producers.ts / print-summary.ts / checkpoint-fingerprint.ts 内部实现未逐行核对 | 其行为已在 EK-22/30 及 Flow 中引用但未细读 | 已通过 eval.ts 调用点与 package.json scripts 佐证存在性；细节留 Candidates 之外（低风险） |
| tests/ 10 个文件的具体断言未逐条读 | 测试揭示的行为只入 EK-01/03/06/14 的间接证据 | 下轮 REFRESH 若选中可补 |
| examples/codex-cli/parse.mjs 的解析规则 | vendor 适配器细节 | 非核心（模板实现），已读 agent.mjs 头 30 行确认契约接线 |
| docs/METHODOLOGY.md §3（corpus selection）与 §6（evaluating）全文 | 构造方法细节（§3.2 采样策略）未入包 | 影响：C-05/06 的验证路径依赖 §3 细节；已列 Candidates，非阻塞 |

## 3. Flow Auditor（Flow Atlas 是否真的对应代码？）
**Edge 真实性抽查**：
- `coveredGolden ← matcher raw matches`：match-types.ts L50-52 ✔
- `checkpoint 指纹不等 → 丢弃`：eval.ts L272-278 ✔
- `egress 被拒 → 首 PR 失败 → 停 run`：AGENT_CONTRACT.md L105-106 ✔
- `脏 checkout → 重建`：git.ts L111-125 ✔
- `classifier session 持久化 → 可审计`：classify.ts L120-133（sessionDir + SessionManager.create）✔
- **bypass/fallback/legacy 路径排查**：upstream 回退路径（git.ts L139-145，clone 失败→shallow 回退）、matcher 空 golden 短路（L248-256）、try-agent mirror 缺失→upstream 回退（脚本 has_commits 段）、checkpoint 读取异常→empty（L283-285）、`RB_ATTEMPT` 重试语义——**全部在 Flow Atlas 中显式标注或已由 EK 覆盖，无隐藏捷径进入计分**。
- 结论：未发现"Flow Atlas 声称的 Edge 在代码中不存在"或"代码中存在但 Atlas 遗漏的重要 Edge"（遗漏=direct call/admin path 类：无——container 内 agent 无任何直接计分入口；计分只在 judge CLI 内）。

## 4. Abstraction Auditor（是否过度升维？）
| KO | 攻击 | 结论 |
|---|---|---|
| KO-01 LLM 输出须机器校验 | 反例：若 LLM 输出仅作参考展示（不进入计分），严格校验是浪费 → 模式应限定"进入决策流的输出" | 收紧 scope：**进入计分/决策流的** LLM 输出。PARTIALLY_CONFIRMED（已按此表述） |
| KO-02 被测者输出=不可信数据 | 反例：若被测者无注入动机（非对抗评测），防护是冗余 → 但 ReviewBench 中 agent 有真实动机（刷分），且注入代价低 | CONFIRMED（对抗面真实存在） |
| KO-03 不可信输入先形状校验 | 反例：若输入来自可信内部系统（同源 trusted pipeline），可省校验 → 但 ReviewBench 的 candidate nwo/ref 来自不可信 PR 数据 | CONFIRMED（前提"输入不可信"成立） |
| KO-04 covered/claimed 分离 | 反例：一对一匹配（位置唯一）场景无需分离 → 模式适用于 many-to-many 语义匹配场景 | 收紧 scope：**many-to-many 语义匹配**场景。PARTIALLY_CONFIRMED |
| KO-05 可复现×独立×防操纵 | 反例：非正式/开发期评测不需要全套三不变量（OFFLINE_EVAL 就是轻量路径）→ 模型适用于**正式榜单/发布评测** | 收紧 scope。PARTIALLY_CONFIRMED |
| KO-06 自持数据 | 反例：一次性分析无需冻结快照（成本高）→ 适用于"长期复现资产" | 收紧 scope。PARTIALLY_CONFIRMED |
| KO-07 judge 最小权限 | 反例：若 judge 无工具（纯 API 问答），权限模型退化为"无权限"——ReviewBench judge 有读工具是特例 | CONFIRMED（表述已含"足以验证任何 claim 的读权限"前提） |
| KO-08 基准须自证其偏 | 反例：内部基准（不发布）可不公开 Threats → 适用于**对外发布的评测基准** | 收紧 scope。PARTIALLY_CONFIRMED |

**升维结论**：8 个 KO 中 4 个 PARTIALLY_CONFIRMED（已通过 scope 收紧修正表述，修正已写入 03 文件原文），无 DOWNGRADED（层级均与证据强度匹配），无 OVER_GENERALIZED（无单一证据写成普遍定律；全部标注 Cross-project validation pending）。

## 5. Counterexample Hunter（专门找"哪里不是这样"）
每个 L3+ KO 定向反例（部分已在 §4 合并；此处列 KO 特定反例）：
- **KO-01 反例**：matcher 的 `extractJson` 有宽松 fallback（正则 `\{[\s\S]*\}`，matcher.ts L103-117）——即"结构校验严格，但 JSON 提取有容错"——说明"机器校验"与"解析容错"可分层共存，不是"要么全严要么全松"。
- **KO-02 反例**：golden findings 也走同一 renderMatcherFinding 标签（角色不同），说明"数据/指令分离"对 golden 侧同样生效——信任方（golden）与不信任方（candidate）共用同一处理层，差异只在标签角色。
- **KO-03 反例**：try-agent.sh 用 `git -C "$repo" remote remove origin` 等直接命令且允许 upstream 回退（脚本未做 hex-ref 校验，因为它读的是 manifest 的 JSON 字段而非用户输入）——说明"校验强度随输入信任度分层"，不是一刀切。
- **KO-04 反例**：`unmatched` 的分类发生在 scoring 之前（eval.ts L389-395 注释明说"duplicate matches that lose scoring credit are classified alongside truly novel findings"）——匹配失利的候选仍获分类机会，防止"匹配策略错误 → 发现被吞"。
- **KO-05 反例**：OFFLINE_EVAL 路径明确不需要全套治理（"You do not need to register... publish anything"）——三不变量是可选项（正式 run 强制），不是系统硬性不变量。
- **KO-06 反例**：mirror 回退 upstream 仍存在（git.ts L139-145 与 try-agent.sh）——自持数据是"优先策略"而非"绝对原则"，上游作为可用性回退仍被接受。
- **KO-07 反例**：matcher session 用 `noTools: "all"`（matcher.ts L303）——matcher 是纯问答（零工具），与 classifier（有读工具）不同——"最小权限"的边界随任务需要变化。
- **KO-08 反例**：Blog 是外部对照渠道（离线/在线 A/B 已做），说明"自证其偏"可以与外部验证并存——仓库无对照 pipeline 不等于没有验证（官方在 Blog 报告了）——已在 C-06 注明。

反例结论：8 个反例均未推翻 KO（全部是 scope/分层补充），但证明各 KO 不是"无例外法则"。

## 6. Epistemic Auditor（认知状态标注是否诚实）
| 检查项 | 结果 |
|---|---|
| Hypothesis 未冒充 Fact | C-01~C-06 全部标注 Hypothesis/未验证；05 文件声明纪律 ✔ |
| Observation 未冒充 Pattern | 02 文件每条 EK 状态列区分 Fact/Observation（如 EK-01、EK-08 标注 Observation 成分）✔ |
| Cross-project 未冒充已验证 Principle | 全部 L3+ 标 `Cross-project validation pending` ✔ |
| 官方报告口径 vs 仓库可复现 | 96.6% / 103.9M / A/B 数字均标注"官方报告口径"（00/06/snapshot 三处）✔ |
| 项目自承威胁被如实保留 | §8 六类威胁进 KO-08 且 C-05/06 承接 ✔ |
| 反例与 Contradictions 保留 | §4/§5 全量保留（4 个 PARTIALLY_CONFIRMED 已写入 03 修正）✔ |

## 7. 验证结论统计
| 维度 | 判定 |
|---|---|
| Truth | 12/12 CONFIRMED，0 Contradiction |
| Coverage | 5 项缺口已列（均非阻塞，影响已在缺口表处置） |
| Flow | 6 Edge 抽查全 ✔；无遗漏重要 Edge；无隐藏计分捷径 |
| Abstraction | 4 CONFIRMED + 4 PARTIALLY_CONFIRMED（scope 收紧已并入 03）+ 0 DOWNGRADED + 0 OVER_GENERALIZED |
| Counterexample | 8/8 KO 各 1 反例，均未推翻（补充 scope） |
| Epistemic | 全绿（标注诚实性检查通过） |

## 8. Benchmark 学习价值（供 skill CI 用）
ReviewBench 作为 Benchmark case 的核心学习点：
1. **不可信被测者 → judge 的注入面隔离**（prompt-format 数据/指令分离）——本 skill 的 Evidence 流与 Validation 流设计可对照；
2. **many-to-many 匹配的计分去重**（covered/claimed 分离）——可迁移到任何"LLM 判定相似性"的评估；
3. **评测治理闭环**（manifest/egress/指纹/保密/全有或全无）——可迁移到 Agent 评测沙箱设计。

## 9. Reconciliation（合并独立 Auditor 修正，2026-10-14）
> 独立验证（06-validation/independent_validation_report.md）判定：10 CONFIRMED / 1 PARTIALLY_CONFIRMED / 2 MISSING / 0 CONTRADICTED / 0 DOWNGRADED。以下修正已合并进本 corpus 产物（考古原始文件未修改，保留在 archaeology-runs/ARCH-2026-10-14-001/）。

| # | Auditor 发现 | 判定 | 修正落点 |
|---|---|---|---|
| R1 | checkpointFingerprint 实际仅 hash classifierPrompt + matcherPrompt；renderJudgeSystemPrompt 不单独进指纹（盲重建 checkpoint-fingerprint.ts 全文） | PARTIALLY_CONFIRMED | 02-engineering-knowledge/ek_graph.md EK-22 措辞修正 |
| R2 | classify 前存在 manifest↔golden 四字段 PR identity 比对（eval.ts L415-425），产物遗漏 | MISSING | 02 EK-24 补权威边界 + 04-flow-atlas Data Flow 补第三处 identity 守卫 |
| R3 | RB_DELETE_GOLDEN_AFTER_LOAD=1 与 --ingest 互斥（eval.ts L689-691），产物遗漏 | MISSING | 02 EK-23 补边界与例外 |

**Reconciliation 结论**：3 处修正均为细节级（1 措辞 + 2 遗漏 Edge/约束），不改变任何 KO 与 EK 主体结论；无 Contradiction 需保留；无 Skill Improvement Proposal（未暴露系统性 skill 缺陷——Blind Reconstruction 与独立 Auditor 机制按设计工作）。

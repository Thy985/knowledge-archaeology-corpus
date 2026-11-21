# 02 · Engineering Knowledge · ReviewBench（EK Graph）

> L1/L2 工程知识（宽底座 · 推理原材料）。每条 EK 必须声明 links（六类边）；孤立 EK 不允许存在。证据列 = 文件:行（仓库 commit `c17b05b`）。

## 0. EK Graph 概览
- 节点：28 条 EK
- 边数：42（平均出边 1.5）
- 游离 EK：0（全部 ≥1 边）
- 聚合规则覆盖率：100%（见 03_knowledge-layer.md 的 KO 聚合矩阵）

## A. 评测管道核心机制（EK-01 ~ EK-07）

### EK-01 · 语义匹配的判定标准是"修一个就修另一个"
匹配器把两份 findings 判为同源底层问题的标准：`fixing one would fix (or substantially address) the other`；同一位置但不同 issue **不**匹配。这比位置/文本相似更强——它以"修复动作的等价性"定义匹配。
- 证据: scripts/eval/matcher.ts L43-46
- 状态: Fact（prompt 原文）+ Observation（由 prompt 语义导出）
- links: mechanism→EK-03（JSON 契约同一判定标准落地）; contrast↔EK-06（评分层不再重新判同源，直接用匹配结果）; subsystem→EK-02

### EK-02 · 文件作用域 + 块交叉的 LLM 匹配策略
匹配先按 file 分组（只比较同文件 findings），文件内按 CHUNK_SIZE=10 分块，候选块 × golden 块交叉比较；单 session 复用（model resolution 一次）。空候选/空 golden 短路。
- 证据: scripts/eval/matcher.ts L258-372（L336-346 交叉循环；L176-181 短路）
- 状态: Fact
- links: mechanism→EK-13（单 session 复用模式）；subsystem→EK-01; causal→EK-04（匹配结果进入 resolveForScoring）

### EK-03 · matcher 输出 JSON 严格校验（索引合法/无重复/全覆盖）
parseMatcherResponse 要求：index 是合法整数且在范围内、candidateFindings 无重复、matchedGoldenIndices 每项合法、seen.size 必须等于 candidateCount（一个都不能少）。失败即 throw，不静默容错。
- 证据: scripts/eval/matcher.ts L119-164
- 状态: Fact
- links: mechanism→EK-12（"LLM 输出必须机器强校验"同一模式，classify 也如此）；constraint→EK-14（超时中止下的响应完整性）

### EK-04 · 匹配→评分的解耦：covered（recall）与 claimed（precision）分离
resolveForScoring：**所有** raw match 的 golden 都进 coveredGolden（recall 分母），但 precision 只认"每个 golden 第一个未认领的候选"（claimedGolden 按 golden 顺序抢占，matchedToGolden 一对一）；一个候选匹配多个 golden 只算一个 scored match——**防"一个发现认领多个金标刷 precision"**。未匹配进 unmatched。
- 证据: scripts/lib/match-types.ts L30-68
- 状态: Fact + Observation（由算法行为导出）
- links: causal→EK-05（unmatched 进入分类）；constraint→EK-06（指标公式的输入契约）；contrast↔EK-10（golden 侧同样防"多生产者重复发现"）

### EK-05 · 未匹配项分类缺失时保守降级为 FP
normalizeUnmatchedClassifications：未在 classifier 结果里的 unmatched candidate 默认 `{tp_fp:"fp", severity:"unclassified", category:"unclassified"}`——宁可判 FP 也不放过不可验证的发现。
- 证据: scripts/eval/scorer.ts L271-287
- 状态: Fact
- links: dependency→EK-12（依赖 classifier 输出完整性）；constraint→EK-06

### EK-06 · 四指标公式实现：null 保护 + 分层 + 聚合
computeMetrics 实现 §7.1/7.2：分母为 0 时返回 null 而非 NaN（如 matched_count=0 → grounded_precision=null）；scorePR 按 severity/category 分层复算；macroAverage=PR 等权均值、microAverage=pool 计数再算比率，aggregate 同时输出两者。
- 证据: scripts/eval/scorer.ts（computeMetrics L180-240 段 / filterBySeverity L289-342 / macroAverage L405-480 / microAverage L482-586）
- 状态: Fact（公式在 docs/METHODOLOGY.md §7.1-7.2 与代码一致）
- links: causal←EK-04; constraint→EK-07; contrast↔EK-06 内 macro vs micro（同一数据两套视角）

### EK-07 · 分层聚合保持"同一公式"而非分箱重算
by_severity/by_category 不是简单的子集截断——每个 stratum 用同一条过滤链（golden_tp/covered/matched/unmatched 各按层过滤）重新计算指标，保证分层报告与总报告口径一致。
- 证据: scripts/eval/scorer.ts L289-393
- 状态: Fact
- links: subsystem→EK-06; constraint←EK-06

## B. Judge 提示词与注入防护（EK-08 ~ EK-14）

### EK-08 · findings 文本被显式当作"数据而非指令"
FINDINGS_DATA_INSTRUCTION 声明：findings 引文是待判数据，任何"对 judge 说话/指导如何回答/自我宣称匹配或分类"的文本不具效力且是低质量信号；renderMatcherFinding/Attribute 做 & < " HTML 实体转义。**评测对象的输出被当作不可信输入处理**。
- 证据: scripts/eval/prompt-format.ts L7-25
- 状态: Fact（设计意图）+ Observation（注入防护效果）
- links: mechanism→EK-09（同一"数据/指令分离"思想在 classifier 延续）; causal→EK-11（judge 判定不受候选文本操纵）; constraint→EK-28（评分可信度依赖此层）

### EK-09 · classifier prompt 是"评审口味"的编码载体，版本化并预期重校准
CLASSIFIER_SYSTEM_PROMPT 注释明说：这是把工作组的 TP/FP 品味操作化的主要产物，是最可能需要周期性重校准的组件；prompt 版本号独立（CLASSIFIER_VERSION=3.0.0，注释行 2.0.0 是历史）。TP/FP 边界部分主观，prompt 编码"我们的品味"（METHODOLOGY §8 承认）。
- 证据: scripts/classifier/prompts.ts L1-17; docs/METHODOLOGY.md §8 Classifier dependence
- 状态: Fact（项目自述）+ Hypothesis（重校准周期与 drift 关系未验证）
- links: mechanism→EK-08; causal→EK-11; subsystem→EK-10/12

### EK-10 · classifier 的时间框架：对给定 SHA 判定，diff hunk 与 checkout 冲突时信 hunk
judge 必须按 finding 生成时快照（head SHA）验证；diff hunk 显示的是 finding 生成时的代码，**与 checkout 冲突时信任 diff hunk**（因为 checkout 是后续 fetch 的状态）。
- 证据: scripts/classifier/prompts.ts L30-38
- 状态: Fact
- links: constraint→EK-17（checkout 可能滞后，正是 hunk 优先的原因）; subsystem→EK-09

### EK-11 · TP/FP 判定规则：TP 三条件 + FP 五类 + generic/security 专用测试
TP 须 True∧Relevant∧Non-trivial；FP 任一即触发：Incorrect/Unverifiable/Generic(可一字不改复制到别的项目)/Sub-threshold/Harmful。Generic 测试是"copy-paste 到不同项目一个字不改还能成立"。Security 额外加严：必须演示**具体可利用路径**（数据流到危险 sink/硬编码密钥/鉴权绕过），action-pinning 等模板警报即使指名变量也是 FP。Scope 不参与 TP/FP（pre-existing 也可 TP），单独用 Scope 轴记录。
- 证据: scripts/classifier/prompts.ts L43-77
- 状态: Fact（prompt 规则）+ Observation（这是基准"品味"的核心）
- links: causal←EK-09; mechanism→EK-08; constraint→EK-06（分类标签直接进指标）; contrast↔EK-11 内 security 加严 vs 常规判定

### EK-12 · judge 只读验证：classifier 工具集限定 read/grep/find/ls（无 shell）
CLASSIFIER_TOOLS = ["read","grep","find","ls"]——judge 可以读文件、搜代码、验证 claim，但**不能改任何东西**；且"不要猜，验证"是 prompt 明文指令。
- 证据: scripts/classifier/classify.ts L52-53; prompts.ts L28-29
- 状态: Fact
- links: mechanism→EK-08（judge 的权限最小化）; dependency←EK-05; subsystem→EK-13

### EK-13 · 每 PR+SHA 一个共享 session（上下文复用 + 审计）
classifyPR 为整个 PR 开一个 pi session：先发 PR 上下文（title/body/diff≤100k/SHA），再逐 finding 分类；session 持久化到磁盘（sessionDir，可 `pi --resume/--export` 审计工具调用）；thinkingLevel=medium；retry maxRetries=3。
- 证据: scripts/classifier/classify.ts L93-150
- 状态: Fact
- links: mechanism→EK-02（matcher 同样单 session 复用）; causal→EK-14（session 受超时预算约束）; subsystem→EK-12

### EK-14 · judge 调用超时预算：10min 基础 + 2min/finding，AbortController 硬中止
JUDGE_CALL_TIMEOUT_MS 默认 10min（环境变量可覆盖）；classifier 每 finding 额外 +2min；withTimeout 用 AbortController + setTimeout 硬中止，"模型永不返回不能卡死 run"；超时抛 JudgeTimeout（命名 what，如 "owner/repo#12: classifier gpt-5"）。
- 证据: scripts/eval/judge-call.ts L8-44
- 状态: Fact
- links: constraint→EK-03（超时后响应可能不完整，靠严格校验兜底）; constraint→EK-13

## C. Git 快照与不可信输入防御（EK-15 ~ EK-19）

### EK-15 · git 以 argv 数组执行 + LFS smudge 跳过 + diff 确定性 flags
execFileSync("git", args)（永不经 shell，untrusted ref/path 不能变成 shell 语法）；GIT_LFS_SKIP_SMUDGE=1（mirror 只带 LFS pointer，smudge 会失败；pointer 就是评分所需）；DIFF_FLAGS = --no-ext-diff/--no-textconv/--no-color（diff 不随环境漂移）。
- 证据: scripts/extraction/git.ts L24-48
- 状态: Fact
- links: mechanism→EK-16; subsystem→EK-17/18

### EK-16 · 输入到达文件系统前先校验：assertHexRef + parseNwo
assertHexRef：只接受 4-64 位 hex（拒绝 "-" 开头 → 防 git 选项注入）；parseNwo：严格 `owner/repo`，拒绝 "." / ".." 段（防路径穿越逃逸 cache 目录）；注释明说 GitHub 仓库名可以标点开头（.github），所以只拒绝字面穿越段。checkoutRepo 在任何 fs 操作（含 rmSync）前完成两者校验。
- 证据: scripts/extraction/git.ts L53-72, L93-100
- 状态: Fact
- links: constraint→EK-17; mechanism→EK-15; causal→EK-18

### EK-17 · checkout 缓存复用仅当 HEAD==sha 且工作树干净；脏则重建
缓存目录复用条件：rev-parse HEAD==sha AND status --porcelain 为空；否则 rmSync 重建。原因（注释）：中断的 checkout、失败 LFS smudge、或 **classifier 自己的工具（cwd 就在 checkout 里）** 都可能留下 HEAD 对但树脏/不完整，会静默喂错误文件上下文给后续评分。复用前还要 clean -ffdx 清 ignored/untracked。
- 证据: scripts/extraction/git.ts L103-126
- 状态: Fact（含明确 root-cause 注释）
- links: causal←EK-10; subsystem→EK-15; contrast↔EK-16（校验在前，重建兜底）

### EK-18 · mirror 优先于 upstream：冻结快照保证评分永不依赖会腐烂的上游
checkoutRepo 从 review-bench mirror（`review-bench/<owner>_<repo>`，MIRROR_ORG 可覆盖）clone 而非 upstream——corpus PR 的上游仓库可能被删/force-push（rot），mirror 是"我们保留的不可变快照"。getFullDiff 用 3-dot（merge-base..head）匹配 GitHub PR diff（2-dot 会在 base 非 head 祖先时混入 base 专属变更）；两 commit 必须本地存在否则 throw（不静默产出错误/空 diff）。try-agent.sh 同策略（mirror 为 source of truth，upstream 回退）。
- 证据: scripts/extraction/git.ts L93-99, 129-149, 227-246; scripts/try-agent.sh（has_commits/mirror fetch 段）
- 状态: Fact
- links: causal→EK-19; constraint←EK-16; mechanism→EK-23（"自持数据不依赖外部"模式）

### EK-19 · 行区域 unchanged 检测 + 行号重映射
checkRegionUnchanged：解析 diff hunks，hunk 与目标行区间重叠→已变化；否则累加 hunk 偏移把 old 行号映射到 new 行号。extractCodeRegion 取行区间 ±3 行上下文（"> " 标目标行）。
- 证据: scripts/extraction/git.ts L254-330
- 状态: Fact
- links: subsystem→EK-15; causal←EK-18

## D. 评测输入完整性与保密（EK-20 ~ EK-25）

### EK-20 · strict 输入门：malformed/invalid/missing-golden/空 overlap 全部拒绝
strictInputProblems：任何 malformed JSON / invalid shape / 无 golden 的候选 PR / （allowEmpty 未开时）0 有效 PR 或 0 findings 或 0 overlap → 列出问题（带样例路径）。strict 默认开。
- 证据: scripts/eval/input-validation.ts L193-235
- 状态: Fact
- links: constraint→EK-21; causal→EK-06（坏输入不进指标）

### EK-21 · candidate 加载时的 PR identity 校验 + key 冲突检测
loadCandidateFindings：JSON.parse 失败→malformed；shape 校验失败→invalid；同 prKey 文件再次出现时校验 repo/pr_number/base/head 四字段全等，不等→拒（"truncated PR key collides"）；按文件递归加载。validateExtractionOutput 校验 findings 字段类型与 usage 非负有限数。
- 证据: scripts/eval/input-validation.ts L64-142, L237-275
- 状态: Fact
- links: mechanism→EK-20; constraint→EK-22

### EK-22 · checkpoint 指纹隔离：judge/prompt/输入任一变化→丢弃重评
checkpointFingerprint 对 judge 模型标识（provider/modelId）、prompt hash（MATCHER_SYSTEM_PROMPT + CLASSIFIER_SYSTEM_PROMPT，SHA-256 + canonicalize）、PR 集、candidates、golden、manifest 求指纹（`provider/modelId | hash(classifier) | hash(matcher) | hash({pr_keys,candidates,golden_sets,manifest_entries})`）；loadCheckpoint 仅当指纹完全匹配才复用，否则 warn + 从零重评——"不能把别的 judge 的分数挂在本 run 的模型标签下"。[Reconciliation 修正] renderJudgeSystemPrompt 不单独进指纹（独立 Auditor 盲重建 checkpoint-fingerprint.ts 确认其参数仅含 classifierPrompt/matcherPrompt 两个 prompt hash；judge 侧隔离由 provider/modelId 承担）。
- 证据: scripts/eval/eval.ts L248-309; checkpoint-fingerprint.ts（全文）
- 状态: Fact
- links: causal→EK-24; constraint←EK-23; mechanism→EK-28（可复现/不可混淆）

### EK-23 · golden 驻内存 + 可删盘：防后续 job 读到标签
GoldenStore 构造时把 golden 全部读进内存（raw+parsed 双份），keys/raw 供指纹哈希；`deleteFromDisk()` 在 RB_DELETE_GOLDEN_AFTER_LOAD=1 时删盘而 store 继续服务——防止同一 runner 上后续 job 从磁盘读到 golden 标签。[Reconciliation 修正] 边界：`RB_DELETE_GOLDEN_AFTER_LOAD=1` 与 `--ingest` 互斥（eval.ts L689-691 显式 throw——ingest 需重写 golden，删盘后无法写回；独立 Auditor 盲重建确认）。
- 证据: scripts/eval/eval.ts L189-236, L689-691
- 状态: Fact
- links: mechanism→EK-18（自持数据）；constraint→EK-22

### EK-24 · manifest 缓存：评分读本地 manifest 而非 upstream GitHub API
getManifestEntry 从 manifestPath 一次性加载进 Map（重复 key 直接 throw），评分时 PR 上下文（title/body）来自 manifest 而非 upstream API——"永不让评分依赖上游仓库还存在"。[Reconciliation 修正] 补充权威边界：classify 喂给 judge 前，manifest entry 与 golden 做四字段（repo/pr_number/head/base）PR identity 比对，不等则 throw "manifest PR identity does not match golden"（eval.ts L415-425）——manifest 是缓存数据、golden 是权威数据，两者必须对齐才能把 manifest 文本当 judge 上下文（独立 Auditor 盲重建确认）。
- 证据: scripts/eval/eval.ts L336-362, L415-425
- 状态: Fact
- links: mechanism→EK-18; causal→EK-22（manifest 进指纹）; subsystem→EK-20

### EK-25 · dedup/merge：novel findings 可回灌 golden（ingest）
dedup：新 findings 与 golden 用 matcher 判重复（有 match→duplicate，无→novel）；merge：novel 分类（缺失时占位 tp/medium/unclassified）追加进 golden；eval --ingest 把 novel TPs 写回 golden/（writeGoldenSet），随后 removeCheckpoint 强制重评。classify-adapter 复用同一 classifyPR。
- 证据: scripts/golden/pipeline.ts L84-209; eval.ts L238-242, L304-309
- 状态: Fact
- links: causal→EK-23（写回后 golden 变化）；subsystem→EK-13; contrast↔EK-20（ingest 主动改数据 vs strict 校验只读）

## E. 运行契约与治理（EK-26 ~ EK-28）

### EK-26 · run 全有或全无 + 契约输出校验
单 PR 失败（非 0 退出/缺文件/坏 JSON/head 或 PR 号不符）→ retry → 整 run 失败，不发布部分分数（AGENT_CONTRACT.md L83-92）。try-agent.sh 实现同款校验：pr.head==RB_HEAD、pr.pr_number==RB_PR_NUMBER、findings 数组字段合法、file 不以 ./ 或 / 开头。
- 证据: AGENT_CONTRACT.md L83-92; scripts/try-agent.sh（problem jq 段）
- 状态: Fact
- links: constraint→EK-01（输出质量直接影响匹配）；mechanism→EK-29（校验驱动）

### EK-27 · egress 网络白名单 + RB_MODEL_BASE_URL 单一来源
容器网络被 forward proxy 管控：只放行 manifest egress 主机（443）；model_api_url 主机自动加入；RB_MODEL_BASE_URL 是 agent 读端点的唯一位置——"端点与白名单出自同一处"。被拒主机 → 首 PR 失败即停 run 并点名。
- 证据: AGENT_CONTRACT.md L102-112; schema/agent-manifest.schema.json model_api_url/egress
- 状态: Fact
- links: mechanism→EK-28; constraint→EK-26（网络失败计入 run 失败）

### EK-28 · image digest-pinned：可复现 + 字节不可变
manifest image 字段必须 `ghcr.io/<owner>/<name>@sha256:<64hex>`（schema pattern 强制）：按 digest 引用使 run 可复现、"bytes cannot change under us"。image_visibility 控制 runner 匿名拉取还是凭 GHCR_PULL_TOKEN。
- 证据: schema/agent-manifest.schema.json image/image_visibility 字段
- 状态: Fact
- links: constraint→EK-27; mechanism→EK-22; causal→EK-26

### EK-29 · label 有效性检查：配置值必须出现在 agent 输出中
RB_CONFIG_* 键规范化（upper-case + 非字母数字→_）；run 名会点名"其值从未出现在 agent 输出的 label"——因为"你的镜像不读的 label 没有效果"（防 manifest 声明配置但 agent 未应用，造成榜单配置幻觉）。
- 证据: AGENT_CONTRACT.md L38
- 状态: Fact（设计意图）+ Observation
- links: subsystem→EK-27; constraint→EK-26

### EK-30 · golden 数据的 producer 混合结构（数据事实）
full set 4632 findings / 2623 TP（56.6% TP rate）；按 producer：llm_review:claude-sonnet-4.6 46.0% TP 份额、ccr:gpt-5.5[medium] 36.5%、gemini-2.5-pro 5.7%（21.1% TP rate）、human_review 1.5%（20.4% TP rate）、semgrep 0.3%。**human review 在 golden 中占比极小**——golden 主要由 LLM 生产者构成。
- 证据: docs/METHODOLOGY.md §5.5 表（仓库原文）
- 状态: Fact（仓库文档数字）+ Observation（对基准构造含义）
- links: causal→KO-08（producer 混合 → 基准偏差威胁）; contrast↔EK-06（数据分布影响指标解读）; subsystem→EK-24

## EK Graph 边统计
- mechanism 边：EK-01↔03、02↔13、08↔09、08↔12、15↔16、18↔23、18↔24、22↔28、26↔29（9）
- subsystem 边：01→02、06→07、09→10/12、12→13、15→17/18、19→15、20→21、24→20、27→29、30→24（10）
- causal 边：02→04→05→06、09→11→06、13→14、16→18→19、16→17、22→24、25→23、22→28→26、28→26、30→08（若干）
- constraint 边：03←14、04→06、06→07、10→17、16→17、20→21、22←23、26→01、27→26、28→27（若干）
- dependency 边：05→12、27→28
- contrast 边：01↔06、04↔10、11↔security 加严、17↔16、25↔20

**反退化检查**：无孤立 EK（全部 ≥1 边）；聚合理由均非"同子系统=知识"（KO 聚合见 03 文件）。

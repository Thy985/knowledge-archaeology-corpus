# Independent Validation Report — ARCH-2026-09-20-001（dynamo）

> 独立 Auditor 盲重建：不把考古结果当事实来源，独立重读仓库（commit `6562e7d`）建立 Independent Findings 后对比。

## 判定统计
| 判定 | 数 | 对象 |
|---|---|---|
| CONFIRMED | 30 | F-01~35 主体 / EK-01~20 主体 / KO-01~09 主体 / 170M ops/s（flash-indexer.mdx:289,315）/ AGENTS.md 独裁（agents/hypothesis-challenger/CLAUDE.md=`@AGENTS.md`）/ nvext.agent_hints（nvidia-request-extensions-nvext.md:136）/ cache_control 拒绝（blog 双处） |
| PARTIALLY_CONFIRMED | 3 | EK-03 ops 数字=文档声称非本地实测；EK-07 engine 归一化证据仅 blog 无代码路径；KO-07 实现状态（KVBM toward 4-tier 未完成） |
| MISSING | 4 | Guardrails 调度安全不变量 / DC KV Relay（多 DC KV 路由）/ salt=prefix-cache isolation key / recovery 模块（目录） |
| OVER_GENERALIZED | 1 | KO-06 三项目"分层互补"归纳偏宽 |
| DOWNGRADED | 0 | — |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 1 | C-07 170M ops/s 可复现性（bench 环境依赖） |

## Independent Findings（盲重建发现）
- **ID-1**（修正考古检索错误）：`lib.rs:20` 有 `pub mod recovery;`，路径为 `lib/kv-router/src/recovery/{cursor.rs, mod.rs}`（目录非单文件）。考古首次 grep 误判"不存在"，实际为恢复游标模块。
- **ID-2**（重大 MISSING）：scheduling/AGENTS.md Guardrails 章节含完整调度安全不变量：①`admit_one` 是必需准入路径（projected load→select→receiver closed 则跳过容量预留→预留后响应；failed delivery 必须释放容量；"Do not bypass this for normal scheduling"）②RequestGuard 以 `(request_id, attempt_id)` 围栏调度变更；cleanup 超出 admit 尝试必须带 AttemptId ③不得删除/弱化 `admission_gate` ④potential-load 投影必须走 `ActiveSequencesMultiWorker::potential_blocks_and_tokens_at` + `SchedulingRequest::prefill_token_deltas()`，禁止调度层直接扫 per-worker ActiveSequences ⑤`SchedulingRequest` 是 effective cached tokens/overlap/allowance 唯一源 ⑥WSPT 必须 cache-aware（pinned 用 pinned cached tokens；unpinned 用 best allowed worker），禁止静默回退 raw ISL ⑦pinned/allowed 约束 selection 前验证。考古仅记"有 Guardrails 章节"——实际是不可绕过性安全契约。
- **ID-3**（CONFIRMED）：170M ops/s 在 flash-indexer.mdx:289（"42x faster than Radix Tree v0.1.0 的 4M ops/s；440x vs naive 385K ops/s"，mooncake_bench 实测 trace）与 :315（"sustained 170M"）双处；description 同。考古数字准确。
- **ID-4**（MISSING）：salt.rs 定义 salt 为 **prefix-cache isolation key**——应共享前缀的请求必须产生相同 salt hash，不应共享的必须不同；`(salt=None,lora=None)→CHAIN_XXH3_SEED`；multimodal 不进 salt（折叠进 per-block hashing 保图像前前缀共享）。考古仅记"kv-hashing 含 salt.rs"。
- **ID-5**（CONFIRMED）：session_prefix_index.rs 含 SessionPrefixIndexer / LogicalNode（block_hash/parent/frontier_refs/child_count）——会话级前缀索引存在，考古未展开但未遗漏主体。
- **ID-6**（重大 MISSING）：`indexer/cuckoo/` = **Cuckoo-filter KV Indexer**——DC KV Relay 架构：`DcCkfState`（单 owner 可变 producer）+ `GlobalCkfIndexer`（并发转置 consumer，16 DC pool lanes，lossy CKF projection）；`PoolId=(IndexerDomainId, DcId)`。
- **ID-9**（重大 MISSING）：multi-dc-kv-routing.md（Experimental）——endpoint-local KV pools + serving topology + universal publisher（WAN Protobuf/gRPC）；CKF 投影避免全量事件流跨 WAN 复制；`PoolId=(identity_version, IndexerDomainId, DcId)` 跨重启稳定，colliding pools **fenced 不 merge**；base model + LoRA 共享物理 CKF 用 distinct salt 分离；"pool presence alone never implies readiness"。
- **ID-8**（CONFIRMED）：agents/hypothesis-challenger/CLAUDE.md 内容精确为 `@AGENTS.md`——AGENTS.md 独裁治理实现证据。

## 原考古最重要的 3 个成功
1. **KO-04（Agent 上下文跨 API 边界）**：nvext.agent_hints 的提炼准确抓住项目灵魂——"harness 知识 vs 基础设施可见性"gap 是 agentic inference 的核心优化面；证据链完整（blog + nvext 文档双源）。
2. **KO-02（记忆面与执行面解耦）**：把 Dynamo 的 KV 全局索引 vs worker 选择，与 hermes 冻结快照、letta 记忆文件树正确对照为跨项目认知模型——且诚实标记 cross-project pending。
3. **EK-04（ConcurrentRadixTreeCompressed）**：radix 压缩 + per-worker cutoffs + sticky internal nodes + split-no-merge 的精确提炼，直接可复用于任何 KV 前缀索引设计。

## 原考古最重要的 3 个错误
1. **Guardrails 章节被压缩为占位（ID-2）**：调度安全不变量（admit_one 不可绕过、RequestGuard 围栏、WSPT cache-aware 禁回退 raw ISL、admission_gate 保护）是"不可绕过性"工程知识，考古只写"有 Guardrails 章节"——覆盖失败，影响 Critical Coverage。
2. **多 DC KV 路由完全遗漏（ID-6/9）**：Cuckoo Indexer + DC KV Relay 是"KV 记忆跨数据中心"扩展面，含 fenced-not-merged 等关键设计——考古焦点放在单 DC 路由，把 Experimental 标记当成了排除理由。
3. **salt 隔离语义被扁平化（ID-4）**：salt 是 prefix-cache isolation key（共享/隔离判据）而非普通哈希盐——考古只记文件存在，丢失了隔离契约。

## 是否存在关键遗漏
是（4 项 MISSING 均为内容遗漏，非事实错误）：Guardrails 安全不变量、DC KV Relay、salt 隔离、recovery 模块。均已在 Reconciliation 补入。

## 是否存在错误升维
否。KO-02/KO-04 的 L4 均有跨项目支撑且标 pending；无单案例→Pattern 错升。KO-06 归纳偏宽（三项目"互补"方式差异大）→ OVER_GENERALIZED，Reconciliation 收紧 scope。

## 是否存在事实错误
否。盲重建未发现考古写错的事实（170M ops/s、NoEndpoints、cache_control 拒绝、AGENTS.md 独裁均复核）。

## 是否存在 Flow 错误
否。7 类流关键 Edge 全部复核可回溯；Guardrails 补入后 Authority/Policy Flow 需增补"admit_one 不可绕过"边。

## 新 Benchmark / Regression Case
- **B-1（调度安全）**：admit_one 绕过检测——构造绕过准入路径直接调 selection 的测试，应被 guardrail 拒绝（回归用例：failed stream 必须先释放 booking）。
- **B-2（salt 隔离）**：同 salt→同 block hash、异 salt→异 hash；multimodal 图像位置不破坏图像前前缀共享（单元级，salt.rs 契约）。
- **B-3（WSPT cache-aware）**：pinned/unpinned 请求的 WSPT 必须用 effective cached tokens，禁止回退 raw ISL（策略级断言）。
- **B-4（DC Relay）**：PoolId 跨重启稳定 + colliding pools fenced-not-merged（集成级，Experimental 标记需单独 gate）。

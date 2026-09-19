# Project Layer — dynamo（ARCH-2026-09-20-001）

> 本层 = L0 项目事实。所有条目可回溯到 commit `6562e7d`（2026-09-19）。

## F-01 定位与许可证
ai-dynamo/dynamo v1.6.0，Apache-2.0，NVIDIA（2024-2026）——"The open-source, datacenter-scale inference stack"，是推理引擎（SGLang/TensorRT-LLM/vLLM）**之上**的编排层，不替代引擎。README 明示："If you're running a single model on a single GPU, your inference engine alone is probably sufficient."（来源：README.md:30-44, Cargo.toml）

## F-02 技术栈
双栈：Rust 性能核心（Cargo workspace 20+ crates，1,285 .rs 文件）+ Python 扩展层（`ai-dynamo` wheel 1.6.0，PyO3/maturin 绑定，1,627 .py 文件）+ Kubernetes 部署层（deploy/）。总 5,888 文件。（来源：Cargo.toml, pyproject.toml, AGENTS.md:20-24）

## F-03 核心工作区结构
`lib/` 含：kv-router / kv-hashing / kvbm-engine / kvbm-physical / kvbm-logical / kvbm-consolidator / kvbm-kernels / kvbm-config / kvbm-common / llm / rl / runtime / tokens / memory / mocker / sidecar / gpu_memory_service / backend-common / bindings / data-gen / truthy / router-plugins。（来源：`ls lib/`）

## F-04 KV 感知路由 crate
`dynamo-kv-router`：RadixTree / ConcurrentRadixTree / ConcurrentRadixTreeCompressed / ThreadPoolIndexer / PositionalIndexer / KvRouterConfig / RouterQueuePolicy / LocalScheduler；compute_block_hash_for_seq / compute_seq_hash_for_block 哈希助手。（来源：lib/kv-router/README.md:3-24）

## F-05 KV 块哈希
`lib/kv-hashing/`：block.rs / compute.rs / error.rs / request.rs / salt.rs——块级哈希与序列级哈希（block hash → sequence hash 两级）。（来源：lib/kv-hashing/src/）

## F-06 KV Block Manager 族
`lib/kvbm-*` 8 个 crate：engine（collectives/leader/object/offload/pubsub/runtime/worker/testing）、physical、logical、consolidator、kernels、config、common——面向 4 层内存层级（GPU→CPU→SSD→远程）的 KV 块管理。（来源：lib/kvbm-engine/src/{collectives,leader,object,offload,pubsub,runtime,worker}）

## F-07 调度子系统
`lib/kv-router/src/scheduling/`：SchedulerQueue（公共句柄，解析 class_index/采样快照/盖到达戳）→ SchedulerQueueActor（路由每请求过 policy queue + 同 turn 排空 backlog）→ PolicyQueue（DRR 加权轮转）→ PolicyClassQueue（每类排序与记账）→ WorkerSelector。（来源：lib/kv-router/src/scheduling/AGENTS.md:7-45）

## F-08 调度队列结构
PolicyClassQueue 含：`pending: BinaryHeap<PolicyQueueEntry>`（WorkerPlacement::Any）+ `ready_by_worker: FxHashMap<WorkerWithDpRank, BinaryHeap>` + `blocked_workers: FxHashSet` + `candidate_worker_heads: BTreeSet<WorkerLaneHead>`。BinaryHeap::peek O(1)、push/pop O(log n)。（来源：lib/kv-router/src/scheduling/AGENTS.md:58-70）

## F-09 路由策略插件
`lib/router-plugins/`：builtin 提供 `dynamo-two-tier-cost-fn`（POLICY_TYPE），catalog + 3 个 custom-policy-example（soft-pin-repin / simple-filter-score-pick / disagg-filter-score-pick）。策略经 WorkerSelectionPolicyRegistry / Factory / Parameters 注册。（来源：lib/router-plugins/builtin/src/two_tier_cost_fn.rs:1-52, Cargo.toml workspace members）

## F-10 two-tier cost function
内置选择器：①Load tier：active-request 极差 > balance_abs_threshold(32) 且最大/最小 > balance_rel_threshold(1.1) 时选最轻载 worker；②Cache tier：最大 device-KV overlap > cache_threshold(0.5)×请求块数时选持有最大 overlap 的最轻载 worker；③否则选最轻载。默认参数精确移植自 experimental/sgl-router 的 cache_aware_zmq。（来源：lib/router-plugins/builtin/src/two_tier_cost_fn.rs:21-52）

## F-11 条件拆分布置
`conditional_disagg.rs`：ConditionalDisaggDecisionInput（prompt_tokens / decode_chosen_cached_tokens / busy 状态）→ `should_bypass_remote_prefill` 决策；IslBoundingPolicy（eff_isl_threshold / eff_isl_ratio_threshold）。（来源：lib/kv-router/src/conditional_disagg.rs）

## F-12 压缩并发基数树
ConcurrentRadixTreeCompressed：每非根节点持压缩边（(LocalBlockHash, ExternalSequenceBlockHash) 对向量）；支持分裂不支持合并；per-worker cutoffs（覆盖用 cutoff 跟踪，删除缩短单 worker 覆盖而不分裂物理树）；sticky internal nodes（曾有孩子保持逻辑内部）。（来源：lib/kv-router/src/indexer/concurrent_radix_tree_compressed/README.md:9-40）

## F-13 Flash Indexer
并发全局 KV 块索引，170M ops/s（100M+），六次迭代（Python dict → jump-optimized spatial index），Dynamo v1.0.0 起为默认 indexer。jump_size 默认 64，摊还 O(D/J + W)。（来源：docs/fern/pages/blog/2026/flash-indexer.mdx）

## F-14 清理机制
`cleanup.rs`：CLEANUP_INTERVAL_MS = 5 分钟；CleanupState（try_schedule/cancel）+ CleanupGuard（drop 时 mark_completed）。清理移除 stale leaves，但不重新压缩 split 边。（来源：lib/kv-router/src/cleanup.rs:1-40, concurrent_radix_tree_compressed README:12-14）

## F-15 到期修剪
`indexer/pruning.rs`：`next_expiries: Mutex<BinaryHeap<(Reverse<Instant>, WorkerWithDpRank)>>`——按到期时间堆化 worker 覆盖修剪。（来源：lib/kv-router/src/indexer/pruning.rs:259-279）

## F-16 跟踪哈希
`tracking_hash.rs`：KEY_SIZE=32；domain 分离 `dynamo.router.tracking-hash/keyed-xxh3-v1`；TrackingHashScope / TrackingHashContext / compute_sequence_hashes。（来源：lib/kv-router/src/tracking_hash.rs:5-35）

## F-17 KV 提示协议
`kv_hints.rs`：KvHint（message_id + actions）/ KvHintAction（fetch + KvSourceLocationsPayload）/ KvTransferCandidates.best_source——KV 传输候选源选择。（来源：lib/kv-router/src/kv_hints.rs）

## F-18 错误族
KvSchedulerError::NoEndpoints（零 discovered-worker 窗口或 scaled-to-zero 时诚实报错）；IdentityParseError / IdentitySpecError；cuckoo 索引错误族（LocalCkfAdapterBuildError / GlobalCkfBuildError / GlobalCkfQueryError 等）。（来源：lib/kv-router/src/scheduling/queue.rs:2163,3065；identity.rs:343,403；indexer/cuckoo/*）

## F-19 多协议前端
Dynamo 同时服务 v1/chat/completions、v1/messages、v1/responses——统一内部表示，单部署可作任意 harness 的推理后端；typed content blocks 使 orchestrator 看到 thinking/tool call/text 边界。（来源：docs/fern/pages/blog/2026/agentic-inference-optimizations.mdx:60-90）

## F-20 Agent Hints（nvext）
`nvext.agent_hints`：priority（调度，router 队列序 + engine 优先级归一化）、osl（output sequence length 估算，router 负载均衡）、speculative_prefill（请求就绪前预热缓存）。v1 API，社区共设计。cache_control **明确不支持**（Anthropic 兼容但不作 TTL pinning）。（来源：同博客 :92-119）

## F-21 KV 内存 4 层层级
GPU HBM(~ns) → CPU pinned DRAM(~us) → Local NVMe(~ms) → Remote Storage/NIXL(~ms, RDMA)。write-through：worker 计算 KV 后块自动 GPU→CPU→disk 流动；按 sequence hash 全局去重；块注册后不可变、可被任何可达 worker 寻址。（来源：同博客 :167-177）

## F-22 WORM 访问模式
Claude Code 42-call 会话：cache reads 891K tokens vs writes 76K——11.7x read/write 比；85-97% cache hit；4 Opus 队友 97.2% 聚合命中。（来源：同博客 :40-55）

## F-23 块价值异质性
系统提示+工具定义=每 turn 复用（最高）；对话历史=高；thinking/reasoning tokens=推理循环关闭后近零复用；subagent KV=多 turn 后 agent 死（近零）。LRU 只看 recency，2-30 秒 tool call 暂停可挤出整个 prefix。（来源：同博客 :152-165）

## F-24 subagent 冷启动问题
lead 79.4% vs explore subagent 91.3% cache hit；差距几乎全部来自 teammate 首次调用的 cold-start writes。共享存储路径：一次 compute + 三次 load（NIXL RDMA）替代四次冗余 prefill。（来源：同博客 :167-177）

## F-25 选择性保留
SGLang priority-based radix eviction；TRT-LLM TokenRangeRetentionConfig（per-region，@jthomson04）；设计模式：零或多条 retention directive 挂请求/token range；evictor = LRU free list（O(1) 无标注块）+ priority queue（标注块）双结构。（来源：同博客 :183-191）

## F-26 Agent 生命周期感知
subagent 终止 / 上下文压缩（175K→40K）/ 推理循环关闭均产生 ephemeral KV；`<think>` 占生成 token ~40% 但循环关闭即 ephemeral。方案：session tagging 使 evictor 先回收 ephemeral 块并跳过 L2 write-back。（来源：同博客 :193-199）

## F-27 调度双路径
direct path（never-queues 类或零 worker 窗口）→ 直接 WorkerSelector；policy-queue path（其余请求）→ DRR 加权轮转 → select_worker → 保留容量。同 turn 排空 backlog 使释放容量优先给其他类队头（DRR），而非到达请求。（来源：lib/kv-router/src/scheduling/AGENTS.md:7-45）

## F-28 Deadline 语义
同 strict-priority tier 内：deadline 条目 earliest-due-first 排在所有非 deadline 前，再按 FCFS/LCFS/WSPT 策略分。deadline 用有序 due set 索引 + conditional actor timer 拒绝过期条目；时间用 tokio::time::Instant（paused-clock 测试一致）。（来源：lib/kv-router/src/scheduling/AGENTS.md:40-44）

## F-29 Guardrails
scheduling/AGENTS.md 末尾有 Guardrails 章节（未逐条展开，属调度不变量声明）。（来源：lib/kv-router/src/scheduling/AGENTS.md:75+）

## F-30 治理：AGENTS.md 独裁
"AGENTS.md is the canonical and only source of agent instructions at every repository scope"；每 AGENTS.md 必须配 CLAUDE.md 且只含 `@AGENTS.md`；skills 规范副本在 .agents/skills/，skills/ 与 .claude/skills/ 为符号链接。（来源：AGENTS.md:26-36）

## F-31 自有 Agent 工具链
顶层 agents/：hypothesis-challenger / hypothesis-generator / perf-analyzer / recipe-deployer / user-interviewer；.agents/skills/ 25+ 技能（analyze-aiperf-results / create-optimization-hypothesis / dynamo-agent-harness / dep-create 等）。（来源：`ls agents/` `.agents/skills/`）

## F-32 Python 扩展入口
KvRouter 类：`best_worker(token_ids, request_id, update_indexer, router_config_override)` / `get_potential_loads(token_ids)` / `generate(token_ids, model, worker_id)`——自定义路由可逐请求覆盖 router_config（如长上下文 overlap_score_credit=1.0）。（来源：agentic-inference-optimizations.mdx:126-147）

## F-33 外部集成案例
NeMo Agent Toolkit（NAT）用上述 API 构建 Thompson Sampling bandit 路由：4x p50 TTFT 降、1.5x p50 tokens/s 增、priority tagging 63% p50 TTFT 降（中等内存压力）。（来源：同博客 :149）

## F-34 测试体系
tests/ 264 文件；kv-router 5 个 bench（indexer_delegate / policy_queue / selection_core / tracking_hash / worker_selection）。（来源：`git ls-files tests/`, lib/kv-router/benches/）

## F-35 供应链与合规
deny.toml（license 检查）、DCO.md、CODEOWNERS、CONTRIBUTORS.md（160+ 社区贡献者）。（来源：根目录 + README 徽章）

## F-36 恢复游标模块（Reconciliation R-4 新增）
lib/kv-router/src/recovery/{cursor.rs, mod.rs}——lib.rs:20 `pub mod recovery;`，恢复游标（services/indexer/recovery.rs 亦存在）。（来源：lib/kv-router/src/lib.rs:20, lib/kv-router/src/recovery/）

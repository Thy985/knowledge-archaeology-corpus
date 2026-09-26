# Engineering Knowledge — dynamo（ARCH-2026-09-20-001）

> EK Graph：34 条工程知识（L1/L2 宽底座），每条声明 links（六类边）。证据可回溯到 commit `6562e7d`。

## EK-01 编排层定位：不替代引擎，协调引擎
Dynamo 是推理引擎（SGLang/TRT-LLM/vLLM）之上的编排层——disaggregated serving / intelligent routing / multi-tier KV caching / automatic scaling 把 GPU 集群变成协调推理系统。单 GPU 单模型场景明确"引擎足够"。
- Evidence: README.md:30-44, AGENTS.md:17-20
- links: contrast→EK-06（与 harness 侧 hermes 的"快照优先"对照的推理侧先例）；causal→EK-02
- aggregation: R4 主题簇（KO-06 编排层职责分离）

## EK-02 KV-aware placement：全局索引→overlap score→成本最小化
无 cache-aware 路由时 turn 2 只有 ~1/N 概率落回同 worker，每次 miss=全量 prefix 重算。Dynamo 维护全局 KV 块→worker 索引，每请求查 per-worker overlap scores，选最小化"cache miss + decode load"组合成本的 worker。
- Evidence: agentic-inference-optimizations.mdx:110-119
- links: dependency→EK-03（索引）；causal→EK-05（调度）；mechanism↔EK-04（radix 家族）
- aggregation: R2 因果链簇（KO-03 路由决策链）

## EK-03 Flash Indexer：六次迭代到 170M ops/s
并发全局 KV 块索引（Python dict → jump-optimized spatial index 六代），170M ops/s，v1.0.0 默认。jump_size=64 时摊还 O(D/J+W)。瓶颈最终是网络延迟/tokenization/hashing。
- Evidence: flash-indexer.mdx
- links: mechanism↔EK-04；constraint→EK-02（路由依赖其吞吐）
- aggregation: R4 主题簇（KO-08 性能工程方法论）

## EK-04 ConcurrentRadixTreeCompressed：radix 压缩 + 不合并
每非根节点持压缩边（LocalBlockHash/ExternalSequenceBlockHash 对）；prefill 链单节点表示，decode 直接追加叶子；per-worker cutoffs 用覆盖缩短替代每次 eviction 分裂；sticky internal nodes 避免 cleanup 后重开 fanout。**split 但永不 merge**；cleanup 删 stale leaves 不重压缩。
- Evidence: concurrent_radix_tree_compressed/README.md:9-40
- links: mechanism↔EK-03（索引家族）；constraint→EK-14（清理不合并）
- aggregation: R1 机制簇（KO-05 radix 压缩家族）

## EK-05 调度双路径：direct vs policy-queue
never-queues 类/零 worker 窗口走 direct（直接 select_worker）；其余走 PolicyQueue DRR 加权轮转。同 actor turn 排空 backlog，释放容量优先给其他类队头而非到达请求。zero-worker 窗口 surface `NoEndpoints`。
- Evidence: scheduling/AGENTS.md:7-46, queue.rs:2163
- links: causal→EK-07（priority 入队）；dependency→EK-06（队列结构）
- aggregation: R2 因果链簇（KO-03）

## EK-06 PolicyClassQueue 结构：四容器 + BinaryHeap
pending（Any 可放）+ ready_by_worker（按 WorkerWithDpRank 固定）+ blocked_workers（FxHashSet）+ candidate_worker_heads（BTreeSet 每 unblocked lane 一头）。BinaryHeap peek O(1)/push-pop O(log n)；full worker 只挡自己第一请求，不藏其他 worker 可运行请求。
- Evidence: scheduling/AGENTS.md:58-74
- links: mechanism↔EK-07（堆优先级）；dependency→EK-05
- aggregation: R4 主题簇（KO-09 调度数据结构族）

## EK-07 priority 传播：router 队列序 + engine 归一化
单一用户旋钮 priority 双层生效：router 端 BinaryHeap 按有效到达时间（高 priority 看似更早到达）；engine 端按后端归一化极性（SGLang/vLLM/TRT-LLM 解释不同），SGLang 可 priority-based radix eviction。threshold 以下请求绕过队列直选 worker。
- Evidence: agentic-inference-optimizations.mdx:120-121
- links: causal→EK-20（agent_hints 来源）；constraint→EK-05（queue 阈值门）
- aggregation: R2 因果链簇（KO-03）

## EK-08 deadline 语义：earliest-due-first + paused-clock 安全
同 strict-priority tier 内 deadline 条目先于非 deadline，再按 FCFS/LCFS/WSPT；due set 有序索引 + conditional actor timer 拒绝过期；tokio::time::Instant 保 paused-clock 测试一致。
- Evidence: scheduling/AGENTS.md:40-44
- links: mechanism↔EK-07（优先级族）；contrast→EK-21（诚实失败）
- aggregation: R4 主题簇（KO-09）

## EK-09 two-tier cost function：load gate 优先于 cache
①active-request 极差 >32 且 最大/最小 >1.1 时选最轻载；②否则最大 device-KV overlap >0.5×块数时选持有该 overlap 的最轻载；③否则最轻载。默认参数精确移植 sgl-router cache_aware_zmq。
- Evidence: two_tier_cost_fn.rs:10-52
- links: contrast↔EK-27（Python 自定义路由）；subsystem→EK-10（插件族）
- aggregation: R4 主题簇（KO-09）

## EK-10 路由策略插件注册：Registry/Factory/Parameters
WorkerSelectionPolicyRegistry + Factory + Parameters 三段式注册自定义策略；policy type 经 `worker_selection.instances[].type` 选择；无 parameters 映射时精确复现移植策略。
- Evidence: two_tier_cost_fn.rs:44-52, custom-policy-example/*
- links: mechanism↔EK-27（扩展面）；subsystem→EK-09
- aggregation: R4 主题簇（KO-09）

## EK-11 conditional_disagg：bypass remote prefill 决策
ConditionalDisaggDecisionInput（prompt_tokens/decode_chosen_cached_tokens/busy）→ should_bypass_remote_prefill；IslBoundingPolicy（eff_isl_threshold/ratio）界定有效 ISL。prefill 与 decode 分离部署时按需跳过远端 prefill。
- Evidence: conditional_disagg.rs
- links: causal→EK-24（disagg KV 一致性）；constraint→EK-05
- aggregation: R2 因果链簇（KO-07 KV 全局共享化）

## EK-12 4 层内存层级 + write-through
GPU HBM(~ns)/CPU pinned(~us)/NVMe(~ms)/NIXL RDMA(~ms)。块 write-through 自动 GPU→CPU→disk；sequence hash 全局去重；注册后不可变可寻址。disagg 场景：decode worker 写回新块→prefill 下 turn 可取。
- Evidence: agentic-inference-optimizations.mdx:167-177
- links: mechanism↔EK-13（去重）；dependency→EK-24
- aggregation: R1 机制簇（KO-07）

## EK-13 块去重与寻址：immutable + 全局注册
块计算后按 sequence hash 去重注册，不可变，任何可达 worker 可寻址——subagent 冷启动从四次冗余 prefill 变一次 compute + 三次 NIXL load。
- Evidence: agentic-inference-optimizations.mdx:167-177, kv_hints.rs
- links: mechanism↔EK-12；causal→EK-16（prefetch 依赖）
- aggregation: R1 机制簇（KO-07）

## EK-14 块价值异质 vs LRU recency
系统提示最高价值/tool defs 高/thinking 近零/subagent 近零；LRU 只看 recency——2-30 秒 tool call 暂停挤出整个 prefix。uniform eviction 对 agentic 负载结构性失效。
- Evidence: agentic-inference-optimizations.mdx:152-165
- links: causal→EK-15（选择性保留）；contrast↔EK-26（ephemeral 检测）
- aggregation: R3 不变量簇（KO-02 KV 价值分层）

## EK-15 选择性保留双结构 evictor
LRU free list（无标注块，O(1) 不变）+ priority queue（标注块）。零或多条 retention directive 挂请求/token range；TRT-LLM TokenRangeRetentionConfig per-region 控制。
- Evidence: agentic-inference-optimizations.mdx:183-191
- links: dependency→EK-14；mechanism↔EK-07（priority）
- aggregation: R3 不变量簇（KO-02）

## EK-16 prefetch hooks：tool call 返回预测
harness 用历史时序预测 tool call 何时返回→预取共享层块回 GPU；与 priority/retention 组合给 harness 全生命周期控制。
- Evidence: agentic-inference-optimizations.mdx:179
- links: dependency→EK-13；causal→EK-17（生命周期）
- aggregation: R4 主题簇（KO-04 agent-aware 基础设施）

## EK-17 生命周期感知：ephemeral KV 识别
subagent 终止/上下文压缩（175K→40K）/推理循环关闭产生 ephemeral KV；`<think>` 占生成 ~40% 但循环关闭即死。session tagging 使 evictor 先回收 ephemeral 并跳过 L2 write-back；引擎侧语义检测为探索方向。
- Evidence: agentic-inference-optimizations.mdx:193-199
- links: causal→EK-15；contrast↔EK-14
- aggregation: R3 不变量簇（KO-02）

## EK-18 多协议统一内部表示
v1/chat/completions、v1/messages、v1/responses 三端点统一内部表示；typed content blocks 使 orchestrator 见 thinking/tool/text 边界，可按块类型应用 cache/scheduling 策略。GLM-5/MiniMax2.5 内部分署供 Codex/Claude Code harness。
- Evidence: agentic-inference-optimizations.mdx:60-90
- links: mechanism↔EK-19（nvext）；dependency→EK-02
- aggregation: R4 主题簇（KO-04）

## EK-19 agent_hints：harness→orchestrator 接口
nvext.agent_hints 三字段：priority（调度）、osl（输出长度估算→负载均衡）、speculative_prefill（预热）。v1 API 社区共设计；harness 全局上下文（谁阻塞/谁刚 spawn/还有几 turn）第一次跨 API 边界。
- Evidence: agentic-inference-optimizations.mdx:92-119, nvidia-request-extensions-nvext.md
- links: causal→EK-07；contrast↔EK-20（cache_control 不支持）
- aggregation: R4 主题簇（KO-04）

## EK-20 cache_control 明确不支持
Anthropic cache_control 注释仅为 API 兼容，Dynamo 不当 self-hosted TTL pinning 指令；生产路径是 priority-driven（nvext.agent_hints.priority）。TTL/per-token-range retention 为未来 API。
- Evidence: agentic-inference-optimizations.mdx:118-119, 187-189
- links: contrast↔EK-19；constraint→EK-15
- aggregation: R4 主题簇（KO-04）

## EK-21 诚实失败：NoEndpoints
零 discovered-worker 窗口或 scaled-to-zero 时 surface NoEndpoints（而非无限等待或假成功）；scaled-to-zero 时 selection 报真实条件。
- Evidence: scheduling/AGENTS.md:46, queue.rs:693,1253,2163,3065
- links: contrast↔EK-08；mechanism↔EK-22（错误族）
- aggregation: R3 不变量簇（KO-03 诚实失败）

## EK-22 错误族分层：Identity/cuckoo/scheduler
IdentityParseError/IdentitySpecError；cuckoo 索引 Build/Assignment/Query/Ingestion 错误族；KvSchedulerError::NoEndpoints——各子系统独立错误类型，不吞错。
- Evidence: identity.rs:343,403, indexer/cuckoo/*.rs
- links: mechanism↔EK-21；subsystem→EK-03
- aggregation: R3 不变量簇（KO-03）

## EK-23 tracking_hash：keyed-xxh3 domain 分离
KEY_SIZE=32；domain `dynamo.router.tracking-hash/keyed-xxh3-v1`；TrackingHashScope/Context 隔离——同一块可有多域哈希而不冲突。
- Evidence: tracking_hash.rs:5-35
- links: dependency→EK-04（索引键）；subsystem→EK-02
- aggregation: R1 机制簇（KO-05）

## EK-24 disagg KV 一致性：decode 写回共享层
disaggregated prefill-decode：prefill worker 算 KV 经 NIXL 传 decode；decode 生成新 KV 状态；下 turn prefill 需要 prefix+turn1 tokens——共享存储使 decode 写回新块、任意 prefill 可取。
- Evidence: agentic-inference-optimizations.mdx:177
- links: causal→EK-12（write-through 支撑）；dependency→EK-11
- aggregation: R2 因果链簇（KO-07）

## EK-25 清理不变量：5 分钟 + Guard
CLEANUP_INTERVAL_MS=300000；CleanupState try_schedule/cancel + CleanupGuard drop 标记完成；清理删 stale leaves 不重压缩 split 边。
- Evidence: cleanup.rs:1-40
- links: constraint→EK-04（不合并）；mechanism↔EK-26（到期修剪）
- aggregation: R1 机制簇（KO-05）

## EK-26 到期修剪：BinaryHeap 堆化
next_expiries: Mutex<BinaryHeap<(Reverse<Instant>, WorkerWithDpRank)>>——按到期时间修剪 worker 覆盖，避免全扫描。
- Evidence: pruning.rs:259-279
- links: mechanism↔EK-25；dependency→EK-04
- aggregation: R1 机制簇（KO-05）

## EK-27 Python 绑定自定义路由
KvRouter.best_worker/get_potential_loads/generate；router_config_override 逐请求覆盖（长上下文 overlap_score_credit=1.0）；generate 可绕过默认 selector 直指 chosen_worker（session affinity）。
- Evidence: agentic-inference-optimizations.mdx:126-147
- links: contrast↔EK-09（内置 vs 自定义）；mechanism↔EK-10
- aggregation: R4 主题簇（KO-09）

## EK-28 集成案例：Thompson Sampling bandit 路由
NAT 团队：从 nvext 抽 session 元数据喂 TS bandit 成本函数；4x p50 TTFT 降、1.5x p50 tokens/s 增、priority tagging 63% p50 TTFT 降（中等压力）。计划成为 Dynamo 内置策略。
- Evidence: agentic-inference-optimizations.mdx:149
- links: contrast↔EK-09；causal→EK-10（沉淀为插件）
- aggregation: R4 主题簇（KO-09）

## EK-29 WORM 模式量化
Claude Code 42-call：891K read vs 76K write（11.7x）；85-97% cache hit；4 队友 97.2% 聚合命中。write-once-read-many 是 agentic 推理的中心优化目标。
- Evidence: agentic-inference-optimizations.mdx:40-55
- links: causal→EK-02（cache 感知路由动机）；contrast↔EK-14
- aggregation: R3 不变量簇（KO-02）

## EK-30 数据并行路由固定
ready_by_worker 按 WorkerWithDpRank（worker + dp_rank）固定请求；每 worker/rank 堆在最后请求离开时移除；全 worker 只挡自己 lane。
- Evidence: scheduling/AGENTS.md:60-70
- links: mechanism↔EK-06；subsystem→EK-05
- aggregation: R4 主题簇（KO-09）

## EK-31 KV 提示传输：KvHint/KvTransferCandidates
KvHintAction::fetch + KvSourceLocationsPayload；KvTransferCandidates.best_source 选最优传输源——跨 worker KV 迁移协议。
- Evidence: kv_hints.rs
- links: mechanism↔EK-12（层级流动）；dependency→EK-13
- aggregation: R1 机制簇（KO-07）

## EK-32 同 turn 排空 backlog：DRR 公平性
SchedulerQueueActor 路由每请求过 policy queue，并在同 actor turn 排空 ready backlog——释放容量优先给其他类队头（DRR），避免到达请求插队。
- Evidence: scheduling/AGENTS.md:20-24, 34-39
- links: mechanism↔EK-06；constraint→EK-07
- aggregation: R4 主题簇（KO-09）

## EK-33 开发治理：AGENTS.md 独裁
canonical 唯一指令源；CLAUDE.md 只含 @AGENTS.md；skills 规范 .agents/skills（skills/、.claude/skills/ 符号链接）——单一来源避免多 agent 指令漂移。
- Evidence: AGENTS.md:26-36
- links: contrast↔EK-34（符号链接一致性）；subsystem→F-30
- aggregation: R3 不变量簇（KO-03 治理不变量）

## EK-34 自有 agent 工具链：agents/ + skills
顶层 agents/（hypothesis-challenger/perf-analyzer/recipe-deployer 等）+ .agents/skills/ 25+（dep-create/dynamo-agent-harness/analyze-aiperf-results）——Dynamo 自身用 agent 驱动开发/性能分析/配方部署。
- Evidence: ls agents/, ls .agents/skills/
- links: mechanism↔EK-33（治理）；contrast↔EK-01（dogfooding）
- aggregation: R4 主题簇（KO-06 编排层哲学）

## EK-35 调度不可绕过性安全契约（Reconciliation R-1 新增）
admit_one 是必需准入路径（projected load→select→receiver closed 则跳过容量预留→预留后响应；failed delivery 必须释放容量；"Do not bypass this for normal scheduling"）；RequestGuard 以 (request_id, attempt_id) 围栏调度变更；超出 admit 尝试的 cleanup 必须带 AttemptId；admission_gate 不得删除/弱化（除非证明 selection+reservation 不超分配）；potential-load 投影必须经 ActiveSequencesMultiWorker::potential_blocks_and_tokens_at + SchedulingRequest::prefill_token_deltas（禁直接扫 per-worker ActiveSequences）；SchedulingRequest 是 effective cached tokens/overlap/allowance 唯一源；WSPT 必须 cache-aware（pinned 用 pinned cached tokens、unpinned 用 best allowed worker），禁静默回退 raw ISL；pinned/allowed 约束 selection 前验证。
- Evidence: lib/kv-router/src/scheduling/AGENTS.md:77-107
- links: constraint→EK-05（调度双路径）；constraint→EK-07（priority 传播）；dependency→EK-06（队列结构）
- aggregation: R3 不变量簇（KO-03 诚实失败/不可绕过）

## EK-36 DC KV Relay：多数据中心 KV 路由（Reconciliation R-2 新增）
DC KV Relay（Experimental）导出数据中心 KV cache 与 serving topology 的紧凑事实：endpoint-local pools + serving topology + universal publisher（WAN Protobuf/gRPC）；Cuckoo-filter（CKF）投影——DcCkfState（单 owner 可变 producer）保精确块所有权本地化，GlobalCkfIndexer（并发转置 consumer，≤16 DC pool lanes，lossy 投影）避免全量事件流跨 WAN 复制；PoolId=(identity_version, IndexerDomainId, DcId) 跨重启稳定；colliding live pools fenced-not-merged；base model + LoRA 共享物理 CKF 用 distinct salt 分离；pool presence never implies readiness。
- Evidence: docs/fern/pages/developer-guide/knowledge-base/modular-components/router/multi-dc-kv-routing.md；lib/kv-router/src/indexer/cuckoo/README.md
- links: mechanism↔EK-13（KV 全局共享）；subsystem→EK-03（索引家族）；dependency→EK-04
- aggregation: R1 机制簇（KO-05 radix/索引家族扩展面）

---

## EK 边统计
- 总 EK：36；声明边：≥76（每条 ≥2）
- 边类型覆盖：mechanism×8 / subsystem×6 / causal×10 / dependency×8 / constraint×5 / contrast×7
- 游离 EK：0
- 聚合规则覆盖：R1×3（KO-05/07/09 部分）、R2×3（KO-03/07）、R3×4（KO-02/03）、R4×5（KO-04/06/08/09）——见 KO 文件

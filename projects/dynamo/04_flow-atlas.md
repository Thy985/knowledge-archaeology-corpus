# Flow Atlas — dynamo（ARCH-2026-09-20-001）

> 七类流从真实代码/文档导出。关键 Edge 可回溯 symbol/file/condition。

## 1. Control Flow（请求路由主链）
```
request → SchedulerQueue（解析 class_index/采样快照/盖到达戳）
  → SchedulerQueueActor::Enqueue
    ├─ direct path：never-queues 类 或 零 worker 窗口
    │    → WorkerSelector::select_worker → admit_one（保留容量）
    └─ policy-queue path：
         → PolicyQueue::enqueue → PolicyClassQueue::push（按类）
         → 同 turn pop_next() → DRR winner（round_cursor/carry_class）
         → WorkerSelector::select_worker → admit_one
  → capacity 变化 → 排空 backlog（释放容量优先其他类队头）
```
Edge 回溯：scheduling/AGENTS.md:7-45；queue.rs:2163（NoEndpoints 条件）

## 2. State Flow（调度队列状态）
```
PolicyClassQueue("agents")
├── pending: BinaryHeap<PolicyQueueEntry>            # WorkerPlacement::Any
├── ready_by_worker: FxHashMap<WorkerWithDpRank, BinaryHeap>
├── blocked_workers: FxHashSet<WorkerWithDpRank>
└── candidate_worker_heads: BTreeSet<WorkerLaneHead> # 每 unblocked lane 一头
状态转移：worker full → 从 heads 移除 → capacity 变化 → 重检
deadline：due set 有序索引 + conditional actor timer 拒绝过期
到期修剪：next_expiries: Mutex<BinaryHeap<(Reverse<Instant>, WorkerWithDpRank)>>
清理：CleanupState::try_schedule → CleanupGuard（drop 时 mark_completed），5min 间隔
```
Edge 回溯：scheduling/AGENTS.md:58-74；pruning.rs:259-279；cleanup.rs

## 3. Data Flow（路由数据）
```
token_ids → compute_block_hash_for_seq（块级）→ compute_seq_hash_for_block（序列级）
  → radix 索引查 overlap scores（Flash Indexer 全局索引，170M ops/s）
  → two_tier_cost_fn（load gate → cache gate → least-loaded）
  → worker 选择（overlap + decode load 组合成本）
Python 面：KvRouter.get_potential_loads / best_worker(router_config_override) / generate(worker_id)
```
Edge 回溯：kv-router/README.md:3-24；two_tier_cost_fn.rs:10-52；agentic-inference-optimizations.mdx:126-147

## 4. Evidence Flow（KV 块证据链）
```
worker 计算 KV → 块 write-through（GPU→CPU→disk）
  → sequence hash 全局去重注册（不可变可寻址）
  → Flash Indexer 索引（哪 worker 有哪个块）
  → 路由决策证据：per-worker overlap scores（authoritative device-KV overlap）
  → 反例：sgl-router 用 approximate cache history（被替换）
```
Edge 回溯：agentic-inference-optimizations.mdx:167-177；two_tier_cost_fn.rs:30-36

## 5. Authority Flow（策略权威）
```
nvext.agent_hints.priority（harness 声明）
  → router：BinaryHeap 有效到达时间（threshold 以下 bypass）
  → engine：后端归一化极性（SGLang/vLLM/TRT-LLM 不同解释）
  → SGLang priority-based radix eviction
策略注册权威：WorkerSelectionPolicyRegistry → Factory → Parameters
  内置：dynamo-two-tier-cost-fn；custom-policy-example（soft-pin-repin 等）
开发治理权威：AGENTS.md canonical 唯一指令源（CLAUDE.md 仅 @AGENTS.md）
```
Edge 回溯：agentic-inference-optimizations.mdx:120-121,187-189；two_tier_cost_fn.rs:44-52；AGENTS.md:26-36

## 6. Memory Flow（KV 记忆层级）
```
GPU HBM(~ns) → CPU pinned DRAM(~us) → Local NVMe(~ms) → Remote Storage/NIXL(~ms RDMA)
worker 计算 → write-through 自动流动 → 全局去重 → 任何 worker 可寻址
subagent 冷启动：一次 compute + 三次 load（NIXL）替代四次冗余 prefill
disagg：decode worker 写回新块 → prefill 下 turn 可取
selective retention：LRU free list（无标注块）+ priority queue（标注块）
ephemeral 识别：subagent 终止/上下文压缩/推理循环关闭 → session tagging → 跳过 L2 write-back
prefetch：harness 预测 tool call 返回 → 预取块回 GPU（建设中）
```
Edge 回溯：agentic-inference-optimizations.mdx:167-199

## 7. Policy Flow（治理闭环）
```
决策：agentic 负载需要块价值感知（非 uniform eviction）
  → 批准：nvext.agent_hints v1（社区共设计）
  → 策略：priority 传播 + selective retention + session tagging（未来）
  → 执行：router 队列序 + engine 归一化 + evictor 双结构
  → 反馈：NAT bandit 路由（4x TTFT 降）→ 计划内置为策略
反例路径：cache_control 不被采纳（兼容不承诺）；单 GPU 场景引擎足够
```
Edge 回溯：agentic-inference-optimizations.mdx:118-119,149,183-199

---

## Flow 验证摘要
- 7/7 类流全部存在且可回溯符号
- 关键 Edge：NoEndpoints（queue.rs:2163,3065）、BinaryHeap（scheduling/AGENTS.md:63）、two-tier gates（two_tier_cost_fn.rs:21-48）、write-through（blog:167-177）
- 反例路径已标（direct path / bypass / cache_control 拒绝 / 单 GPU 免责）

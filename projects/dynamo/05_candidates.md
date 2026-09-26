# Candidates — dynamo（ARCH-2026-09-20-001）

> 不能确认的内容留在候选。Hypothesis 不冒充 Fact；Cross-project Candidate 不写成已验证 Principle。

## C-01 agent_hints 会成为 agent 推理协议第三面（Hypothesis, cross-project）
- 当前证据：Dynamo nvext.agent_hints v1（priority/osl/speculative_prefill）；文档称与 MCP/A2A 构成协议栈三层
- 缺失证据：仅 Dynamo 一家采用；无第二实现方；社区反馈未闭环
- 验证路径：观察 2026Q4 是否出现 vLLM/SGLang 侧等价 hints 协议；NAT 集成是否带动生态
- Epistemic: Hypothesis（cross-project validation pending）

## C-02 session tagging 会落地为 KV 生命周期协议（Hypothesis）
- 当前证据：blog 明言"设计空间 wide：harness-driven tagging / engine-native / hybrid"；TTL/per-token-range 为 future API
- 缺失证据：无实现；engine 侧 `<think>` 语义检测仅探索
- 验证路径：KVBM 后续 release 的 retention API 变化
- Epistemic: Hypothesis

## C-03 推理侧 cache 管理与 harness 侧 prompt cache 管理的跨项目对照（Cross-project Candidate）
- 当前证据：Dynamo（推理侧：块价值分层/共享化/prefetch）vs hermes-agent（harness 侧：冻结快照/不 mid-session 改写）vs letta（git 投影）
- 缺失证据：三方在同一负载上的量化对比未做；"哪侧管理更优"无定论
- 验证路径：用同一 Claude Code 会话 trace 跑三方；对比 cache hit 与 TCO
- Epistemic: Hypothesis（跨项目候选，不得升 Principle）

## C-04 DRR 多类调度的真实 agent 负载证据不足（Observation）
- 当前证据：PolicyClassQueue("agents") 类存在（scheduling/AGENTS.md:62）；NAT bandit 路由证明自定义策略价值
- 缺失证据：内置 agents 类的线上配置/基准数据未在仓库呈现；"agents 类 vs latency 类"权重如何设无文档
- 验证路径：寻找 recipes/deploy 中的多类配置样例
- Epistemic: Observation

## C-05 two_tier_cost_fn 移植保真度（Observation）
- 当前证据：默认参数精确等于 sgl-router cache_aware_zmq；用 authoritative device-KV overlap 替换 approximate cache history；tie 按候选行序（host 未定）解析
- 缺失证据：行为等价未用 benchmark 证明；tie 语义与内置 selector（均匀采样）不同但无解释
- 验证路径：sgl-router 对照 benchmark
- Epistemic: Observation

## C-06 Dynamo 自有 agent 工具链的成熟度（Observation）
- 当前证据：agents/ 5 角色 + .agents/skills/ 25+；AGENTS.md 独裁治理
- 缺失证据：hypothesis-challenger/perf-analyzer 是否生产使用、成功率、CI 集成度未在仓库呈现
- 验证路径：检查 CI workflow 中 agent 工具调用；issue/PR 中 dogfooding 证据
- Epistemic: Observation

## C-07 170M ops/s 的可复现性（Hypothesis）
- 当前证据：flash-indexer.mdx 声称 170M ops/s；bench 目录存在（lib/kv-router/benches/ 5 个）
- 缺失证据：硬件环境/复现脚本细节；独立第三方复现
- 验证路径：在标准硬件跑 INDEXER_BENCH.md 流程
- Epistemic: Hypothesis

## C-08 调度 deadline 语义的实际用途（Observation）
- 当前证据：deadline 机制实现完整（due set + conditional timer + paused-clock 测试）
- 缺失证据：哪个真实负载/协议携带 deadline（SLA？定时任务？）未在仓库/文档呈现；可能是为 SLA 驱动的 Planner 预留
- 验证路径：搜索 deadline 字段的 producer（Planner/网关层）
- Epistemic: Observation

## C-09 DC KV Relay 的生产采纳路径（Observation, Reconciliation R-7 新增）
- 当前证据：multi-dc-kv-routing.md 明示 Experimental；CKF producer/consumer 双所有权模型实现完整（cuckoo/）
- 缺失证据：fenced-not-merged 语义是否被多 DC 消费者遵守无测试证据；WAN 投影的规模化验证数据未呈现
- 验证路径：kv-dc-relay 部署配置文档 + 集成测试；观察 Experimental 标记何时移除
- Epistemic: Observation

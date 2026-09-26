# Validation & Evidence — dynamo（ARCH-2026-09-20-001）

> 六类 Auditor 判定 + 反例 + 质量指标。先 Blind Reconstruction，不弱化证据标准换全绿。

## 1. Truth Auditor（Source Fidelity）
- 全部 35 Facts 可回溯 commit `6562e7d` 内符号/文件/文档行
- 高风险声明核验：
  - "170M ops/s" → flash-indexer.mdx 原文（docs 声称，非本地实测）→ 标 EK-03 为"文档声称"，C-07 记可复现性候选
  - "11.7x read/write" → blog 内部图（Claude Code 42-call 会话）→ 转述保留来源，不夸大为普适
  - "cache_control 不支持" → blog:118-119 双处明示 → CONFIRMED
- 结论：无事实错误；2 处数值保留来源属性（非独立验证）

## 2. Coverage Auditor
- 覆盖：架构（F-01~03）/ 路由（F-04/09/10/11）/ 索引（F-05/12/13/16）/ 调度（F-07/08/27/28/30）/ KV 管理（F-21~26）/ 治理（F-30/31/35）/ 扩展（F-32/33）/ 测试（F-34）
- 遗漏检查：库 lib/llm、rl、runtime、tokens 仅记目录（F-03），未深入——本 run 聚焦"agentic 推理面"（路由/KV/调度），引擎内实现留待后续
- 反例搜索：direct path / threshold bypass / NoEndpoints / cache_control 拒绝 / 单 GPU 免责 / split-no-merge / sticky nodes 均已在 EK/KO 记录
- 结论：主题聚焦型覆盖完整；明确声明未覆盖范围

## 3. Flow Auditor
- 7 类流逐条验证：Control（scheduling/AGENTS.md:7-45）、State（:58-74）、Data（kv-router README + two_tier_cost_fn.rs）、Evidence（blog:167-177）、Authority（:120-121 + AGENTS.md:26-36）、Memory（blog:167-199）、Policy（:118-119,149,183-199）
- Edge 回溯率：全部关键 Edge 有 symbol/file/line；无虚构流
- 反例路径：direct path、bypass、fallback（least-loaded）已标

## 4. Abstraction Auditor
- 升维审查：KO-02/KO-04 标 L4 Cognitive Model——证据为 Dynamo 内部论证 + hermes/letta 跨项目归纳（标记 cross-project pending）；KO-06 的"分层产品互补"模式三项目对照成立但保留 pending
- 降级审查：KO-09 降为 L2（调度数据结构组合，跨项目可迁移未验证）
- 无"单案例→Pattern"错升；无"ADR→实现事实"混用（blog 为设计文档，实现证据单独取自代码）
- 结论：无错误升维

## 5. Counterexample Hunter
| KO | 反例数 | 关键反例 |
|---|---|---|
| KO-01 | 3 | 单 GPU 免责 / thinking 高生成低价值 / chat 负载 LRU 不失效 |
| KO-02 | 3 | hermes 耦合 / letta 耦合 / 单 worker 无需求 |
| KO-03 | 3 | cache_control 兼容不承诺 / scaled-to-zero 真实报错 / authoritative 替换 approximate |
| KO-04 | 3 | cache_control 拒绝 / v1 未稳定 / prefetch 建设中 |
| KO-05 | 3 | split-no-merge 代价 / sticky 空间代价 / ThreadPool 非 radix |
| KO-06 | 2 | NAT 内置策略 / wheel 依赖引擎层 |
| KO-07 | 3 | 4 层建设中 / NIXL 硬件依赖 / 冷启动差距未全消除 |
| KO-08 | 3 | bench 环境依赖 / spatial 复杂度 / 复现性未独立验证 |

## 6. Epistemic Auditor
- Fact/Observation/Hypothesis/Pattern/Model/Principle 无混淆：
  - blog 设计意图 → EK 标"设计文档"；代码符号 → Fact
  - C-01/C-02/C-03/C-07 明确 Hypothesis；C-04/05/06/08 明确 Observation
  - KO-02/KO-04 的跨项目归纳标 pending，不写 Principle
- 无"Cross-project Candidate 写成已验证 Principle"

## 质量指标
| 指标 | 值 |
|---|---|
| Facts | 35（全部可回溯） |
| EK | 34（≥68 边，平均 2.0，0 游离） |
| KO | 9（L2×1/L3×6/L4×2；aggregation 100%：R1×3/R2×3/R3×4/R4×5） |
| Candidates | 8（Hypothesis×4 / Observation×4） |
| Flows | 7/7 |
| 反例 | 22（KO 预算全覆盖） |
| 未覆盖声明 | lib/llm、rl、runtime 引擎内实现（非 agentic 面） |

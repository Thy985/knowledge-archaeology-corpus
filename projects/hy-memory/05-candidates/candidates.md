# 05 · Candidates（未验证假设 / 待补证据）

> 严格区分：本文件内容**不得**被当作已验证知识引用。每条含"当前证据 / 缺失证据 / 验证路径"。

---

## C-01 · 场景路由的静默回退缺少对账机制
- **type**: Cross-project Candidate（可观测性）
- **hypothesis**: 如果 coding→chat 的静默回退（judge 失败 / 无 tool / 强制但无 trace）在生产中高频发生，用户会以为"写了 coding memory"但实际落到 chat 层，且只有一条 warning 日志，长期会形成"记忆质量莫名下降"的错觉。
- **当前证据**: EK-04 / EK-05；`client.py:3001-3005` 只有 `logger.warning`；返回值含 `resp["variant"]`，但未见对账/统计接口。
- **缺失证据**: 生产日志中该 warning 的出现率；是否有 metrics 暴露 fallback 次数；用户侧是否能感知。
- **验证路径**: 检索 `_store_coding_log` 是否记录 judge fallback 原因并可聚合；在测试环境构造三类失败输入，检查返回值与日志。
- **若成立的价值**: 记忆系统的"降级"必须可观测，否则等于静默丢功能。

## C-02 · `fact_chain_summary` 的语义与"是否保留历代说法"
- **type**: Evidence Gap（**已修正**：字段存在，见 Independent Validation V-06）
- **hypothesis**: Coding 的 superseded fact 通过 `fact_chain_summary` 保留可追溯性（README 声明）。**字段确实存在并参与 upsert**，但"它是否保留历代说法、检索时是否展开"尚未读完。
- **当前证据**: `coding/store.py:112`（建表列）、`:364`（读取）、`:1425/1441`（upsert 写入）、`:1454`（落值）。
- **缺失证据**: 写入时是"只存上一版摘要"还是"累积历代"；召回接口是否返回该字段。
- **验证路径**: 读 `_upsert_fact_sql_sync` 对 `fact_chain_summary` 的赋值逻辑；在测试库连续 supersede 三次，比对字段内容。
- **修正说明**: 初稿判断"未在代码中定位"是错误的（grep 范围不足），已在 Reconciliation R-03 修正。不再作为"文档过度承诺"风险。

## C-03 · L1 写入幂等键的覆盖范围
- **type**: Hypothesis（可靠性）
- **hypothesis**: EK-08 要求"reconcile 失败后由调用方重跑"，因此 L1 写入必须幂等；但本轮只确认 `request_id` 贯穿，未确认所有 L1 upsert 都以它为键——若同一 request 重跑会产生重复 raw。
- **当前证据**: `client.py:1724-1810`（request_id 生成/透传）；`writer.py:2660-2705`（L1 upsert）。
- **缺失证据**: `vector_store.upsert` 的 id 生成规则；是否 `id=request_id+index`；重复写入行为。
- **验证路径**: 读 `data/vector_store_base.py` 的 id 构造；在临时库连续 upsert 同 request 两次，比对条数。
- **若证伪的影响**: KO-02 的"可重跑"前提被削弱，all-or-nothing 会变成"重复污染"。

## C-04 · SHADOW 同时表示"低置信度"与"已删除"
- **type**: Design Observation → Candidate
- **hypothesis**: 一个状态承载两种语义（不确定 vs 不成立），会让运维无法区分"证据不足"与"被新事实推翻"；未来若需要"重新评估低置信度记忆"，无法只筛出前者。
- **当前证据**: `models/memory.py:121-128` 的枚举注释；`reconciler.py:18-26`（DELETE→SHADOW）。
- **缺失证据**: 是否有 `reason`/`status_reason` 字段区分；查询接口是否暴露。
- **验证路径**: 检查 `MemoryNode` 是否有 shadow_reason 字段；检索写入处是否记录原因。
- **性质**: 不是 bug，是**状态语义压缩**的代价；若产品需要"复审队列"，会成为真实缺口。

## C-05 · 版本链无保留策略且可能记录敏感内容
- **type**: Cross-project Candidate（隐私/成本）
- **hypothesis**: `round.said` 存用户原话、`round.patch` 存真实 diff，两者都可能包含密钥/内部路径；链只追加、`body` 每轮覆写，但未见 TTL/容量上限/脱敏。
- **当前证据**: `chain_store.py:148-170` schema；`_add_round` 无裁剪；无 `DELETE FROM round` 逻辑（本轮 grep 范围）。
- **缺失证据**: 是否有外部清理脚本；`MEMORY_CODING_CHAIN_*` 中是否有保留配置；MemHub 是否有"删除会话即清链"的联动。
- **验证路径**: grep `DELETE FROM` / `vacuum` / `retention`；检查 `delete_all` 是否清理 chain DB。
- **若成立的风险**: 违反"最小必要存储"；企业部署时成为合规障碍。

## C-06 · MemHub 派生视图与引擎状态的失效协议
- **type**: Evidence Gap（一致性）
- **hypothesis**: MemHub 只存 `memory_id` 外键，但 UI 需要列表/计数/队列等派生视图；若引擎侧 supersede 后产品层缓存不失效，会出现"UI 显示已被推翻的记忆"。
- **当前证据**: `ARCHITECTURE.md` 原则 3；`api.py` 存在 `list_source_grants` 等管理视图，但记忆列表的缓存策略未读。
- **缺失证据**: 是否有事件/轮询/版本号；列表接口是实时穿透还是缓存。
- **验证路径**: 读 `api.py` 中 memories 列表实现；检查是否有 `updated_at` 游标或 SSE。
- **性质**: 属"单一权威"模式的必然配套问题，不是对 KO-06 的反驳。

## C-07 · 厂商指标与公开基线的可比性
- **type**: Cross-project Candidate（评测）
- **hypothesis**: README 的 LoCoMo 72.6% / MemoryArena 55.31 与 Mem0 的 92.5 不可直接比较（题目子集、裁判模型、adversarial 是否计入、token 预算不同）。
- **当前证据**: README 脚注写明 LoCoMo 为"1,527 道非 adversarial"；Mem0 论文口径未见。
- **缺失证据**: 统一 harness 下的同题重跑结果。
- **验证路径**: 用 corpus 中 mem0/graphiti 的 benchmark 产物与同一评测脚本复跑。
- **当前处理**: 所有厂商指标**未进入 Fact 层**，只在 Overview §5 作为"文档声明"列出。

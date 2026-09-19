# Reconciliation — ARCH-2026-09-20-001（dynamo）

> 合并独立验证修正。修改限 Corpus Artifact 层，不改生产 Skill。

## R-1 补 Guardrails 调度安全不变量（EK-35）
- 修正：考古 F-29 仅记"有 Guardrails 章节"→ 展开为完整 EK
- 新增 EK-35：调度不可绕过性安全契约——admit_one 必需准入路径（projected load→select→receiver closed 跳过预留→预留后响应，failed delivery 必须释放容量，禁止 bypass）；RequestGuard `(request_id, attempt_id)` 围栏；cleanup 超 admit 尝试必须带 AttemptId；admission_gate 不得删除/弱化；potential-load 投影必须走 ActiveSequencesMultiWorker::potential_blocks_and_tokens_at + SchedulingRequest::prefill_token_deltas；SchedulingRequest 是 effective cached tokens/overlap/allowance 唯一源；WSPT cache-aware 禁静默回退 raw ISL；pinned/allowed 约束 selection 前验证
- Evidence: lib/kv-router/src/scheduling/AGENTS.md:77-107
- links: constraint→EK-05（调度双路径）、constraint→EK-07（priority 传播）、dependency→EK-06（队列结构）
- 影响：调度安全面从"占位"补为完整工程知识；KO-03（诚实失败/不可绕过）获新支撑

## R-2 补 DC KV Relay + Cuckoo Indexer（EK-36）
- 修正：考古遗漏多 DC KV 路由面
- 新增 EK-36：DC KV Relay（Experimental）——endpoint-local KV pools + serving topology + universal publisher（WAN Protobuf/gRPC）；Cuckoo-filter 投影（DcCkfState producer 单 owner / GlobalCkfIndexer consumer 16 DC lanes lossy 投影）避免全量事件流跨 WAN 复制；PoolId=(identity_version, IndexerDomainId, DcId) 跨重启稳定；colliding pools fenced-not-merged；base+LoRA 共享物理 CKF 用 distinct salt 分离；pool presence ≠ readiness
- Evidence: docs/fern/pages/developer-guide/knowledge-base/modular-components/router/multi-dc-kv-routing.md；lib/kv-router/src/indexer/cuckoo/README.md
- links: mechanism↔EK-13（KV 共享）、subsystem→EK-03（索引家族）、dependency→EK-04
- 影响：覆盖"KV 记忆跨 DC"扩展面；C-09 新增

## R-3 补 salt 隔离语义（并入 EK-23）
- 修正：EK-23 只记 tracking_hash，补 salt=prefix-cache isolation key
- 追加内容：salt 是 cache 隔离判据——应共享前缀的请求必须同 salt，不应共享的必须异 salt；(None,None)→CHAIN_XXH3_SEED；multimodal 不进 salt（折叠进 per-block hashing 保图像前前缀共享）
- Evidence: lib/kv-hashing/src/salt.rs:10-30
- 影响：隔离契约显式化

## R-4 补 recovery 模块（并入 EK-22 错误族/新增 F-36）
- 修正：考古首次 grep 误判 recovery 不存在
- 新增 F-36：lib/kv-router/src/recovery/{cursor.rs, mod.rs} 恢复游标模块（lib.rs:20 pub mod recovery）
- 影响：项目地图完整性

## R-5 收紧 KO-06 scope（OVER_GENERALIZED 修正）
- 修正：KO-06"分层产品以互补而非替代进入生态"在三项目（Dynamo 编排层/hermes 多前端/letta 记忆层）的"互补方式"差异大（不替代引擎 vs 不重造 CLI vs 不重造 runtime）
- 收紧：L3 Pattern 保留但 scope 改为"编排/平台层项目通过增量价值而非替代进入生态"（Dynamo 单项目强证据 + hermes/letta 弱对照），反例增补
- 影响：消除过度归纳

## R-6 run_metadata 同步
- ek: 34→36；facts: 35→36（F-36 新增）；candidates: 8→9（C-09 新增）；验证统计补 MISSING×4/OVERGEN×1 修正

## R-7 candidates 新增 C-09（DC KV Relay 成熟度）
- C-09：DC KV Relay 的 Experimental 标记 → 生产采纳路径不明（Observation）；fenced-not-merged 语义是否被多 DC 消费者遵守无测试证据

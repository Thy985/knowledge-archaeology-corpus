# Independent Validation Report — ARCH-2026-10-27-001

> Auditor 盲重建：独立重读仓库关键文件（不把主考古当事实源），对比后判定。禁止修改原考古产物；修正经 Reconciliation 写入 corpus。

## 判定统计
| 判定 | 数量 | 项 |
|---|---|---|
| CONFIRMED | 9 | IV-2,3,4,5,6,7,8,9,10,12,13,14,15（13 项） |
| CONTRADICTED | 2 | IV-1（Command 计数）、IV-11（Completion 计数） |
| DOWNGRADED | 1 | IV-15（表族 15+→13 目录级） |
| MISSING | 4 | metadata-server 实现 / query_engine SQL 面 / replicated_loglet sequencer 故障转移 / object-store-plugin（主考古已声明 PARTIAL，确认遗漏） |
| NEEDS_HUMAN_REVIEW | 1 | IV-2'（is_resumable fail-open 语义） |

## 3 成功（重点 CONFIRMED）
1. **is_resumable 跳过已完成**：`JournalEntry::Entry(e) => e.header().is_completed().unwrap_or(true)`；`Completion(_) => false`（storage-api/journal_table/mod.rs:41-46）——重放跳过已完成 entry 语义属实
2. **OnMaxAttempts Pause|Kill**：`Pause=0 #[default] / Kill=1`（schema/invocation_target.rs:73-78）——默认挂起属实
3. **suspend 前 notification 优先**：suspend.rs:80 "Resuming instead of suspending service because a notification is already available."——竞态防御属实

## 3 错误（CONTRADICTED / DOWNGRADED）
1. **[CONTRADICTED] journal Command 计数 17→20**：command.rs Command 枚举实际 20 个变体（Input/Output/GetLazyState/SetState/ClearState/ClearAllState/GetLazyStateKeys/GetEagerState/GetEagerStateKeys/GetPromise/PeekPromise/CompletePromise/Sleep/Call/OneWayCall/SendSignal/Run/AttachInvocation/GetInvocationOutput/CompleteAwakeable）。主考古 EK-R01/00_overview/01_project_map 均写"17 种"——计数错误，需 reconciliation 修正
2. **[CONTRADICTED] Completion 计数 12→11**：notification.rs 实际 11 个 Completion struct（GetLazyState/GetLazyStateKeys/GetPromise/PeekPromise/CompletePromise/Sleep/CallInvocationId/Call/Run/AttachInvocation/GetInvocationOutput）；NotificationType=Completion|Signal 两分支。主考古"12 类 Completion"不精确
3. **[DOWNGRADED] partition-store 表族 15+→13 目录级**：partition-store/src/ 下实际 13 个表族目录（dedup/fsm/inbox/invocation_status/journal/journal_v2/locks/outbox/promise/service_status/state/timer/vqueue）；partition_store.rs 为容器非表族。主考古"15+ 表族"偏多（若计入容器与内部列族可近似，但目录级应为 13）

## 遗漏（MISSING，确认主考古 Coverage=PARTIAL 声明）
- metadata-store/server 全实现（分区→节点拓扑权威细节）
- query_engine（DataFusion）SQL 查询面能力边界
- replicated_loglet sequencer 故障转移/日志截断细节
- object-store-plugin / cloud-tunnel-client（云存储面）
- admin/CLI 面（cli/src/main.rs 子命令全集）

## 附加发现（Observation，非主考古错误）
- **IV-2' [NEEDS_HUMAN_REVIEW]**：`is_completed().unwrap_or(true)` 对 header 解析失败**默认视为已完成**（fail-open）——若 header 损坏，重放会跳过该 entry 而非重试。与"恢复安全"直觉（fail-closed）相反；需确认是否有其他保护（如 entry 校验和/完整性保证）兜底。列入 Needs Human Review

## Benchmark case（供 skill CI 使用）
**Case**: "durable execution 运行时恢复时是否跳过已完成 journal entry？"
- 期望：跳过（journal 次序保证确定性）
- Restate 证据：is_resumable（storage-api/journal_table/mod.rs:41-46）+ append→Lsn（bifrost/appender.rs:92）+ FSM last_applied_lsn（processor/fsm.rs:31-37）——**PASS**
- 反例面：header 损坏默认完成（IV-2'）——恢复语义的边缘 case 值得进 benchmark 反例集

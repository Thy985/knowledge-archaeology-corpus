# Validation & Evidence — dbos-transact-py（ARCH-2026-10-28-001）

> 六审计（先 Blind Reconstruction）；验证结果保留 Contradictions/Counterexamples。

## 审计结果
| 审计 | 结果 | 说明 |
|---|---|---|
| Truth | **PASS_WITH_RESERVATIONS** | 全部 EK/KO 可回溯 file:line；保留项：README 宣称（"no additional infrastructure"）为意图证据非实现事实；企业 sharding 闭源面不可审计 |
| Coverage | **PARTIAL** | 覆盖：执行内核/checkpoint/queue/恢复/通知/数据源/去重/序列化。遗漏：CLI 子命令全集（_cli.py）、Kafka 消费实现（_kafka.py）、scheduler cron 实现（_scheduler.py 内部）、fork/rewind 全路径、debouncer |
| Flow | **PASS** | 34 Edge 全可回溯 symbol/file/condition；Flow→KO 交叉校验 8/8 PASS |
| Abstraction | **PASS_WITH_PENDING** | KO-D01/D03/D08 为 L4/L5 需跨项目验证（restate 对照）；未弱化证据标准 |
| Counterexample | **PASS** | 定向反例：SQLite 非生产（排除 C-D07）；错误不可序列化 fail-open（D07）；owner 丢失 park（D04）；无 0 反例 KO |
| Epistemic | **PASS** | Fact/Observation/Hypothesis/Pattern/Model/Principle 严格区分；C-D01/02/03 标假说；KO-D08 标 Cross-project validation pending |

## 质量指标
- EK 24 条：平均出边 2.38 / 游离 0 / 聚合规则覆盖率 100%
- KO 8 个：R1×2 / R2×2 / R3×1 / R4×3；L3×2 / L4×5 / L5×1；Cross-project pending 3（D01/D03/D08）
- Flow 34 Edge（Control 5 / State 5 / Data 5 / Evidence 4 / Authority 5 / Memory 4 / Policy 6）
- Candidates 7（跨项目假说 3 / 暂定 3 / 已排除 1）
- 采证锚 45+（全部 code file:line）

## 待 IVR 攻击面（10 项）
1. owner_xid 所有权竞争路径（含 resume/cancel/recovery 交错）
2. dequeue RETURNING 与 SQLite 兼容性（≥3.35 断言）
3. 步骤 checkpoint 三态（成功/失败/不可序列化）
4. data source checkpoint 与 workflow 完成竞态
5. asyncio.wait durable 语义（恢复时 done/pending 重建）
6. LISTEN/NOTIFY 与轮询双路径
7. dead-letter/resume 预算语义
8. duplicate_policy return-existing attach 语义
9. 序列化 per-row（DEFAULT/PORTABLE）
10. SQLite 迁移并发（#923）

## Benchmark case（供 skill CI）
**Case**: "库级 durable execution（数据库即 journal）在恢复时如何保证不重复执行已完成步骤？"
- 期望：查表跳过（checkpoint 结果缓存）而非重跑
- DBOS 证据：check_operation_execution（_sys_db.py:3648）+ record_operation_result（:3410）+ is_resumable 对照（restate journal_table:41-46）
- 反例面：错误不可序列化→重跑（D07）；owner 丢失→park 轮询（D04）

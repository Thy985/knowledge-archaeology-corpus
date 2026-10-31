# Restate — Knowledge Archaeology

**一句话定位**：**步骤持久化派 durable execution 的独立运行时侧代表**——"the Durable Runtime for your Agents and Backends"。每次 handler 执行切成 journal 条目（journal_v2，20 种 Command），每步写持久化存储（Bifrost 日志 + RocksDB 表），崩溃/重启后从最后完成的 entry 恢复；幂等、挂起恢复、keyed 调度、元数据版本化是其核心机制。

## 考古索引
| 产物 | 路径 | 说明 |
|---|---|---|
| Overview | `00_overview.md` | 项目定位 + 核心发现 + 三层配比 |
| Project Layer | `01_project-layer/` | 架构/子系统/数据结构/配置/治理/测试 |
| Engineering Knowledge | `02_engineering-knowledge/` | EK Graph（44 条，全部带 links，Reconciled 后不删除） |
| Knowledge Layer | `03_knowledge-layer/` | 9 个 Generalized KO（R1-R4 聚合，可回溯 EK） |
| Flow Atlas | `04_flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| Candidates | `05_candidates/` | 6 条 Hypothesis / 跨项目候选（C-1 已由 IVR 澄清为 Fact） |
| Validation | `06_validation/` | 六审计 + 独立验证（IVR）+ Reconciliation |
| Run 归档 | `archaeology-runs/ARCH-2026-11-01-001/` | 本次 run 原始产物（job_manifest/snapshot/package，不覆盖历史） |

## 考古日期 / 版本
- 日期：2026-11-01
- repository：`restatedev/restate` @ `c30b4bce`（v1.8.2-dev，main）
- Skill：knowledge-archaeology v3.2
- run_id：`ARCH-2026-11-01-001`（mode=initial，P0）
- License：**BSL 1.1**（Change Date: 4 years after release → Apache-2.0）

## 核心命题
1. **Journal 即真相**：每步执行写 journal_v2（append 原子：put_journal + length 更新 + put_status 同调用，debug_assert 无 gap），崩溃从最后完成 entry 恢复——步骤持久化派主机制。
2. **幂等是显式状态机**：idempotency key → 确定性 InvocationId → handle_duplicated_requests 按 InvocationStatus 分派（Free=新调用 / in-flight=append sink / Completed=回放响应）；workflow 走唯一标识特例（忽略 key 返回 WORKFLOW_ALREADY_INVOKED）。
3. **幂等分片是不可变契约**：deterministic_partition_key（service_key→idempotency_key）+ UNSCOPED_SERVICE_PARTITION_KEY_FANOUT=512 + controlled idempotent sharding——改分片即破坏去重。
4. **挂起/恢复优先解析**：OnSuspendCommand 先尝试用现有 notifications resolve awaiting_on（UnresolvedFuture 树），能解析直接 Resume；否则写 SuspendedEvent + 转 Suspended + 挂起 vqueue entry。
5. **日志与状态分离**：Bifrost（"durable interconnect"）loglet append 返回 offset 即 durable committed；分区 FSM 消费日志推进 applied_lsn，快照+重放恢复。
6. **治理证据驱动**：AGENTS.md/CLAUDE.md 强制 evidence-based + cite file:line + `cargo nextest run`；CI 18 workflows；存储迁移（migrations/ + PersistedFeatures + VersionBarrierCommand）双轨。

## 质量指标（ARCH-2026-11-01-001）
| 指标 | 值 |
|---|---|
| Engineering Knowledge | 44（带 links 100%，孤立 0，平均出边 ~2.1） |
| Generalized KO | 9（aggregation_rule 覆盖率 100%，平均簇规模 ~4.4） |
| Candidates | 6（C-1 已澄清为 Fact） |
| Flow Atlas | 7 类流（关键 Edge 可回溯 symbol/file/condition） |
| IVR 判定 | CONFIRMED 9 / PARTIALLY_CONFIRMED 1（C-1 澄清）/ DOWNGRADED 0 |
| 反例 | 9 KO × ≥1（3 处措辞收窄） |

## 对照价值
与 `projects/agent-substrate`（ARCH-2026-10-31，挂起恢复派）构成同一问题两条技术路线；Restate 的 suspend 也持久化 awaiting_on 说明两派在真实实现中并非互斥。

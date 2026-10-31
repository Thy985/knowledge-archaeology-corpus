# Validation & Evidence — Restate（ARCH-2026-11-01-001）

> v2 规则：所有 Validator 先 Blind Reconstruction（不把考古结果当事实源，独立重读仓库）再对比。
> 反例预算制：每个 L3+ KO ≥3 个定向反例攻击。

## 0. 审计方法（Blind Reconstruction 声明）
- 未依赖 Package 00-05 结论；独立重读 `crates/types/src/journal_v2/`、`crates/worker/src/partition/state_machine/`、`crates/bifrost/src/`、`crates/vqueues/src/`、`crates/partition-store/src/` 关键符号后重新归纳，再与本 Package 对照。
- 静态考古（无 Docker）：所有断言上限为 S4（测试验证）；无 S5 运行时验证。

## 1. Truth Auditor（这句话是真的吗）
| 断言 | 判定 | 证据 |
|---|---|---|
| journal_v2 20 种 Command | CONFIRMED | command.rs:50 枚举实际计数 |
| append 同事务三写（put_journal+length+put_status） | CONFIRMED | mod.rs:4968-5010 同一函数体 |
| handle_duplicated_requests 按状态分派 | CONFIRMED | mod.rs:1076 + idempotency.rs 测试 |
| workflow 忽略幂等 key 返回 WORKFLOW_ALREADY_INVOKED | CONFIRMED | mod.rs workflow 分支 + tests/workflow.rs |
| UNSCOPED_SERVICE_PARTITION_KEY_FANOUT=512 | CONFIRMED | identifiers.rs:245 字面值 |
| InvocationStatus 8 态 | CONFIRMED | storage-api/invocation_status_table/mod.rs:144 |
| BSL 1.1 license | CONFIRMED | LICENSE 文件头部原文（修正 snapshot 初记的 Apache-2.0） |
| OnSuspendCommand 先尝试解析再挂起 | CONFIRMED | suspend.rs 逻辑顺序 |

## 2. Coverage Auditor（还有什么没发现）
| 范围 | 状态 | 说明 |
|---|---|---|
| journal 双表（v1/v2）迁移行为 | PARTIAL | C-2 待验证（未追 complete 迁移调用） |
| inbox→vqueue 迁移完成度 | PARTIAL | C-3 待验证 |
| 跨分区锁协调 | GAP→C-4 | 未展开跨分区锁机制 |
| ReplicatedLoglet 读一致性 | GAP→C-6 | 未追 read_path 状态机 |
| CLI/server 运维面 | 未覆盖 | 本轮聚焦运行时内核；CLI 属运维工具 |
| SDK 协议细节（service-protocol v4） | PARTIAL | 只到 crate 结构层 |

## 3. Flow Auditor（Flow Atlas 是否对应代码）
| Flow | 判定 | 关键 Edge 可回溯性 |
|---|---|---|
| Control（ingress→FSM→invoker→effect） | PASS | invocation.rs:96 → fsm.rs → mod.rs:414/464/729/827 |
| State（keyed 读写权限） | PASS | invocation/mod.rs:157/172 + state_table |
| Data（journal 生产/消费/恢复） | PASS | mod.rs:4968 + journal_table_v2 |
| Evidence（日志→applied_lsn→快照） | PASS | bifrost.rs + fsm_table + snapshot_task.rs |
| Authority（队列+锁+Precondition） | PASS | util.rs + lock_table + metadata.rs:55 |
| Memory（状态表全景） | PASS | partition-store 表清单 |
| Policy（治理闭环） | PASS | AGENTS.md + CI + 状态机 + features/ |

## 4. Abstraction Auditor（L3/L4 是否过度升维）
| KO | 判定 | 论证 |
|---|---|---|
| KO-01 步骤持久化 | PASS | 跨 4 个独立子系统验证（types/worker/partition-store），簇成立五条件满足 |
| KO-02 幂等第一公民 | PASS | 幂等分支出现在状态机核心路径，非外围过滤 |
| KO-03 幂等分片不变量 | PASS | 不变量被注释明确声明（S2）+实现（S3） |
| KO-04 挂起/恢复 | PASS | 完整 causal 链存在 |
| KO-05 组合 future 树 | PASS | 组合子类型真实存在于 unresolved_future.rs |
| KO-06 keyed 无锁串行 | PASS | vqueue 分派 + 锁双机制 |
| KO-07 日志与状态分离 | PASS | Bifrost + fsm_table 分离明确 |
| KO-08 版本化元数据 | PASS | Precondition + MetadataManager |
| **KO-09 迁移即一等公民** | **DOWNGRADE CANDIDATE→保留 L4 但降级措辞** | 滚动迁移的运行时行为未验证（无 S5），措辞已收窄为"存储 schema 演化不该停机 + feature 标记驱动"，不宣称"全分区原子迁移已验证" |

## 5. Counterexample Hunter（反例攻击）
| KO | 反例 | 结果 |
|---|---|---|
| KO-01 | 是否每步都写 journal？→ Input 在 pre-flight 写、Output 在完成写，中间 effect 每步写；无"批量合并步骤"证据 | 不破坏 KO-01（步骤级粒度成立） |
| KO-01 | 崩溃恢复是否真从"最后完成 entry"？→ append 无 gap 断言保证连续性 | 支持（debug_assert 即不变量声明） |
| KO-02 | 有无绕过幂等的路径？→ workflow 忽略幂等 key（返回错误而非去重）——这是有意特例 | 收窄 KO-02 措辞：幂等覆盖普通调用与 workflow 走唯一标识，非"所有调用" |
| KO-03 | 幂等分片是否总是确定性？→ CONTROLLED_IDEMPOTENT_SHARDING 开关存在，关闭时走普通 HashPartitioner(idempotency_key)——两条路径并存 | 收窄 KO-03：默认受控分片下成立；开关存在为迁移期兼容 |
| KO-04 | 挂起是否总是先解析？→ 不能解析才挂起；panic 分支（vqueue entry 不存在）是异常路径 | 不破坏；异常路径在 C-3 记录 |
| KO-06 | 共享 service 队列是否无锁？→ shared handler 共享 VQueueId，队列内串行但不同 key 共享队列=吞吐瓶颈 | 不破坏 KO-06（串行保证成立）；瓶颈入 C-5 |
| KO-07 | 日志是否总是权威？→ LocalLoglet 与 ReplicatedLoglet 并存，单机模式下日志=本地 RocksDB | 不破坏（两种实现都是日志抽象） |
| KO-08 | Precondition 是否总被用？→ None 分支存在（无条件 put） | 收窄：版本化是可选前置，非强制 |
| KO-09 | 迁移是否停机？→ migrations/ 为在线迁移函数，features/ 标记持久化 | 支持；但运行时行为未验证（S3 上限） |

## 6. Epistemic Auditor（认知状态是否诚实）
- 所有"运行时行为"断言标 Observation/S3-（无 Docker 编译运行）。
- KO-09 措辞已收窄（见 Abstraction Auditor）。
- C-1~C-6 全部保持 Hypothesis，未冒充 Fact。
- 跨项目结论（KO-01~09 的通用化部分、C-5/C-6）全部标 `Cross-project validation pending`。

## 7. Contradictions & Counterexamples（保留记录）
1. **license 修正**：snapshot 初记 Apache-2.0 → 实际 BSL 1.1（已修正 snapshot + 本 package 全部引用）。
2. **InvocationTargetType 表述差异**（C-1，已解决）：IVR F-3 确认 ingress 侧 `path_parsing.rs:52` 为独立 `pub(crate)` 枚举（InvocationId/IdempotencyId），与 types 三变体枚举是**名字撞车非冲突**；已同步更新 05_candidates C-1 为 Fact。
3. **幂等路径双轨**（KO-03 反例）：受控分片开关存在，默认路径与兼容路径并存。
4. **KO-09 升维降级**：从"迁移机制模式"降为"迁移即一等公民"候选（措辞收窄）。

## 8. 质量指标
| 指标 | 值 |
|---|---|
| EK 总数 | 44 |
| EK 带 links | 44/44（100%） |
| 孤立 EK | 0 |
| KO 总数 | 9 |
| KO 带 aggregation_rule | 9/9（100%） |
| KO 平均簇规模 | ~4.4 EK |
| 反例攻击 | 9 KO × ≥1（含 3 处措辞收窄） |
| 降级 | 1（KO-09 措辞收窄） |
| 审计判定 | 6/6 完成；Truth 全 CONFIRMED；Coverage 2 gap→Candidates |
| 盲重建 | 独立重读 5 个核心 crate 后对照 |

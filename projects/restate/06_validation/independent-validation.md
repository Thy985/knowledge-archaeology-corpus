# Independent Validation Report — ARCH-2026-11-01-001（Restate）

> **盲重建声明**：本报告由独立 Auditor 完成——不把 Package 00-05 当事实源，独立重读 repo 关键符号（journal_v2 / identifiers / state_machine / vqueues / bifrost / path_parsing / LICENSE）后重新归纳，再与 Package 对照。判定统计见文末。

## 1. 独立 Findings（盲重建产物）

### F-1 journal_v2 Command 枚举 = 20 变体
- 独立计数：`crates/types/src/journal_v2/command.rs:50-71` 逐变体数 = 20（Input/Output/GetLazyState/SetState/ClearState/ClearAllState/GetLazyStateKeys/GetEagerState/GetEagerStateKeys/GetPromise/PeekPromise/CompletePromise/Sleep/Call/OneWayCall/SendSignal/Run/AttachInvocation/GetInvocationOutput/CompleteAwakeable）
- **Package 对照**：EK-01 称 20 种 Command → CONFIRMED

### F-2 InvocationStatus 枚举 = 7 变体 + Free（8 态）
- 独立计数：`crates/storage-api/src/invocation_status_table/mod.rs:144-157` = Scheduled/Inboxed/Invoked/Suspended{metadata,awaiting_on}/Paused/Completed/Free(default) = 7 显式 + Free = 8
- **Package 对照**：snapshot/01 称 8 态 → CONFIRMED

### F-3 **C-1 澄清（核心争议解决）**：InvocationTargetType 是名字撞车，非 types 枚举缺变体
- 独立证据：ingress 侧 `crates/ingress-http/src/handler/path_parsing.rs:52-60` 定义**独立枚举** `InvocationTargetType { InvocationId(String), IdempotencyId{name,target,handler,idempotency_id} }`——这是 URL 路径解析专用类型；`crates/types/src/invocation/mod.rs:148` 的 `InvocationTargetType { Service, VirtualObject, Workflow }` 是执行语义类型。
- 两类型同名但完全独立（不同 crate、不同用途、不同变体集）。
- **Package 对照**：05_candidates C-1 曾标"表述差异待审计" → **升级为 CONFIRMED 澄清**：ingress 侧为路径解析专用枚举，types 侧三变体无误。**05_candidates C-1 需从 Hypothesis 改为 Fact 记录**（Reconciliation 处理）。

### F-4 append 原子性（三写 + 无 gap 断言）
- 独立重读 `mod.rs:4968-5010`：`put_journal_entry(...)` → `get_journal_metadata_mut()` → `debug_assert_eq!(journal_meta.length, entry_index, "journal should not have gaps")` → `journal_meta.length = entry_index + 1` → `put_invocation_status(...)`（同函数体内）
- **Package 对照**：EK-02 称"同事务三写 + 无 gap 断言" → CONFIRMED（注：debug_assert 为开发不变量，S4 测试部分覆盖）

### F-5 handle_duplicated_requests 分派语义
- 独立重读 `mod.rs:1076-1160`：
  - `is_workflow_run && has_idempotency_key` → `warn!("The idempotency key for workflow methods is ignored!")` + 置 false
  - `previous_invocation_status == Free` → 新调用（返回 Some）
  - workflow duplicate → `send_response_to_sinks(..., WORKFLOW_ALREADY_INVOKED_INVOCATION_ERROR, ...)`
  - 其他 duplicate（Invoked/Suspended/Paused/Inboxed/Scheduled）→ `do_append_response_sink`（去重 contains 检查）
- **Package 对照**：EK-04/07 描述准确 → CONFIRMED（workflow 特例 = 不 append sink 直接返回错误，验证 EK-07 与 KO-02 反例收窄）

### F-6 VQueue hash 模板
- 独立重读 `crates/vqueues/src/util.rs:30-60 generate_vqueue_id` + `:102-127 infer_vqueue_id_from_invocation`：模板注释原文 `"<partition-key> || ty || len(scope) || len(limit-key) || <exclusive/shared?> || service || key"`；SHARED_HANDLER=b"0"、EXCLUSIVE_HANDLER=b"1"；Service→shared(false,无 key)、VirtualObject Exclusive→exclusive(true,key)
- **Package 对照**：EK-17 描述一致 → CONFIRMED

### F-7 幂等分片常量
- 独立证据：`identifiers.rs:246 UNSCOPED_SERVICE_PARTITION_KEY_FANOUT: u64 = 512`；`:248 static CONTROLLED_IDEMPOTENT_SHARDING: AtomicBool = AtomicBool::new(false)`；`:256 set_controlled_idempotent_sharding(true)`；`:274 debug_assert!(bucket < 512)`
- **Package 对照**：EK-10/11 + KO-03 描述一致（含"开关存在、默认关"） → CONFIRMED（KO-03 反例收窄正确）

### F-8 LICENSE = BSL 1.1
- 独立证据：`LICENSE` 头部 "Business Source License 1.1"；Licensor: Restate Software, Inc., Restate GmbH；`:38 Change Date: 4 years after release`；`:40 Change License: Apache License, Version 2.0`
- **Package 对照**：snapshot 已修正 + 00/01/run_metadata 均写 BSL 1.1 → CONFIRMED（快照修正正确）

### F-9 .rs 规模
- 独立实测：`find -name '*.rs' -not -path '*/target/*' | wc -l` = 1271；`-exec cat {} + | wc -l` = 367514
- **Package 对照**：snapshot 1271/367,514 → CONFIRMED（IVR 计数攻击通过）

### F-10 bilrost Oneof 版本兼容
- 独立证据：`types/src/invocation/mod.rs:143-147` 注释原文 "Do not add a variant to InvocationTargetType without skipping a version first. This is a `bilrost::Oneof`, and a node running an older version decodes an unrecognized tag as `Self::Service` instead of failing"
- **Package 对照**：EK-38 引用准确 → CONFIRMED

## 2. 判定统计
| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 9 | F-1/F-2/F-4/F-5/F-6/F-7/F-8/F-9/F-10 |
| PARTIALLY_CONFIRMED | 1 | F-3（结论确认，但 Package 05 的 C-1 表述需从 Hypothesis 更新为 Fact） |
| DOWNGRADED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| MISSING | 0 | — |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 3. 3 个成功项（Succ）
1. **F-5 workflow 幂等特例**：独立确认 workflow duplicate 不 append sink 而是直接返回 `WORKFLOW_ALREADY_INVOKED_INVOCATION_ERROR`——验证 Package 对 KO-02 的收窄措辞正确（幂等非"所有调用"，workflow 走唯一标识特例）。
2. **F-6 VQueue hash 模板**：模板注释与实现逐字段对上（partition_key/ty/len(scope)/len(limit-key)/exclusive|shared/service/key），确认 EK-17 无虚构。
3. **F-8 BSL license**：独立读取 LICENSE 确认 BSL 1.1 + Change Date/Change License 条款，修复了 snapshot 初记 Apache-2.0 的错误。

## 4. 3 个错误尝试/陷阱（Err）
1. **计数陷阱**：初查 journal_v2 Command 用 `grep -c "pub struct"` 得 27（含辅助结构），改用逐变体枚举计数才得 20——若用前者会把 EK-01 误判为错。教训：计数类断言必须数枚举变体而非 struct 数。
2. **C-1 陷阱**：初看 ingress invocation.rs:96 出现 `InvocationTargetType::IdempotencyId` 怀疑 types 枚举缺变体——追到 path_parsing.rs:52 才发现是独立枚举（crate 内私有 `pub(crate)`），非同一类型。若停止在 invocation.rs 会得出错误结论。
3. **流程陷阱**：尝试直接验证 `CONTROLLED_IDEMPOTENT_SHARDING` 行为时发现它是 AtomicBool 静态开关（默认 false），不能按"始终生效"理解——需按"默认关、可开"表述（KO-03 已收窄）。

## 5. 遗漏（Miss / Gap）
- 无强制遗漏；覆盖 2 个 Gap 已在 Package 05 列为 C-4（跨分区锁）、C-6（ReplicatedLoglet 读一致性），本轮静态考古未展开运行时验证。

## 6. Benchmark Case（参照 agent-substrate 对照）
- **对照维度**：步骤持久化派（Restate journal_v2）vs 挂起恢复派（agent-substrate suspend/resume）。
- **独立观察**：Restate 的 suspend 也写 SuspendedEvent（journal_events）并持久化 awaiting_on——说明"挂起恢复派"与"步骤持久化派"在真实实现中不是互斥：Restate 主机制是 journal，suspend 是等待条件（promise/signal/timer）时节省资源的手段，awaiting_on 本身也被持久化。
- **Package 对照**：KO-01/KO-04 并列描述两条路线，未宣称互斥 → CONFIRMED（该对照在 00_overview/03 层表述准确）。

## 7. Reconciliation 要求（不改原考古产物，仅记录需修正项）
| 项 | 修正动作 | 归属 |
|---|---|---|
| R-1 | 05_candidates C-1 从 Hypothesis 升级为 Fact 记录（两处同名枚举已澄清：ingress path_parsing.rs:52 独立枚举 vs types invocation/mod.rs:148 三变体） | corpus 05_candidates |
| R-2 | 06_validation §1 C-1 表述更新为 CONFIRMED 澄清（原为"未解决保留攻击点"） | corpus 06_validation |
| R-3 | 无其他修正（F-1~F-10 全部 CONFIRMED，无 DOWNGRADE/CONTRADICT） | — |

> 注：按 SOP，原 package 产物只读不修改；上述修正将在阶段⑥ Reconciliation 时写入 corpus 副本（corpus 为可写归档，生产 skill 与本地 package 不改）。

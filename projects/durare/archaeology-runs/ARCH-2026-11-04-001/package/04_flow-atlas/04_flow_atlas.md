# 04 Flow Atlas — durare（七类流）

> 从真实代码导出，关键 Edge 可回溯到 symbol/file/condition。每条 Edge 标注来源文件。

## 4.1 Control Flow（控制流：谁调用谁，谁触发谁）

```
Client.enqueue (client.rs)                    DurableEngine.start (engine.rs)
   │                                              │
   ▼                                              ▼ insert_workflow_status（created flag）
insert workflow_status (ENQUEUED/DELAYED)         │
   │                                              ▼
   ▼                                        direct run → runtime.dispatch（INTERNAL_QUEUE 或用户队列）
dispatcher loop（每 queue，engine.rs launch）      │
   │ dequeue_workflows (ENQUEUED→PENDING)         ▼
   ▼                                        Execution::scope（execution.rs）
workflow body（DurableContext）                    │
   │ step/transaction/select/...                  ▼
   ▼                                        durable-call 协议：
run_transaction_step / record_step_result    check_placement → claim_position →
（postgres.rs）                                replay 记录 or 执行 body → checkpoint
   │                                              │
   ▼                                              ▼
settlement loop（execution.rs seam）          recover() → recover_pending_workflows
   │ RecoveryRequired → backoff 100ms*2→5s        │ claim_for_recovery (CAS) → task tracker spawn
   ▼                                              ▼
re-dispatch（或 park at cap）                resume run（新 Execution，同一 workflow_id）
```

**关键 Edge**：
- E1: `DurableEngine::launch` → spawn dispatcher per queue + schedule loop per cron（engine.rs:1497）
- E2: workflow body 每次 durable call → `Execution::check_placement`（execution.rs:60）→ `claim_position`（context.rs:660）
- E3: 恢复 run 走 `claim_for_recovery` 的 CAS 三分支（postgres.rs:2472）
- E4: `Error::RecoveryRequired` 从 execution.rs seam 向上 → runtime settlement loop

## 4.2 State Flow（状态流：状态机）

```
              ┌──────────────────────────────────────────────┐
              │                                              │
              ▼                                              │
        ENQUEUED ──dequeue_workflows──▶ PENDING ──terminal──▶ SUCCESS
           │                              │     ──terminal──▶ ERROR
           │                              │
           │           recovery CAS       │ claim_for_recovery 超 cap
           ▼                              ▼
        DELAYED ──transition_delayed──▶ ENQUEUED        MAX_RECOVERY_ATTEMPTS_EXCEEDED (park)
        （delay_until_epoch_ms 到点）          ▲                    │
                                            └──── resume 显式 ─────┘
                                    
        （operator）──────────────────────────▶ CANCELLED（terminal，replay 拒绝）
```

**关键 Edge**：
- E5: `transition_delayed_workflows(now_ms)` 只迁到期的 DELAYED 行（provider.rs:1672）
- E6: `dequeue_workflows` 条件更新 `ENQUEUED→PENDING`（只有一个 dispatcher 赢，postgres.rs:1638）
- E7: 终态写只从 PENDING 出发；第二写者 ON CONFLICT DO NOTHING → WorkflowConflict（execution-layers.md "One recorded outcome"）
- E8: `Error::Cancelled` 是控制流不 checkpoint（CHANGELOG 0.5.0）

## 4.3 Data Flow（数据流：值如何流动）

```
输入 (Value/JSON) ──serializer.encode（DBOS_JSON base64 | portable_json plain）
      │                                   │
      ▼                                   ▼
workflow_status.inputs  ←────────  serialization 列（格式名）+ TEXT
      │
      ▼  step body 输出
operation_outputs（output | error + serialization 列）
      │
      ▼  replay 时
读 operation_outputs → format-directed decode（按存储格式名，非当前配置）
      │
      ▼  child
operation_outputs.child_workflow_id → 子 workflow_status 行
      │
      ▼  messaging
notifications（send/recv FIFO，exactly-once 消费）
workflow_events（key→latest，last write wins）
streams（write_stream + STREAM_CLOSED_SENTINEL）
```

**关键 Edge**：
- E9: 编码用 provider 配置的 serializer，**解码按存储 format name 分派**（serialize.rs:1-25）
- E10: 时间戳全 BIGINT epoch ms，非 SQL timestamp（schema.md）
- E11: `authenticated_roles` 单列 JSON 数组（encode_roles/decode_roles，provider.rs）

## 4.4 Evidence Flow（证据流：如何证明系统正确）

```
源码实现（S3）───▶ 集成测试断言（S4）
   │                    │
   │                    ├─ crash_consistency.rs：7 边界 × 精确 effect 计数
   │                    ├─ adversity.rs：多 executor 竞争 × exactly-once
   │                    ├─ interop.rs：golden strings × 跨 SDK byte-identical
   │                    ├─ security.rs：EVIL sweep × verbatim round-trip
   │                    └─ child_atomic.rs：contract(db) 三后端同跑
   │
   ├─ CHANGELOG（0.5.0/0.4.0/Unreleased）：语义变更记录
   ├─ docs/design/*.md：设计意图（≠实现事实，doc-analyst 标记）
   └─ CI（ci.yml/bench.yml/security.yml）：四门 + semver + audit/deny
```

**关键 Edge**：
- E12: 设计文档（doc）与实现（code）分离——explicit-context.md 的"官方 SDK 对照表"是**文档声明**，未在测试中直接验证（Validator 将标记）
- E13: failpoints（fail crate）使 bug 注入在测试/example 激活——崩溃是可编程的

## 4.5 Authority Flow（权威流：谁有权做什么）

```
调用方 start/enqueue 时提供 AuthContext
   │  authenticated_user / assumed_role / authenticated_roles
   ▼
engine start 前 check（require_roles 声明式门槛）───▶ NotAuthorized 拒绝
   │
   ▼
AuthContext 流入：workflow body / children（identity per field 继承）/ trace spans
   │
   ▼
持久化：workflow_status.authenticated_user / assumed_role / authenticated_roles（JSON 数组）
   │
   ▼
（读取方）按需解码 authenticated_roles（跨 SDK shape）
```

**关键 Edge**：
- E14: `require_roles` 在 **start 之前**检查（engine.rs）；launch 时拒未注册 workflow 的 roles 声明（"typo that would silently enforce nothing"）
- E15: AdminServer **无认证** by design（security.rs）——权限边界交给网络层（loopback/firewall）
- E16: secrets：db URL 不 retain；ConductorConfig 无 Debug（compile_fail doctest）

## 4.6 Memory Flow（记忆流：什么是持久的）

```
进程内（非持久，重启丢失）：
  - Engine registry（register 的 workflow/queue 表）
  - Execution latch（OnceLock failure + Notify）——per-run，崩溃即失
  - task-local CURRENT_EXECUTION——scope 内有效
  - TaskTracker 计数——spawn 瞬间计数

持久（数据库行，重启/崩溃保留）：
  - workflow_status（每 workflow 一行，含 checkpoint 指针）
  - operation_outputs（每 position 一行 = 记忆单元）
  - notifications / workflow_events / workflow_events_history / streams
  - workflow_schedules（调度）
  - queues（fleet 注册表）
  - application_versions（版本注册表）
```

**关键 Edge**：
- E17: "记忆单元" = operation_outputs 行：(workflow_uuid, function_id) 唯一 → 重放记忆 = 按 position 查行（EK-08）
- E18: 恢复 = 读 PENDING 行 + 按已有 checkpoint 重放——不重新执行已完成 step（EK-32）
- E19: 内部队列/调度 reconciler 把"需要继续的事"记忆在 DB 行，任何 live executor 可接（EK-18/19）

## 4.7 Policy Flow（策略流：规则如何固化）

```
决策（doc）───▶ 规则（AGENTS.md / docs）───▶ 固化（代码/测试/CI）───▶ 强制（外部执行）───▶ 未来决策
   │                │                              │                     │
   │                │                              │                     │
 语义对齐纪律    "replay makes exactly         决策表：新行为归层        security.rs hostile
 (AGENTS.md)     the decisions..."            (execution-layers.md)    sweep：外部强制
   │                │                              │                     │
   │                │                              ▼                     ▼
   │           determinism.rs 规则书          require_roles launch       CI 四门 + audit/deny
   │                │                              │ 拒未注册              │
   │                ▼                              ▼                     ▼
   └───▶ 新行为先写规则（condition→behavior）   版本门控（app_version）   golden strings
                再实现（AGENTS.md 强制）        恢复/认领只归同版本        interop 外部强制
```

**关键 Edge**：
- E20: Policy 闭环：AGENTS.md 规定"先写规则再实现"（condition → durable behavior）→ 实现（execution-layers.md 决策表）→ 测试强制（crash/interop/security）→ 未来决策引用（文档/CHANGELOG 记录为何如此）
- E21: 版本门控是**部署策略**的固化：语义不兼容 = 版本变化，两版本并存安全（EK-16/17）
- E22: 外部强制偏好（"enforced from the outside"）本身就是治理策略：源码可能被重构腐化，外部测试不会静默失效

---

## Flow 质量自检

| 流 | Edge 数 | 全部可回溯 symbol/file | Flow→KO 门 |
|---|---|---|---|
| Control | 4 | ✓ | KO-02 |
| State | 4 | ✓ | KO-04/KO-07 |
| Data | 3 | ✓ | KO-06 |
| Evidence | 3 | ✓ | KO-10 |
| Authority | 3 | ✓ | KO-09 |
| Memory | 3 | ✓ | KO-01/KO-05 |
| Policy | 3 | ✓ | KO-08/KO-07 |

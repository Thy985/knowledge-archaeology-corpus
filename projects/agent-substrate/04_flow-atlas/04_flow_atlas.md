# 04 · Flow Atlas — Agent Substrate 七类流

> 从真实代码/文档导出，每条关键 Edge 可回溯 symbol/file/condition。

## F1 Control Flow（控制流）——actor 生命周期
```
客户端/Envoy Router ──ResumeActor──▶ ateapi Control
   │                                   │ acquireActorLease（分布式租约）
   │                                   ▼
   │                    loadActorForResume → ensureVolumesCreated → ensureWorkerAssigned
   │                                   │（状态推进，版本前提持久化，冲突→Aborted）
   │                                   ▼
   │                    ensureAteletRestored ──▶ atelet（恢复快照到 worker）
   │                                   ▼
   │                    finalizeRunning（state=RUNNING, workerEpoch）
   ◀─────────── 返回 actor ───────────┘
```
- 证据：workflow_resume.go ResumeActor 主流程；请求停车文档
- Key Edges：
  - `ResumeActor → acquireActorLease`（workflow_resume.go L64）
  - `ensureWorkerAssigned → ResourceExhausted`（无 worker 可容纳）
  - `ensureAteletRestored → atelet`（ateletpb）

## F2 State Flow（状态流）——ActorState 状态机
```
(创建) → RESUMING → RUNNING ⇄ SUSPENDING → SUSPENDED → RESUMING → …
                        │                  │
                        ▼                  ▼
                      CRASHED ←────────── 崩溃报告（ReportActorCrashed）
                        │
                        ▼
                      REVERTING → SUSPENDED（清 crash 记录）
                        │
                        ▼
                      DELETING →（删除）
```
- 证据：ateapi.proto ActorState 枚举 + ActorStatus 注释（crash set with CRASHED, cleared when revert returns to SUSPENDED；assigned_node 挂起仍附着直到 detach/revert）
- Worker 状态：UNAVAILABLE（注册初态）→ MarkWorkerAvailable → ACTIVE → DrainWorker → DRAINING（终态，永不复原）
- 证据：ateapi.proto WorkerState 注释

## F3 Data Flow（数据流）——suspend/resume 快照数据
```
worker 内存 ──CheckpointWorkload──▶ snapshot files（worker 本地，checkpointDir）
    │                                │
    │  .checkpoint-done.json         │（BootID/Fidelity/文件清单）
    │                                ▼
    │                    atelet（双重授权：actor+atelet）──上传──▶ 对象存储（GCS/S3）
    ▼
ateapi（状态机：SUSPENDING→SUSPENDED, last_assigned_generation++）
resume: 对象存储 ──atelet 下载──▶ worker 本地快照 ──restore──▶ 内存
```
- 证据：checkpoint.go（checkpointRecord/RecordCheckpoint/ReuseCheckpoint）；security.md Dual-Party Snapshot Authorization；objectstoreplugin/exchange
- Key Edge：BootID 相同 + fidelity 匹配 + 文件齐全 → ReuseCheckpoint 可复用（checkpoint.go）

## F4 Evidence Flow（证据流）——测试与遥测
```
e2e（internal/e2e: backend/env/fixture/pod/manifest）
   │ 双运行时覆盖（gVisor+microVM，AGENTS.md 要求）
   ▼
CI（pr-workflow.yaml + govulncheck）
   ▼
运行时遥测：OTLP → lifecycle metrics（recordLifecycleOp 跳过已运行 no-op）
   ▼
Structured Audit Trail（生命周期/快照/网关事件，凭据脱敏）
```
- 证据：internal/e2e 文件；AGENTS.md；workflow_resume.go restoreTelemetry 注释；security.md Auditability

## F5 Authority Flow（权威流）——身份与授权
```
ateapi ──MintActorJwt/MintWorkerActorCertificate──▶ 短期 X.509/JWT
   │
   ▼
组件 mTLS（ateapi/atelet/ateworker/routers/tunnels，仅 /healthz /readyz 豁免）
   │
   ▼
ingress: mTLS CONNECT 证书 → filter state（Envoy 派生，隧道内不可写）
   │
   ▼
OpenFGA ReBAC Check（Authorizer，bootstrapOwners=contextual tuple）
```
- 证据：security.md Identity/组件表；egress.go 包注释；authorizer.go；atunnel/tunnel.go

## F6 Memory Flow（记忆流）——沙箱内存与快照
```
actor 内存（gVisor/microVM sandbox）
   │ cgroup memwatch/memguard（接近限额 → 主动 suspend 信号）
   ▼
suspend → checkpoint → 内存快照（snapshot files）→ 上传对象存储
   │
   ▼
空闲 actor 完全出内存（SUSPENDED 无 worker_assignment）
resume → 下载 → restore → 内存回沙箱
```
- 证据：ateworker/cgroup/（memwatch/memguard）；checkpoint.go；ActorStatus 注释（SUSPENDED 时 worker_assignment unset）

## F7 Policy Flow（策略流）——治理闭环 Decision→Approval→Policy→Enforcement→Future Decision
```
治理文档（GOVERNANCE.md 角色/晋升）→ 贡献决策（PR/DCO/AGENTS.md 规则）
   │
   ▼
EgressPolicy（ateapi 验证 + 网关匹配同一 parser）→ egress PEP（fail-closed）
   │
   ▼
Enforcement：egress 网关（SNI 规则/凭据注入/403-503 映射） + NetworkPolicies（ate-system 默认拒绝）
   │
   ▼
审计：Structured Audit Trail → 动态隔离（Dynamic Quarantine）→ 未来决策（改善策略）
   ▼
AI 协作治理：AGENTS.md（禁 AI 味词、agent 不 sign-off DCO、PR 单改动）
```
- 证据：GOVERNANCE.md/AGENTS.md；egresspolicy.go 单 parser 复用；security.md Network/Auditability；credentials.go 错误映射
- Key Edge：策略编译（Compile）与验证共用同一 parser——"两处不能漂移"（egresspolicy.go 包注释）

## Flow→KO 交叉校验
| KO | 对应 Flow | 校验 |
|---|---|---|
| KO-01 | F1/F2 | ResumeActor 幂等推进与状态机一致 |
| KO-02 | F7 | 重试分类与 egress 策略错误映射一致 |
| KO-03 | F1 | parking 预算与 ResumeActor 认领路径一致 |
| KO-04 | F5 | 零凭据与权威流一致 |
| KO-05 | F3 | 双重授权与快照数据流一致 |
| KO-06 | F7 | 文档纪律与治理流一致 |
| KO-07 | F2/F6 | 生命周期两条路线与状态/记忆流一致 |

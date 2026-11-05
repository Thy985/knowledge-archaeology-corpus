# Flow Atlas — 七类流（Control / State / Data / Evidence / Authority / Memory / Policy）

> 每条 Edge 具备 `symbol`（函数/类型）、触发条件、失败路径；可回溯到 `repo` 文件行号或 E 编号。标注 `[S3]` = 代码可见，`[S4]` = 有测试，`[D]` = 设计意图。

## 1. Control Flow（控制流）

主链：**事件/定时 → Reconcile → 门控 → 状态计算 → phase 分派 → Ensure → 写状态**。

| Edge | From → To | Symbol / File | 条件 | 失败路径 |
|---|---|---|---|---|
| C-01 | informer event / requeue → Reconcile | `SandboxReconciler.Reconcile` `sandbox_controller.go:233` | 任意 Sandbox 事件或 requeue | 取 Sandbox NotFound → 清理 metrics/expectations 后返回 `[S3]` |
| C-02 | Reconcile → expectation gate | `ScaleExpectation.ObserveScale` / `SatisfiedExpectations` `:274-289` | 创建后等待观察 | 超时 → `DeleteExpectations` 继续 `[S3]` |
| C-03 | Reconcile → cache gate | `ResourceVersionExpectations` `:291-307` | 本地缓存落后 | 不计算，`staleCacheRequeueInterval` 重试 `[S3]` |
| C-04 | Reconcile → rate-limit | `SandboxCreatePodRateLimitGate` + `IsHighPrioritySandbox` `:310-318` | gate on 且创建 Pod | 限流 defer，延迟创建 `[S3]` |
| C-05 | Reconcile → volume ensure | `ensureVolumeClaimTemplates` `:1070` | Pending/Running | 返回 err，进入 status 更新 `[S3]` |
| C-06 | Reconcile → terminate path | `handleTerminating` `:670` | DeletionTimestamp 非空 | 清理后返回 `[S3]` |
| C-07 | Reconcile → timer check | `checkTimers` `:1130` | 每轮 | 到期执行 pause/shutdown `[S3]` |
| C-08 | Reconcile → status calc | `calculateStatus` `:819` | 非终态 | shouldRequeue 时带 requeueAfter `[S3]` |
| C-09 | status calc → phase dispatch | `switch newStatus.Phase` `:403-455` | Phase 已知 | default：记录 invalid phase + requeue `[S3]` |
| C-10 | phase dispatch → Ensure* | `EnsureSandboxRunning/Updated/Paused/Resumed/Upgraded/Recycled` | 对应 Phase | err → 先落 Upgrade 状态再返回 `:461-466` `[S3]` |
| C-11 | Ensure done → auto-pause | `autoPauseTakesOver` → `handleAutoPause` `:467-478` | gate + policy | 非激活 → 跳过 `[S3]` |
| C-12 | auto-pause → status write | `updateSandboxStatus` `:713` | 每轮 | 冲突 → 返回 err 重试 `[S3]` |
| C-13 | status write → requeue | `requeueAfter` `auto_pause.go:573` | 有未来时间 | 无 → 等事件 `[S3]` |

**失败路径汇总**：①缓存落后 → 短重试；②Ensure err → 状态先落盘；③未知 phase → 记录并 requeue；④限流 → 延迟创建；⑤删除中 → finalizer 清理。

## 2. State Flow（状态流）

```
Pending ──EnsureRunning──> Running
Running ──Spec.Paused=true / PauseTime──> Paused
Paused  ──Spec.Paused=false / ResumeTime──> Resuming
Resuming ──EnsureResumed 成功──> Running
Running/Paused ──升级策略──> Upgrading ──各阶段──> Resuming ──> Running
Running ──recycle 条件──> Recycling ──> Pending(池)
任意 ──DeletionTimestamp──> Terminating ──> (删除)
Running ──Pod 终止(全成功)──> Succeeded
Running ──Pod 终止(有失败)──> Failed
```

| Edge | 触发条件（代码） | 副作用 | 失败/旁路 |
|---|---|---|---|
| S-01 Pending→Running | `EnsureSandboxRunning`（Pod 创建/启动成功） | Ready=True | Pod 创建失败 → 保持 Pending `[S3]` |
| S-02 Running→Paused | `handleAutoPause` 或 `checkTimers` 的 PauseTime，最终 `Spec.Paused=true` | PrePause → 删 Pod → Paused=True | hook 失败重试；Pod 删除中等待 `[S3]` |
| S-03 Paused→Resuming | `Spec.Paused=false` 由用户/`tryResume` patch | 进入恢复 | condition 非 True 时 `tryResume` 等待 `[S3]` |
| S-04 Resuming→Running | `EnsureSandboxResumed` + `markResumeSucceeded` | Paused cond 移除、Resumed=True、runtime 重置 | 容器不一致 → 继续 Resuming `[S3]` |
| S-05→Upgrading | `EnsureSandboxUpdated` 检测模板 revision 变化且策略非空 | 进入 Upgrade 状态机 | 空策略 → 不升级 `[S3]` |
| S-06 Upgrading→Resuming | `EnsureSandboxUpgraded` 完成 Pod 替换/恢复 | reason=ResumeSucceed | PreUpgrade 失败 → reason 记录但继续 `[S3]` |
| S-07 任意→Terminating | DeletionTimestamp | finalizer 清理 | 无 finalizer → 直接删除 `[S3]` |
| S-08 运行→Recycling | 池回收条件（recycle controller） | 重置为池对象 | manager 部署不识别 recycle `[D]`（E-06） |
| S-09→Succeeded/Failed | Pod phase 终止 | 终态短路 | 终态不再 Ensure `[S3]` |

**Condition 子流**：Ready / Paused / Resumed / InplaceUpdate / Upgrading / RuntimeInitialized / Recycling / ProbeValid / PrePauseHook / PostResumeHook / `agents.kruise.io/<probe>`。
- 每次 `merge-patch` 更新 condition，`LastTransitionTime` 仅在状态变化时刷新（`SetSandboxCondition` DeepEqual 短路；`ensureStopPaused` 注释解释为何新建 condition 而非原地改）。

## 3. Data Flow（数据流）

| Edge | 载体 | From → To | 关键约束 |
|---|---|---|---|
| D-01 | Sandbox CR | apiserver → controller informer cache | 读纪律：List 禁用 APIReader；Get 需理由（AGENTS）`[D]` |
| D-02 | Sandbox CR | controller → status subresource | 每轮 merge patch；condition 按 type 合并 `[S3]` |
| D-03 | Checkpoint CR | controller → apiserver | label 内容派生；ownerRef 级联 `[S3]` |
| D-04 | Checkpoint status | controller → resume/clone | `PodTemplateDelta` 恢复补丁 `[S3]` |
| D-05 | SandboxSet/SandboxTemplate revision | controller sandboxset → pool | 不可变字段清单（卷/持久化/runtime/probe）`[S4]` |
| D-06 | SandboxClaim | manager → controller | claim 绑定 owner；失败回滚 `[S3]` |
| D-07 | E2B API 请求 | gateway/servers → manager → CRD | 协议映射（未逐行考古，C-07）`[D]` |
| D-08 | Probe 结果 | pod probe → Pod.Status.Conditions → Sandbox.Status | 双通道（virtual-kubelet annotation / Kruise PodProbeMarker）`[S3]` |
| D-09 | Quota 状态 | manager → Redis/backend | anti-drift 双向对账 `[S3]` |
| D-10 | Route | gateway registry → peers | 失败仅日志，最终一致 `[S3]` |
| D-11 | Metrics | controller → Prometheus | 默认不新增（AGENTS）`[D]` |

## 4. Evidence Flow（证据流）

| Edge | 产生 | 消费 | 用途 |
|---|---|---|---|
| E-01 | Reconcile span（before: phase + conditions） | tracing backend | 只保留"可能写"的迭代 `[S3]`（过滤纯读） |
| E-02 | status 写入 span | tracing backend | 与 E-01 对比出状态迁移 |
| E-03 | probe condition（`agents.kruise.io/<name>`） | `calculatePauseTime` / `calculateResumeTime` | 自动暂停/恢复判定 `[S3]` |
| E-04 | `Status.Schedules[ProbedIdle/ProbedSchedule]` | 下一轮 Reconcile、tryResume | 决策可观测 + Pod 删除后的恢复依据 `[S3]` |
| E-05 | Kubernetes Event（AutoPaused/AutoResumed/Warning） | 用户/监控 | 只在变化时发，降噪 `[S3]` |
| E-06 | Checkpoint status（phase/message） | `waitForExistingCheckpoint`、升级 | 快照进度证据 `[S3]` |
| E-07 | Upgrading condition reason（PreUpgradeFailed/CheckpointFailed/...） | 下一轮升级 | 失败步骤续跑依据 `[S3]` |
| E-08 | Quota metrics（antiDriftErrors/Skipped） | 运维 | 对账健康度 `[S3]` |

**证据缺口**：`pkg/tracing` 仅读到 span 结构与过滤注释，未验证 exporter 配置；`Evidence Flow` 的 E-01/E-02 依赖 `[S3]` 注释而非运行观测。

## 5. Authority Flow（权威流）

| Edge | 决策者 | 执行者 | 边界 |
|---|---|---|---|
| A-01 | 用户/kubectl | apiserver | 创建/修改 Sandbox、Checkpoint、UpgradePolicy |
| A-02 | controller | Pod/Checkpoint | 仅通过 CRD status 与 Pod 操作；无越权直接改容器 |
| A-03 | manager | quota backend | `TryClaimSandbox` 成功后才分配；leader-only 修复 |
| A-04 | gateway JWT | 请求 | verifier 未 Ready 时拒绝；配置不可变 `[S3]` |
| A-05 | identity provider | runtime | `SecurityIdentityProviderGate` 关闭时随机 UUID 令牌 `[S3]` |
| A-06 | webhook | apiserver | 校验 min/max、patch、security profile `[S3]` |
| A-07 | RBAC | 所有组件 | config/rbac 生成；controller 权限最小化 `[D]` |

**Authority 关键点**：不存在"AI/agent 自行决定是否执行"的路径；所有状态变更经 CRD + controller。Sandbox 内 agent 的运行权限由 Sandbox 隔离 + 身份令牌约束，控制面不直接执行用户代码。

## 6. Memory Flow（记忆/持久化流）

| Edge | 内容 | 载体 | 生命周期 |
|---|---|---|---|
| M-01 | 根文件系统（可写层） | PVC / Checkpoint filesystem | Pause/Upgrade/Recycle `[S3]` |
| M-02 | 内存状态 | Checkpoint memory（可选） | 仅 pause checkpoint；升级不保 `[S3]` |
| M-03 | Pod 运行时差异 | `PodTemplateDelta` | 恢复时 apply `[S3]` |
| M-04 | GPU 显存 | Checkpoint gpuMemory | 实验性/平台相关 `[S3]` |
| M-05 | 沙箱身份/会话 | Sandbox CR + identity CRD | 跨 pause/resume 保留 `[S3]` |
| M-06 | 池预热状态 | SandboxSet 池对象 | claim 前保持 Pending/Running `[S3]` |
| M-07 | 自动暂停决策 | `Status.Schedules` | Pod 删除后仍可读 `[S3]` |

**Memory 边界**：`PauseStrategy=Hibernate` 未接入 → M-02/M-03 在 pause 路径不可达（C-01）；升级 `CheckpointRestore` 只走 M-01。

## 7. Policy Flow（策略/治理解析）

```
Decision (API spec) → Approval (webhook validation) → Policy (feature gates / CRD enums)
   → Enforcement (controller branch) → Future Decision (condition/schedule 反馈)
```

| Edge | 策略输入 | 校验/审批 | 执行点 | 反馈 |
|---|---|---|---|---|
| P-01 | `AutoPausePolicy` | `autopause/validation.go`（规则必填） | `calculatePauseTime/ResumeTime` | ProbeValid / Schedules |
| P-02 | `PauseStrategy.Type` | CRD enum（Stop/Hibernate） | **无消费点**（EK-17/18） | 无 |
| P-03 | `UpgradePolicy.Type` | enum + webhook? | `EnsureCheckpointForUpgrade` / `performRecreateUpgrade` | Upgrading reason |
| P-04 | `PausedRetention` annotation | `resolveRetentionAnnotationOrDefault` | `pauseSandbox` 改写时间 | ShutdownTime/PauseTime |
| P-05 | Feature gates（15） | 启动 flag 校验 | 控制器/manager 分支 | 无（进程级） |
| P-06 | `AGENTS.md` 分层规则 | 人工 review（无自动检查） | 代码组织 | legacy debt 记录（C-05） |
| P-07 | RBAC / network policy / security profile | webhook + CRD | apiserver/gateway | 审计日志（未验证） |

**Policy 缺口（关键）**：
1. P-02 是"有策略、无执行"——Policy Flow 断在 Enforcement。
2. P-06 的 Approval 环节依赖人，不是自动 Gate。
3. `SandboxPauseCheckpointGate` 是历史 gate（注释明确不再消费），属 Policy 残骸。
4. `KruiseIntegrationGate` 关闭时 probe 条件 Unknown → 暂停决策 fail-closed（策略副作用，非 bug）。

## Flow 层结论

- **Control 与 State 一致**：phase switch 的每个分支都能在 State Flow 中找到转移；无"死转移"。
- **Data 与 Memory 的断点**：pause 路径的 Memory Flow 不可达（Hibernate 未接线），升级路径只走 filesystem。
- **Evidence Flow 完整但依赖 probe 投递**：双通道任一失效会退化为 Unknown → fail-closed。
- **Authority Flow 无旁路**：未发现 controller 直接 exec 用户容器的代码路径（`pkg/agent-runtime` 未逐行，属覆盖缺口）。
- **Policy Flow 两处断裂**：PauseStrategy 的 Enforcement 缺失；AGENTS 规则的 Approval 非自动。

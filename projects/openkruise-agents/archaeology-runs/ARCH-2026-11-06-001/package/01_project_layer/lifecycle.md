# 01 Project Layer — Lifecycle & State Machine

## 1. Sandbox 九态

定义：`api/v1alpha1/sandbox_types.go:504-530`。

| Phase | 含义（代码语义） | 进入路径 |
|---|---|---|
| Pending | 尚未有 Running Pod | `calculateStatus` 从无 pod/创建中判定 |
| Running | Pod 可用 | `EnsureSandboxRunning` 成功 / resume 完成 |
| Paused | Pod 已删除且 Paused condition=True | `EnsureSandboxPaused`（Stop 策略删 Pod） |
| Resuming | 正在重建 Pod | `Spec.Paused=false` 后由状态计算进入 |
| Upgrading | 模板变更后的多阶段升级 | `SandboxUpgradePolicy.Type != ""` |
| Succeeded | 终态（成功后不再 reconcile 工作） | Pod Succeeded |
| Failed | 终态 | Pod Failed / 超时 |
| Recycling | 回收（**sandbox-manager 部署禁用**） | controller 独立能力 |
| Terminating | 删除中 | DeletionTimestamp 非零 |

终态短路：`sandbox_controller.go:384-390`（`isSandboxCompletedPhase` → 直接返回，不再 Ensure）。

## 2. Reconcile 主链路（逐步，带行号）

`pkg/controller/sandbox/sandbox_controller.go:233-505`：

1. **取 Pod 与 Sandbox**（:235-262）；Sandbox 不存在时清 expectations、删除创建结果缓存、enqueue metrics 清理。
2. **空模板早退**（:264-272）：`Spec.Template==nil && Spec.TemplateRef==nil` 且未删除 → 直接返回。
3. **ScaleExpectation 门控**（:274-289）：等待 informer 观察到上一次创建，超时后放弃期望。
4. **ResourceVersionExpectation 门控**（:291-307）：缓存落后于上次写入时，**不基于陈旧缓存做状态计算**，而是 `staleCacheRequeueInterval` 短重试（注释给出理由：否则会算出 no-op transition 并把 pending wake 推迟到下次 informer resync）。
5. **Reconcile tracing span**（:320-355）：在第一次可能写资源前开启；条件值以 `before` 属性记录，便于与 status 更新 span 对比。
6. **`ensureVolumeClaimTemplates`**（:357-361）：休眠/唤醒持久卷恢复。
7. **删除路径**（:363-374）：`handleTerminating`。
8. **完成态短路**（:376-379）。
9. **模板哈希注解**（:381-384）：`addSandboxHashAnnotation` 用于 in-place update 检测。
10. **定时器**（:386-390）：`checkTimers` 处理 `ShutdownTime` / `PauseTime`；返回 done 即结束本轮。
11. **状态计算**（:392-399）：`calculateStatus` 产出 newStatus，必要时 requeue。
12. **phase 分派**（:401-455）：
    - Pending → `EnsureSandboxRunning`
    - Running → `EnsureSandboxUpdated`
    - Paused → `preparePausedPhase`（门控）→ `EnsureSandboxPaused`
    - Resuming → `EnsureSandboxResumed`；Resumed=True 时 `finalizeResumePhase`
    - Upgrading → `EnsureSandboxUpgraded`
    - Recycling → `EnsureSandboxRecycled`
13. **auto-pause**（:467-478）：**在 calculateStatus 与 Ensure\* 之后**执行，保证本轮同步的 probe condition 可用；由 `Spec.Paused` patch 触发的相位变化下一轮生效。
14. **写 status**（:480）：`updateSandboxStatus`。

## 3. 暂停（Pause）的真实链路

### 3.1 进入 Panded 前提（`preparePausedPhase`，:506-575）
1. 首次进入：加 `SandboxFinalizer`（保证暂停期间删除由 controller 中介清理）+ 初始化 `Paused` condition（Status=False, Reason=Pending）+ `BeginPrePause` 标记。
2. `Ready` 置 False。
3. `waitForExistingCheckpoint`（:576-655）：若存在 pending/creating 的外部 Checkpoint → 等待并以 `WaitingForExistingCheckpoint` 记录事件；超过 `checkpointCreationWaitTimeout` 的陈旧 checkpoint 被忽略。
4. `Paused` 已为 True → 直接返回（幂等，不重复执行 PrePause）。
5. 执行 `PrePause` hook 到终态；非终态按 hook 的 requeue 或 `checkpointWaitRequeueInterval` 重试。注释明确 **at-least-once** 契约：命令执行后、状态写入前崩溃会在重启时重放。

### 3.2 实际暂停（`commonControl.EnsureSandboxPaused`，`core/common_control.go:302-306`）
```go
// commonControl only supports the Stop pause strategy.
return ensureStopPaused(ctx, r.Client, args, SandboxPausedReasonStopPauseSucceed)
```
`ensureStopPaused`（`core/util.go:90-131`）：
- pod==nil → `Paused=True`（Reason=StopPauseSucceed），完成。
- pod 删除中 → 等待。
- 否则 `Delete(pod, GracePeriodSeconds=5)`。

### 3.3 Hibernate / Checkpoint 路径（**未接线**）
- `ensureCheckpointPaused`（`core/util.go:138-166`）存在：`AssumePodCheckpointed`（image 校验 + Checkpoint CR 生命周期）→ checkpoint 成功后 `ensureStopPaused`。
- **调用者仅 `core/util_test.go:1099`**；生产代码零调用。
- `PauseStrategyHibernate` 在 `api/v1alpha1` 定义，`pkg/` 生产代码零引用（仅 deepcopy + `sandboxset/revision_test.go`）。
- `getControl`（`pkg/controller/sandbox/sandbox_control.go:25-27`）忽略 pod 参数，恒返回 `controls[CommonControlName]`（= commonControl，只支持 Stop）。
- 结论：**sandbox.spec.pauseStrategy=Hibernate 在当前 HEAD 不改变暂停行为**（仍是删 Pod）。属实现事实（S3），非文档声明。

### 3.4 暂停完成后的保留
- `pauseSandbox`（`auto_pause.go:637-670`）：若配置 paused-retention，暂停时把 `ShutdownTime`/`PauseTime` 延长到 `PausedShutdownTime(now, retention)`。
- 无 retention 时 `ShutdownTime` 仍是硬上限，`handleShutdownTimeout` 不等待可能永不 idle 的 probe。

## 4. 恢复（Resume）

### 4.1 Stop 策略（`common_control.go:308-380`）
- Pod 仍在 Terminating → 返回错误，等待。
- pod==nil → `CreatePod(IsResume=true)`。
- Pod Running 且 **init container image 与 spec 一致** → `markResumeSucceeded`。
  - `isContainersConsistent`（:353-379）比较 init container 的 `Status.InitContainerStatuses[].Image` 与 `Spec.Images`，处理 registry 短名归一化（`normalizeImageRef`）。
- `markResumeSucceeded`（`core/util.go:168-200`）：
  - 无条件 `Resumed=True`（兼容升级 Resuming 阶段未预置 condition 的情况）；
  - `RuntimeInitialized` 重置为 Pending → 每次 resume 触发 runtime 重新初始化与 CSI re-mount；
  - 预置 PostResumeHook 标记（配置了 hook 则 Pending，否则不存在）。

### 4.2 Resume 收尾（`finalizeResumePhase`，:656-669）
- 移除 `Paused` condition；
- `resetProbeDrivenState`（清空 probe 驱动状态，下一轮暂停周期重新开始）；
- 移除 finalizer（best-effort，失败不阻塞，`EnsureSandboxTerminated` 兜底）。

### 4.3 事件驱动唤醒（gateway）
`pkg/sandbox-gateway/wake/wake.go:120-200`：
- UID 匹配防同名重建误唤醒（:70-96）。
- `Wake` 复用 `sandboxcr.Sandbox.Resume`（:143-150）。
- autoPause 判定 = `Spec.PauseTime != nil`；hasDeadline = autoPause || `ShutdownTime != nil`。
- 唤醒时 `ShutdownTime = now + 100 年`（`timeout.ForeverReservePausedSandboxDuration`），防止陈旧 retention 删除刚唤醒沙箱。
- 仅当 wake rule 的 `PauseTimeout > 0` 才设新 `PauseTime`（并应用 resume timeout floor：避免 resume 中途到期被 controller 再暂停）；否则清空 stale PauseTime，保持运行直到再次暂停/删除。
- 唤醒后：本地 registry `Upsert(route)` + `proxy.SyncRouteWithPeers`（失败只记日志，peer 最终靠 informer 追上）。

## 5. 升级（Upgrade）

`SandboxUpgradePolicy.Type` 空 = 不支持升级；`Enum=Recreate;CheckpointRestore`（`sandbox_types.go:164-174`）。

`UpgradeControl.EnsureSandboxUpgraded`（`core/upgrade_control.go:116-315`）状态机（Condition Reason 驱动）：
```
PreUpgrade (hook)
  → Checkpointing (CheckpointRestore 才真正建 Checkpoint；否则 short-circuit)
  → UpgradePod (Recreate；完成后重新 Get Pod，旧对象 stale)
  → PostUpgrade (hook)
  → Resuming (Pod 重建)
  → ResumeSucceed (等待 template patch)
```

证据点：
- `EnsureCheckpointForUpgrade`（`checkpoint.go:365-398`）：非 CheckpointRestore 直接 `(true,"",nil)`；内容固定 `filesystem`；成功/失败/等待三态。
- `performRecreateUpgrade`（:387-505）：执行替换并在完成后 re-fetch pod。
- `SandboxUpgradeResumeFromFailedStepGate`（默认 true）：支持从失败步骤续跑，而非总从 PreUpgrade 重启（`features.go:53-56`）。

## 6. 回收（Recycle）
- controller 保留 `EnsureSandboxRecycled`（`sandbox_controller.go:452-455`）与 `core/recycle.go`（470 行）。
- `pkg/controller/sandbox/core/recycle.go:415` 会把 SandboxSet 的 `PersistentContents` 克隆进回收对象。
- **治理约束**：sandbox-manager 部署中 recycle 不存在（`AGENTS.md:14-26`），manager 的 API/Manager/Infra/cache/quota/token 均不得加 recycle 兼容。

## 7. 定时器与超时
- `checkTimers`（:1130-1162）：ShutdownTime / PauseTime。
- `handlePauseTimeout`（:1162-1203）：到期触发暂停。
- `handleShutdownTimeout`（:1203-1231）：到期删除；不因 auto-pause probe 未 idle 而延迟。
- `calcTimeoutRequeue`（:1231-1249）：计算下一次定时器 requeue。
- retention 解析：`resolveRetentionAnnotationOrDefault`（:1249-）。

## 8. 状态机完整性观察
- 所有 phase 分支都有对应 `Ensure*`；default 分支记录 invalid phase 并 requeue（:456-459）。
- 每次 Ensure 失败仍尝试 `updateSandboxStatus` 持久化升级状态（:461-466），避免错误路径丢状态。
- `phaseBefore` 与 `newStatus.Phase` 对比用于事件/日志（:468-470）。

# 01 Project Layer — Data Model (CRD)

> 全部字段名与 json tag 取自 `api/v1alpha1/*_types.go` 实际内容；生成代码（deepcopy/client）不计入知识。

## 1. Sandbox（核心运行时对象）

`SandboxSpec`（`sandbox_types.go`）：
| 字段 | 类型 | 语义 |
|---|---|---|
| `paused` | bool | 暂停/恢复的**意图**（不是实际状态） |
| `pauseStrategy` | `*PauseStrategy` | `Stop`（默认）/`Hibernate`（类型存在但生产未消费，见 05 C-01） |
| `persistentContents` | []string | `filesystem` / `memory`（是否持久化根文件系统/内存） |
| `pauseTime` | *Time | 自动暂停时间点（存在即视为 auto-pause 模式） |
| `shutdownTime` | *Time | 硬生命周期上限 |
| `lifecycle` | `*SandboxLifecycle` | PrePause/PostResume/PreUpgrade/PostUpgrade hooks |
| `probes` | []Probe | ≤16，仅 exec/periodSeconds/timeoutSeconds/failureThreshold 被消费 |
| `autoPausePolicy` | `*AutoPausePolicy` | Pause.WhenProbedIdleState / Resume.WhenProbedScheduleTime / Resume.OnIngressTraffic |
| `upgradePolicy` | `*SandboxUpgradePolicy` | Type: `Recreate` / `CheckpointRestore`；空=不支持升级 |
| `runtimes` | []RuntimeConfig | 容器运行时配置 |
| `EmbeddedSandboxTemplate` | inline | 模板内联（容器/卷/镜像） |

`SandboxStatus`：`observedGeneration / phase / message / conditions(merge-patch by type) / podInfo / nodeName / sandboxIp / updateRevision / recycledCount / schedules[]`。

**条件类型**（全集，来自 constants）：`Ready`、`SandboxPaused`、`SandboxResumed`、`InplaceUpdate`、`Upgrading`、`RuntimeInitialized`、`Recycling`、`ProbeValid`、`PrePauseHook`、`PostResumeHook`；probe 结果动态类型 `agents.kruise.io/<probe-name>`。

**Schedules**：`ScheduleReasonProbedIdle`（NextPauseTime）/ `ScheduleReasonProbedSchedule`（NextResumeTime）——自动暂停/恢复的**可观测决策记录**。

## 2. Checkpoint（状态快照对象）

`CheckpointSpec`：
| 字段 | 语义 |
|---|---|
| `sandboxName` / `podName` | 目标 |
| `containers` | 空=所有可 checkpoint 容器 |
| `keepRunning` | `*bool`，默认 true（checkpoint 期间保持 Pod 运行） |
| `persistentContents` | `podInfo` / `memory` / `filesystem` / `gpuMemory` |
| `ttlAfterFinished` | "30s,30m,30d" |

`CheckpointStatus`：`phase / checkpointId / checkpointLocation / completionTime / podTemplateDelta / conditions`。
- **`podTemplateDelta`**（注释原文）："Strategic Merge Patch between the running Pod at pause time and the base Pod generated from sandbox.spec.template + runtime injection. Applied at resume time to reconstruct the Pod faithfully." → 恢复时忠实重建 Pod 的关键数据。
- 内容派生 label：`checkpointLabelForContents`（单内容=内容名；多内容排序 join "-"，如 `filesystem-memory`）；`ensureCheckpointCR` 先查新 label，再回退 legacy v0.5.22 label（控制器升级兼容）。

## 3. SandboxSet / SandboxClaim / SandboxTemplate（池化与领取）

- `SandboxSet`：warm pool 管理（replicas、模板 revision、pool 沙箱生成）。
- `SandboxClaim`：从池中领取（claim）一个 running 沙箱并绑定 owner。
- `SandboxTemplate`：模板对象（Checkpoint clone 会生成一个 OwnerReference 指向 Checkpoint 的模板，`clone.go:660-690`）。
- 模板 revision hash 有**双算法兼容**：新算法对完整 `SandboxTemplateSpec` 做 FNV-32；legacy 算法（`revision.go:92-140`）只覆盖 `spec.template`，注释明确**不包含** `VolumeClaimTemplates/PersistentContents/Runtimes/PauseStrategy/Probes/AutoPausePolicy`。用于滚动升级后旧 Sandbox 的 hash 不被误判为 outdated。

## 4. PoolAutoscaler（池扩缩）

`PoolAutoscalerSpec`：`scaleTargetRef`（必填）、`maxReplicas`（≥1）、`minReplicas`（默认 0）、`cronPolicies`（≤20，按 name 去重，**优先于 capacity**）、`capacityPolicy`、`suspend`。
`Status`：`currentReplicas / desiredReplicas / suspended / appliedCronPolicies[] / currentCapacity(water level) / conditions`。

## 5. SandboxUpdateOps（批量升级作业）

`Spec`：`selector`（必填）、`updateStrategy`、`patch`（**Strategic Merge Patch on Sandbox 的 PodTemplateSpec**）、`lifecycle`、`paused`、`stateFilter`（空=`[Running]`；**Upgrading 中的沙箱始终被跟踪**，代表本 ops 拥有的在途升级）。
`Status`：`phase / replicas / updatedReplicas / failedReplicas / updatingReplicas / conditions`。

## 6. Commit / TrafficPolicy / Security

- `Commit`：从 Sandbox Pod 提交容器镜像（`pkg/controller/commit`，commit-job 二进制用 nerdctl）。
- `TrafficPolicy` / `GlobalTrafficPolicy`：流量策略（gateway 路由/访问控制）。
- `security/v1alpha1`：`AgentAuthenticationConfig` / `AgentIdentity` / `AgentRole` / `AgentRoleBinding` / `CredentialProvider`（身份与授权模型）。

## 7. 数据模型层面的设计观察

1. **意图与状态分离**：`spec.paused` 是意图，`status.conditions[SandboxPaused]` 才代表"Pod 已真正删除"。`tryResume` 明确只在 condition=True 时恢复（`auto_pause.go:598-604`）。
2. **决策可观测**：`Schedules` 把"下一次何时暂停/恢复"沉淀为 status 字段，既是观测也是恢复依据（Pod 删除后 probe 不可用）。
3. **快照携带重建信息（读取侧）**：`PodTemplateDelta` 在 resume 时被 `ApplyPodTemplateDelta` 应用（`pod_control.go:170-172`，best-effort）；但 `BuildPodTemplateDelta` 在本仓库无生产调用，写入侧未确认（C-04a）。
4. **升级策略是显式枚举**：空值=不支持升级，避免默认升级带来的意外 Pod 替换。
5. **模板复制有"不可变字段"清单**：身份/存储/运行时/probe 不被 revision 覆盖，防止滚动更新破坏已绑定资源。

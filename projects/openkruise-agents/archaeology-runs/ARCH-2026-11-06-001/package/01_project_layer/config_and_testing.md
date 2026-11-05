# 01 Project Layer — Configuration, Extension & Testing

## 1. Feature Gates（15 个，`pkg/features/features.go`）

| Gate | 默认 | 语义（代码/注释） |
|---|---|---|
| Sandbox | true | Sandbox 控制器 |
| SandboxSet | true | 池/模板 |
| SandboxClaim | true | 领取 |
| SandboxCreatePodRateLimit | false | 创建 Pod 限流 |
| SandboxCreatePodInjectConfig | false | 创建时注入配置 |
| CachePodLabelSelector | true | 缓存按 label 选择 |
| SandboxInPlaceResourceResize | true | 原地资源调整 |
| SandboxMultiClusterNaming | false | 多集群命名防冲突 |
| SandboxUpgradeResumeFromFailedStep | true | 从失败步骤续跑升级（否则每次模板变更从 PreUpgrade 重来） |
| SecurityIdentityProvider | false | 外部身份提供者签发令牌（否则随机 UUID） |
| **SandboxPauseCheckpoint** | **false** | **注释原文：历史开关，"now always on and no longer consumed anywhere; definition kept so existing --feature-gates configurations do not fail validation"** |
| Commit | false | 镜像提交 |
| PoolAutoscaler | true | 池扩缩 |
| AutoPauseController | true | probe 驱动自动暂停/恢复；关闭则不注入 probe、不做决策 |
| **KruiseIntegration** | **false** | 声明集群有 OpenKruise：缺 PodProbeMarker CRD 会 Warning + 真实节点 reconcile 持续失败（**宁可失败也不静默丢 probe**）；关闭则真实节点 probe 条件停留 Unknown → **pause 决策 fail closed** |

**观察**：`SandboxPauseCheckpoint` 是"已消费完毕但保留定义"的 gate 残骸；`KruiseIntegration` 是"依赖可选、语义不能静默降级"的典型表达。

## 2. Webhook（`pkg/webhook/`）

| 子包 | 作用 |
|---|---|
| `pod` | Pod 准入（probe 注入/配置注入/镜像/身份） |
| `poolautoscaler` | 校验 min/max、capacity 约束（如百分比 target 时 minReplicas≥1 防 0 池死锁） |
| `sandboxset` | defaulter：`defaultPersistentContents` 在 Create 且 spec 为空时写入（`sandboxset_defaulter.go:87-95`） |
| `sandboxtemplate` | 模板默认值/校验 |
| `sandboxupdateops` | ops 校验（patch/selector/stateFilter） |
| `securityprofile` | 安全配置校验 |

## 3. 配置目录（`config/`）

`crd/bases`、`default`、`manager`、`rbac`、`webhook`、`samples`、`network-policy`、`prometheus`、`jaeger`、`otel-collector`、`agent-identity-provider`、`sandbox-gateway`（含 `runtime-mtls`）、`sandbox-manager`、`undeploy`。
- 生成产物（crd/rbac/webhook）不得手改（`AGENTS.md`）。
- 可观测性：Prometheus + Jaeger + OTel collector；controller 内 `pkg/tracing` 提供 Reconcile span 与 trace-log 关联（checkpoint 轮询日志用 `klog.FromContext` 以携带 traceID）。

## 4. 扩展机制

| 扩展点 | 机制 | 证据 |
|---|---|---|
| 新增控制器 | `controllers.go` 的 `Deps` 结构追加字段 + `SetupWithManager` 注册；注释明确"新增依赖破坏 compile-time checks，故集中到 Deps" | `controllers.go:22-65` |
| 新 API | `docs/proposals/` 提案（仓库当前无 docs，机制不可见） | `AGENTS.md` |
| 新 CRD | `api/v1alpha1` + `make generate manifests` | `AGENTS.md` |
| 新 feature | `pkg/features` **仅 controller**；manager/servers 禁止导入 | `AGENTS.md` |
| 新生命周期阶段 | `SandboxPhase` + `calculateStatus` + `Ensure*` 分派 | `sandbox_controller.go:819-960` |
| 新 probe 语义 | `Probes` 复用 corev1.Probe，语义由 `AutoPausePolicy` 赋予；仅 exec/period/timeout/failureThreshold 被消费 | `autopause/validation.go` |

## 5. 测试体系（283 个 `*_test.go`）

### 5.1 单元/表驱动
- `AGENTS.md` 强制：优先 table-driven；不引入 stub/fake/mock，优先真实实现；只跑受影响包；并发/概率失败用 `-count=N`/`-race` 复跑。
- 代表：`core/util_test.go`（ensureStopPaused / ensureCheckpointPaused 状态）、`sandboxset/revision_test.go`（不可变字段 + PauseStrategy 透传）。

### 5.2 e2e（16 套）
`test/e2e/`：sandbox、checkpoint、lifecycle hook、recycle、upgrade、claim、set、updateops、commit、okactl、pod probe、tracing、e2b、kwok_loadtest 等。
- CI 矩阵：K8s 1.32/1.37；E2B 2.24/2.51/latest/mysql-latest；kruise-integration、gateway-security、tracing、upgrade、govulncheck/codeql/license/镜像构建。

### 5.3 测试揭示的系统重点（不是"覆盖了什么"，而是"在意什么"）
| 测试主题 | 说明系统重视 |
|---|---|
| lifecycle hook / pause-resume | hook 至少一次语义、状态机幂等 |
| upgrade / updateops | 多阶段升级、批量作业、失败续跑 |
| checkpoint | Checkpoint CR 生命周期、恢复数据 |
| pod probe | probe 注入与条件同步 |
| tracing | span 与 trace-log 关联 |
| gateway-security | JWT/mTLS/路由鉴权 |
| kwok_loadtest | 大规模虚拟节点下的调度/池行为 |

### 5.4 测试覆盖的边界（Coverage 缺口，见 05 C-07）
- 本 Package 未逐条执行测试（无集群），仅读取测试结构与断言。
- `ensureCheckpointPaused` 有单测但无生产调用者 → 测试通过不代表功能可达（**测试覆盖 ≠ 路径可达**）。
- 未覆盖：`pkg/servers` 全量 E2B 协议映射、`pkg/utils` 工具层、`pkg/agent-runtime` 内部实现。

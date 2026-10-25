# Project Layer — Agent Substrate（01 Project Map）

> 事实底座。证据均来自 repo 实际内容（commit 59a58a5f）。设计意图标注 [doc]，实现标注 [code]。

## 1. 定位与边界
- **是**：K8s-native 有状态 agent 沙箱运行时；actor（有状态工作负载）与 worker（热 Pod）映射器；快照-恢复编排 + 网络中介 + 身份签发。
- **不是**：agent 框架 / harness / SDK（README：框架无关，跑标准 OCI 镜像）；不做应用层人-actor 认证（scope：docs/security.md "Security Scope"）。
- 多租户模型：互不信任的 actor 共享物理节点（bin-packing）；**不在 actor 内做多租户**（"Actors are designed to be cheap and efficient enough to run many"）。

## 2. 架构（模块地图）
- **控制面**：`ate-api-server`（gRPC Control service 50+ RPC；scheduler/workflow engine/OpenFGA 授权）+ `atecontroller`（WorkerPool CRD→Deployment 调和）+ `podcertcontroller`（短时 mTLS Pod 证书）
- **节点面**：`atelet`（DaemonSet 节点 supervisor；Run/Checkpoint/Restore/PromoteSnapshot/DetachActor 工作流）+ `ateworker-gvisor` / `ateworker-microvm`（worker Pod 内沙箱 herder，RunWorkload/CheckpointWorkload/RestoreWorkload/TerminateWorkload/GetWorkloadStats）
- **网络面**：`atenet-router`（Envoy ext_proc 入口）+ `atenet-egress`（出站 PEP）+ `atunnel`（worker 侧 mTLS CONNECT 隧道 8443/443）
- **存储面**：`objectstore-plugin`（sidecar 提供 objectstore.v1 gRPC，GCS/S3）+ atelet/ateapi 的 sidecar（NodeProvider/ControlProvider）；sandbox 二进制走 `AssetProvider`
- **工具**：`kubectl-ate`、`ate-setup`、`ate-idp-server`、`credential-provider`、`objectstore-plugin`、benchmarking（Locust/nighthawk）

## 3. 资源模型（双存储体系）
- **K8s CRD（声明式，低频率）**：WorkerPool（spec.sandboxClasses[].name、worker_selector、资源配额）+ SandboxConfig（cluster-scoped 沙箱二进制定义）
- **PostgreSQL（高频态）**：ActorTemplate（golden snapshot 版本锚）/ Actor（七态 status + snapshotRefs）/ Worker（IDLE/BUSY + podIP）/ Tag（不可变别名+保留钉）/ Atespace（分组+存储路径分区）
- **反规范化**：WorkerAssignment 携带 worker_pod_ips 副本——atenet 热路径免二次查询（proto 注释明示）

## 4. Actor 生命周期状态机
- `SUSPENDED → RESUMING → RUNNING → SUSPENDING → SUSPENDED`；`SUSPENDED/CRASHED → (RevertActor) → SUSPENDED`；任意态 → `DeleteActor` → DELETING → GC
- 七态枚举（proto）：RESUMING/RUNNING/SUSPENDING/SUSPENDED/CRASHED/DELETING/REVERTING
- **CRASHED 语义**：crash 字段记录原因+时间；revert 清除 crash 回 SUSPENDED；last_assigned_generation 单调不降（revert 也不回退，保 stale 快照清理）

## 5. 快照体系
- Snapshot 记录：uuid/generation（单调从 1）/owner（actor/tag）/fidelity/locality/durable_snapshot(PENDING|COMPLETED)/sandbox_runtime
- fidelity 三级：VOLUMES(1) ⊂ ROOTFS(2) ⊂ MEMORY(3)（每级含下级）
- 快照 pin 到 sandbox_runtime（memory restore 只能跑在该运行时 worker——跨运行时升级可复现）
- Tag：不可变别名 + 保留钉（自持快照副本，outlive actor；PUBLISHED 可跨 atespace 复用；tag 存在时 atespace 不可删）
- durability 双模式：async（默认，先返后传，PENDING→COMPLETED）/ sync（先传后返，立即 COMPLETED）

## 6. 测试体系
- 670 个 *_test.go（≈47% 文件带测试）；表驱动优先（AGENTS.md 强制）；`-count=3 -shuffle=on`；禁固定 sleep
- e2e（internal/e2e：atenet_dataplane.go 等）；每个 cmd 模块配套 *_test.go（cmd/atelet 44 个文件 25+ 测试文件）
- CI：pr-workflow.yaml + release.yaml + govulncheck.yaml；`.agents/skills/security-status-report/`（threat-model 声明 CI 自动化安全评估）

## 7. 配置与权限
- 环境变量/flag 驱动（eg `--snapshot-upload-delay`、`--snapshot-detach-delay`、`--parked-request-budget=5s`、`--parked-request-max=1024`、`ATE_STORAGE_BACKEND`、`ATE_INGRESS_ALLOWED_SPIFFE_IDS`）
- 授权：OpenFGA 嵌入式（bootstrap owners 必配，ate-api-server 无 bootstrap owner 拒绝启动）；owner/editor/viewer 角色于全局或 atespace；组件持固定 system roles（Pod 证书绑定）
- EgressPolicy 作用域（Reconciled）：actor 级 EgressPolicy（GetActorEgressPolicy/CreateActorEgressPolicy 等 4 RPC）+ Global/Atespace AccessPolicy 并存 [proto]
- 控制面隔离：节点标签 `ate.dev/workloadType=ate-control-plane|ate-postgres` + NoSchedule 容忍（可选安装）

## 8. 外部依赖（go.mod 锚）
k8s v0.36-0.37 · controller-runtime v0.24.1 · spiffe/go-spiffe v2.7.0 · envoy go-control-plane v0.14 · GCP（compute/container/iam/storage）+ AWS SDK v2 · Masterminds/squirrel · containerd ttrpc · CSI spec

## 9. 治理机制（AGENTS.md / GOVERNANCE.md / COLLABORATING.md）
- AI 贡献：PR 必须声明 AI 用途；拒绝 agent/bot 账号（human 签贡献协议）；AI 生成的 commit message 不允许
- fail loudly：只忽略明确预期错误（如 ENOENT）；禁止静默 fallback 到不安全/空操作
- atelet 与 worker 运行时互不透明（仅共享 internal/proto/ateworkerpb + nodepath；v1.0.0 前不强制）
- actor 功能双端 e2e（gVisor+microVM）；image 引用 pin digest；CI 环境变量只在命名 fail-closed 谓词内读
- 表驱动测试；新测试须在无变更时失败（证明测试有效）

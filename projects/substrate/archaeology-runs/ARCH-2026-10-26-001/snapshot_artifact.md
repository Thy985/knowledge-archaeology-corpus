# Repository Snapshot Artifact — ARCH-2026-10-26-001

## 快照信息
- **repository**: https://github.com/agent-substrate/substrate.git
- **branch**: main
- **commit SHA**: `59a58a5fb2c0e7958722ac4c4bc609fe7f16ddf8`（2026-10-25 15:26:43 +0000，`manifests: Run control plane containers as non-root (#3028)`）
- **最新 release**: v0.6.0（2026-10-22）；v0.5.0（10-14）；v0.4.0
- **克隆时间**: 2026-10-26（depth=1 浅克隆）
- **stars/forks**: 5785 / 627（GitHub API 2026-10-26；仅热度参考，非排序依据）

## 项目基础地图

### 定位
Agent Substrate（模块 `go.ate.dev/substrate`）——**零信任 agent 沙箱运行时**：把大量"actor"（有状态 agent 工作负载）映射到少量热"worker"上，actor 空闲时快照出内存、请求到来时恢复，实现高密度多路复用；K8s-native（Pod/autoscaling/NetworkPolicy 之上加 agent 专属调度、快照、身份、网络中介）。

### 语言与规模
- Go 1.27（toolchain 1.27.3）；1425 个 .go 文件（非 vendor）≈ 42.7 万行；670 个 `*_test.go`
- vendor/ 内嵌依赖；license Apache-2.0

### 入口（cmd/，15 个二进制）
| 二进制 | 职责 |
|---|---|
| `ateapi` | 控制面 API server（gRPC：actor 生命周期/调度/OpenFGA 授权/凭据签发） |
| `atecontroller` | K8s controller（WorkerPool → Deployment 调和 + 网络策略） |
| `atelet` | 节点级 supervisor（DaemonSet；沙箱生命周期协调/checkpoint-restore/节点凭据） |
| `ateworker-gvisor` / `ateworker-microvm` | worker Pod 内沙箱-herder（RunWorkload/CheckpointWorkload/RestoreWorkload） |
| `atenet` | 网络栈（atenet-router 入口 Envoy ext_proc / atenet-egress 出口策略代理） |
| `atunnel` | worker 上 mTLS CONNECT 隧道监听（端口 8443/443） |
| `ate-idp-server` | 身份提供商 |
| `podcertcontroller` | 短时 mTLS Pod 证书签发控制器 |
| `credential-provider` / `objectstore-plugin` / `dataplane` / `kubectl-ate` / `ate-setup` / `benchmarking` | 辅助/工具 |

### 核心模块
- **api/ateapi/v1**：`ateapi.proto` 定义（Actor/ActorTemplate/Worker/Snapshot/atespace 等资源 + 生命周期 RPC）
- **internal/ateworker**（2557 行）：沙箱 herder，runsc（gVisor）/Kata+Cloud Hypervisor（microVM）checkpoint/restore
- **internal/atunnel**（4904 行）：worker 侧隧道 + actor 私有网络（veth/port 443 认证监听）
- **internal/actorevent**（380 行）：actor 事件
- **internal/actoridjwt / localjwtauthority**（198/621 行）：actor 身份 JWT 签发与本地权威（1h X.509 + on-demand JWT）
- **internal/credbundle**（222 行）：凭据束（零环境凭据 + 动态水合）
- **internal/egresspolicy**（458 行）：出站策略（fail-closed PEP）
- **internal/ateapiauth / ateinterceptors / atunnel / ateworkervalidation**：API 认证/拦截
- **pkg/sandboxclass / assetfetch / objectstorage / postgressetup / client**
- **internal/e2e**：端到端测试（atenet_dataplane.go 等）

### 核心数据结构/状态机（docs/architecture.md 声明 + proto 佐证）
- Actor state: `SUSPENDED → RESUMING → RUNNING → SUSPENDING → SUSPENDED`；`SUSPENDED/CRASHED → RevertActor`；任意态 → `DeleteActor`
- Actor record：全局唯一 ID、Worker IP、RUNNING/SUSPENDED、snapshotRefs（含 durableSnapshot PENDING/COMPLETED + locality）
- Worker record：IDLE/BUSY + 当前托管 actor
- WorkerPool（CRD，声明式）vs Actor/ActorTemplate/Worker（PostgreSQL 高频态）
- Snapshot：memory+disk 合一版本化；MEMORY/VOLUMES/ROOTFS 三类；tag 不可变别名+保留钉

### 生命周期（控制面 + 节点）
- CreateActor（SUSPENDED + golden snapshot）→ ResumeActor（Gateway 拦截→API 分配 worker→atelet Restore→RUNNING→路由）→ SuspendActor（checkpoint→本地+异步/同步上传→SUSPENDED→worker 回收）→ RevertActor/DeleteActor
- 同步 vs 异步 suspend：durableSnapshot PENDING（后台上传 `--snapshot-upload-delay`）vs COMPLETED（先传后返）；`--snapshot-detach-delay` 清本地 locality

### 网络（文档声明，待代码验证）
- 入口：Envoy ext_proc 拦截 `ate-target-actor: <atespace>/<actor>` → ResumeActor → mTLS CONNECT 隧道到 atunnel:443 → actor 私有 veth
- 出口：atenet-egress fail-closed PEP（身份校验 + 目标 allowlist + TLS 检查 + 凭据注入；未配置/失败即丢弃；非 TCP 丢弃）
- 任意端口：CONNECT <actor-dns>:port；仅 HTTP(S)

### 身份与安全（docs/security.md 状态声明——审计锚）
- Met：Always Sandboxed / Per-Actor Storage Segregation / Per-Actor Network Isolation / Zero Ambient Credentials / Distinct Workload Identity / Universal Component mTLS / Untrusted Storage / Dual-Party Snapshot Authorization / No Access by Default / Authenticated and Policy-Enforced Ingress / Fail-Closed Policy-Enforced Egress / Structured Audit Trail / Dynamic Quarantine
- In Progress：Delete Guarantee / Node Blast Radius / Atespace Separation（OpenFGA ReBAC）/ Comprehensive Quotas（Future）
- 威胁模型 docs/threat-model.md（T-01 起，含 mitigation invariants + CI 自动化安全评估 `.agents/skills/security-status-report/`）

### 治理（AGENTS.md / GOVERNANCE.md / COLLABORATING.md）
- AI 贡献规则：PR 必须声明使用 AI 及方式；拒绝 agent/bot 账号贡献（human 签贡献协议）；AI 生成 commit message 不允许
- 代码规则：fail loudly（只忽略明确预期错误如 ENOENT；禁止静默 fallback 到不安全/空操作）；性能声明须有测量；**atelet 与 worker 运行时互不透明**（仅共享 internal/proto/ateworkerpb + nodepath，v1.0.0 前不强制）
- actor 功能须 gVisor+microVM 双端 e2e；shell 代码独立 .sh（shellcheck）；image 引用 pin digest；CI 环境变量只在命名 fail-closed 谓词内读
- PR：squash-merge；每个 PR 一个变更；issue-first
- 测试：表驱动优先；新测试须在无变更时失败；`-count=3 -shuffle=on`；禁固定 sleep

### CI（.github/workflows）
- pr-workflow.yaml / release.yaml / govulncheck.yaml（3 个 workflow 可见；docs 提到 `.agents/skills/security-status-report/` CI 自动化安全评估）

### 外部依赖（go.mod 锚）
- k8s.io/{api,apiextensions-apiserver,apimachinery,client-go,kubelet,metrics} v0.36-0.37；sigs.k8s.io/controller-runtime v0.24.1
- github.com/spiffe/go-spiffe/v2 v2.7.0（SPIFFE 身份）
- envoyproxy/go-control-plane v0.14（ext_proc 网络策略）
- GCP（compute/container/iam/monitoring/storage）+ AWS SDK v2（S3）云集成
- Masterminds/squirrel（SQL）、CSI spec、containerd ttrpc、crane（digest 工具）

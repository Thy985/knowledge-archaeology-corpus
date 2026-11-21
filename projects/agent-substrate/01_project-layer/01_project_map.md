# 01 · Project Layer — Agent Substrate 项目地图

> L0 事实层。全部内容可回溯到仓库实际内容（文件/代码/proto/文档）。

## 1.1 项目定位
- README 原文事实："Zero-trust sandboxing for millions of agents on Kubernetes"；"security-hardened, zero-trust runtime for sandboxed agents"
- 核心模式（README）：大量 actors（有状态 agent 类工作负载）→ 小池 workers；空闲 actor 快照出内存，请求到达时恢复
- 低意见运行时（"low-opinion runtime, not an agent SDK"）；框架无关（ADK/LangChain/Claude Code/Codex/MCP server 均可承载，无需 SDK 改动）
- 明文 Linux 容器**不支持**：只支持 gVisor（runsc）与 Cloud Hypervisor microVM

## 1.2 架构总览（组件拓扑）
```
                         ┌─────────────────────────────────────────────┐
                         │  ateapi（控制面 API server, cmd/ateapi）     │
                         │  Control gRPC + WorkerService 回调           │
                         │  authz: OpenFGA ReBAC + bootstrapOwners      │
                         │  store: PostgreSQL + GCS/S3 对象存储         │
                         └───────┬──────────────────┬──────────────────┘
                                 │ mTLS             │ mTLS
                 ┌───────────────▼──────┐   ┌───────▼──────────────────┐
                 │ atecontroller (K8s)  │   │ atelet（节点 supervisor） │
                 │ 工作池/容量投影       │   │ 快照存储双重授权/恢复     │
                 └───────────┬──────────┘   └───────┬──────────────────┘
                             │                      │
                 ┌───────────▼──────────────────────▼──────────────────┐
                 │ ateworker-gvisor / ateworker-microvm（worker 运行时）│
                 │ checkpoint/suspend/actornet/cgroup/memborrow        │
                 └───────────┬─────────────────────────────────────────┘
                             │ atunnel（actor 隧道，mTLS CONNECT）
                 ┌───────────▼─────────────────────────────────────────┐
                 │ atenet-router（ingress 外环）→ request parking       │
                 │ atenet-egress（egress 网关）→ 凭据外注 + PEP 策略     │
                 └──────────────────────────────────────────────────────┘
```

## 1.3 核心模块清单（internal/ 与 cmd/ 事实）
| 模块 | 路径 | 职责（代码/文档证据） |
|---|---|---|
| 控制面 API | `cmd/ateapi` + `internal/controlapi` | Actor/Worker/Atespace CRUD + Suspend/Resume/Revert 工作流（workflow_resume.go / workflow_suspend.go） |
| 节点 supervisor | `cmd/atelet` + `internal/atelet*` | 快照存储访问、恢复执行、resetActorDirs（删除保证） |
| worker 运行时 | `cmd/ateworker-gvisor` / `cmd/ateworker-microvm` + `internal/ateworker` | checkpoint.go（快照完成标记）、suspend/（上行挂起请求）、actornet/（沙箱网络）、cgroup/（内存守卫） |
| 网络代理 | `cmd/atenet` | ingress router（ext_proc + parking）+ egress gateway（凭据注入 + SNI 规则） |
| 策略评估 | `internal/egresspolicy` | 纯函数策略引擎：Compile/EvaluateRequest/SNI 规则/rank 排序 |
| actor 隧道 | `internal/atunnel` | ingress（mTLS CONNECT 到 worker）+ egress（出站流）+ dns |
| 身份 | `internal/{localjwtauthority,substratex509,oidcjwt,credbundle,podidentity,clustertrustbundle}` | SPIFFE X.509、JWT、组件 mTLS |
| 授权 | `cmd/ateapi/internal/authz` | OpenFGA ReBAC Authorizer |
| 对象存储 | `internal/objectstoreplugin` + `pkg/objectstorage` | GCS/S3 snapshot 存储、AssetPlugin（SHA-256 校验） |
| 快照 | `internal/snapshotcache` / `internal/ateworker/checkpoint.go` | 本地快照缓存、checkpoint-done 标记复用 |
| 请求停车 | `cmd/atenet/internal/router/ingress/parking.go` + docs/concepts/request-parking.md | 瞬时饱和时指数退避重试 |
| e2e | `internal/e2e`（115 文件） | 端到端测试框架（backend/env/fixture/pod/manifest/egresspolicy/credprovider） |

## 1.4 生命周期（Actor/Worker 状态机）
- **ActorState（proto 枚举，8 态）**：UNSPECIFIED=0 / RESUMING=1 / RUNNING=2 / SUSPENDING=3 / SUSPENDED=4 / CRASHED=5 / DELETING=6 / REVERTING=7
- **WorkerState（4 态）**：UNSPECIFIED=0 / ACTIVE=1 / DRAINING=2（终态）/ UNAVAILABLE=3（注册即此态，MarkWorkerAvailable 后可调度）
- **DurabilityMode（2 模式）**：SYNCHRONOUS=1（快照入持久存储后才返回）/ ASYNCHRONOUS=2（节点级捕获即返回，上传后台继续）

## 1.5 配置与部署
- manifests/ate-install（安装）、egress-credential-injection、microvm
- 组件间 mTLS：credential-bundle（podidentity.podcert.ate.dev/credential-bundle.pem，atunnel 默认路径）
- workercapacity：atecontroller 投影 CPU/内存限额到 `/run/worker-capacity/{cpu_millicores,memory_bytes}`

## 1.6 外部依赖（代码证据）
Kubernetes（Pods/NetworkPolicies）、Envoy（ext_proc 集成）、gVisor（runsc）、Cloud Hypervisor、SPIFFE/X.509、PostgreSQL（state store）、GCS/S3、OpenFGA（ReBAC）、OTLP（telemetry，internal/otlprelay）、gRPC

## 1.7 治理机制（GOVERNANCE.md / AGENTS.md / COLLABORATING.md）
- 四层角色：Default → Contributor（Read）→ Maintainer → 更高级（判断式晋升）
- DCO 签署（每 commit sign-off，agent 不得以自己身份签）
- AGENTS.md：AI 协作规则显式化（PR 声明用 AI、禁 AI 味词汇、atelet/worker 互不透明、PR 单改动）
- API 风格/校验/代码风格文档齐备；威胁模型文档化（threat-model.md + threats.json）

## 1.8 重要声明（文档目标态标注，考古关键）
- `docs/concepts/architecture.md` front matter：`status: planned` + `status_note: "Much of this architecture is aspirational and not yet implemented."`
- `docs/security/security.md`：每个安全原则带状态图例（Met / In Progress / Future / Out of Scope）
- **推论**：文档描述的北极星指标（100ms p95 activation、500 activations/sec 等）是**目标**而非已验证实现——考古时以代码/e2e 测试为实现事实来源。

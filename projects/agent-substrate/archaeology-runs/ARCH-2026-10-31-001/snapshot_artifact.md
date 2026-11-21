# Repository Snapshot Artifact — ARCH-2026-10-31-001

## 仓库元信息
| 项 | 值 |
|---|---|
| project | agent-substrate |
| repository | https://github.com/agent-substrate/substrate.git |
| branch | main |
| commit SHA | `542eca1e348e80001dcca48789e9433a84eeb214` |
| commit 时间 | 2026-10-30 16:53:38 +0000 |
| commit message | atenet/router: fail fast on FailedPrecondition instead of parking (#3472) |
| module | go.ate.dev/substrate |
| go 版本 | go 1.27.0 / toolchain go1.27.3 |
| 许可证 | Apache-2.0 |
| 克隆方式 | --depth 1 浅克隆 |
| stars（时点） | 5973 |

## 项目定位（README 事实）
**Zero-trust sandboxing for millions of agents on Kubernetes**。安全加固的零信任 agent 运行时：
- 10x 密度（vs 标准容器）；<500ms resume；>500 activations/sec
- 核心模式：大量 "actors"（有状态 agent 类工作负载）映射到小池 "workers"；空闲 actor 快照出内存，请求到达时恢复
- 低意见运行时（low-opinion），非 agent SDK；可承载任何 agent 栈（ADK/LangChain/Claude Code/Codex/MCP server）
- 支持 gVisor（runsc）与 Cloud Hypervisor microVM 双沙箱；**明文 Linux 容器不支持**
- 安全模型：每 actor 短时 SPIFFE X.509 + 按需 JWT；**沙箱内零凭据**（egress 网关在沙箱外注入）；默认拒绝网络；可插拔策略执行点（PEP）
- 框架无关：跑标准 OCI 镜像，无需 SDK 改动

## 代码规模（不含 vendor）
| 指标 | 值 |
|---|---|
| .go 文件 | 1466（非 vendor） |
| 代码行数 | 439,130（非 vendor） |
| _test.go 文件 | 689 |
| proto 定义 | api/ateapi/v1/ateapi.proto（3259 行）等 3 个 |
| .py 文件 | 79 全部（76 非 vendor + 3 vendor；口径注记见 IVR） |

## 语言与目录结构
- **语言**：Go（99.4% 按文件数；另有 79 .py 测试/工具——76 非 vendor + 3 vendor，少量 .rs）
- **顶层目录**：api/（proto）、cmd/（14 个二进制）、internal/（52 个子包）、pkg/（client 等）、manifests/（ate-install/egress-credential-injection/microvm）、demos/、benchmarking/、docs/、hack/、site/、tools/、vendor/

## 入口（cmd/ 二进制 map）
| 二进制 | 职责（由代码/文档推断） |
|---|---|
| cmd/atecontroller | K8s controller（4 go 文件） |
| cmd/atelet | 节点 supervisor（55 go 文件）——节点代理 |
| cmd/ateworker-gvisor | gVisor worker 运行时（31 go 文件） |
| cmd/ateworker-microvm | microVM worker 运行时（39 go 文件） |
| cmd/ateapi | 控制面 API server（2 go 文件） |
| cmd/atenet | 网络代理（1 go 文件）——ingress router + egress gateway |
| cmd/kubectl-ate | kubectl 插件 |
| cmd/ate-idp-server / ate-setup / podcertcontroller / objectstore-plugin / credential-provider / dataplane | 辅助组件 |

## 核心模块（internal/ 按文件数 top）
| 模块 | 文件数 | 职责推断 |
|---|---|---|
| internal/e2e | 115 | 端到端测试框架（backend/env/fixture/pod/manifest/egresspolicy/credprovider/atenet_dataplane） |
| internal/benchmarking | 62 | 基准（20 万 agent/500 activations/sec） |
| internal/atunnel | 44 | actor 隧道（mTLS 数据面） |
| internal/imagecache | 31 | 镜像/资产缓存 |
| internal/ateworker | 31 | worker 运行时核心 |
| internal/objectstoreplugin | 28 | 对象存储插件（GCS/S3 snapshot 存储） |
| internal/resources | 20 | 资源管理 |

**其余关键**：actorevent/actoridjwt/actorlock/actorlog、ateapiauth/ateattr/ateinterceptors（API 认证拦截）、atelet/ateletdial、atenet、ateworkervalidation、credbundle（凭据包）、egresspolicy、localjwtauthority/localca/substratex509/oidcjwt（身份）、netns、objectstore、podidentity、portforward、sandboxruntimes、snapshotcache、wakeupprobe、workercapacity/workerstats、sizing、volume/volumepath、bootid、cdi、clustertrustbundle

## 核心数据结构（api/ateapi/v1/ateapi.proto）
- **service Control**：Actor CRUD + Suspend/Resume/Revert/Delete + EgressPolicy CRUD + MintActorJwt + Tag CRUD + Worker CRUD/Drain/MarkAvailable/MarkUnavailable + Atespace CRUD（30+ RPC）
- **service WorkerService**（worker→控制面回调）：AcquireWorkerEpoch / RegisterWorker / SetWorkerSandboxRuntimes / MintWorkerActorCertificate / MintWorkerActorJwt / RequestActorSuspend / ReportActorSnapshotState / ReportActorCrashed（8 RPC）

## 状态机（proto 枚举，事实）
- **ActorState（8 态）**：UNSPECIFIED=0 / RESUMING=1 / RUNNING=2 / SUSPENDING=3 / SUSPENDED=4 / CRASHED=5 / DELETING=6 / REVERTING=7
- **WorkerState（4 态）**：UNSPECIFIED=0 / ACTIVE=1 / DRAINING=2（终态，不再回 active）/ UNAVAILABLE=3（注册即此态，MarkWorkerAvailable 后调度）
- **DurabilityMode（2 模式）**：SYNCHRONOUS=1（快照完成后才返回）/ ASYNCHRONOUS=2（节点级捕获即返回，上传后台继续）
- 其他：SnapshotOwner / SnapshotStorageState / SnapshotFidelity / GoldenSnapshotMode / ActorMetadataField / TagScope

## 生命周期语义（proto + request-parking 文档）
- Suspend：actor 从 RUNNING→SUSPENDING→SUSPENDED，生成 snapshot generation（last_assigned_generation 单调递增，Revert 期间不减）
- Resume：SUSPENDED→RESUMING→RUNNING；ateapi AssignWorkerStep 从 WorkerPool 认领 worker，`ResourceExhausted` 时 atenet router 走 **request parking**（默认开启，指数退避重试；park budget 默认 5s；预算到期不停已 in-flight 的 resume，超时恢复"晚到服务"而非失败）
- Crash：进入 CRASHED 记录 ActorCrash{message, crash_time}；revert 清 crash 回 SUSPENDED
- 2026-10-30 最新提交：router 对 FailedPrecondition 快速失败而非 parking（#3472）——parking 的边界条件持续演化

## 安全模型状态（docs/security/security.md 自带状态图例：Met/In Progress/Future/Out of Scope）
- **Met**：Always Sandboxed（gVisor/CHV）；Per-Actor Storage Segregation；Per-Actor Network Isolation（private netns/atunnel）；Zero Ambient Credentials；Distinct Workload Identity（SPIFFE X.509 1h + on-demand JWT + OIDC）；Universal Component mTLS（仅 /healthz /readyz 豁免）；Untrusted Storage（SHA-256 资产验证）；Dual-Party Snapshot Authorization（actor+atelet 双身份，guest 不能直接访问存储）；No Access by Default（loopback/private netns/DNS 过滤/ate-system NetworkPolicies）；Authenticated Ingress（atenet-router/atunnel mTLS）；Fail-Closed Egress（atenet-egress PEP，未配置即丢弃）；Structured Audit Trail；Dynamic Quarantine（egress 撤销+lifecycle halt）
- **In Progress**：Delete Guarantee（atelet resetActorDirs）；Node Blast Radius（Node-attested PodIdentity/checkActorOnNode）；Atespace Separation（OpenFGA ReBAC 已集成，网络策略/配额 pending）
- **Future**：Per-Actor Encryption；Restore Authorization；Snapshot Tainting；Actor-to-Actor Authorization；Comprehensive Quotas & Anti-DoS
- **Out of Scope**：Multi-tenancy inside an actor；Sub-Actor Provenance 记录（由调用方系统负责）

## 治理机制（GOVERNANCE.md / AGENTS.md / COLLABORATING.md）
- 四层角色：Default → Contributor（Read）→ Maintainer → 更高级（按持续高质量贡献晋升，判断式）
- DCO 签署（Developer Certificate of Origin），每 commit 必须 sign-off
- **AGENTS.md 明确 AI 协作规则**：贡献者对自己 agent 提交的一切负责；PR 描述必须声明用了 AI 及方式；禁止 agent/bot 账号提交；agent 不得以自己身份 sign off commit（仅人能认证 DCO）；大改前 adversarial review；写作风格约束（简短直白、禁 em-dash、禁 AI 味词汇如 "robust/comprehensive/seamless/leverage"、"No behavior change" 除非纯重构）
- **架构约束（AGENTS.md）**：atelet 与 worker runtimes 保持互相 opaque，仅共享 internal/proto/ateworkerpb 与 internal/nodepath 路径；直到 v1.0.0 无强制
- API 风格/校验/代码风格文档齐备（docs/api-style-guide.md、api-validation.md、code-style-guide.md）
- 威胁模型文档化：docs/security/threat-model.md + docs/threats.json

## 测试体系
- 689 测试文件；internal/e2e（115 文件）是主 e2e 框架：backend.go/fixture.go/env.go/pod.go/manifest.go/egresspolicy.go/credprovider.go/atenet_dataplane.go/ingressauth.go 等
- CI：.github/workflows/（pr-workflow.yaml + govulncheck.yaml）；README badge 显示 CI + govulncheck
- 双运行时覆盖要求（AGENTS.md）：actor 功能须在 gVisor 与 microVM 都有 e2e 覆盖，除非 PR 明确限定单一 runtime

## 外部依赖（go.mod 级别推断）
- K8s（Pods/NetworkPolicies/autoscaling）；Envoy（ingress ext_proc 集成，request-parking 文档提及）；gVisor（runsc）；Cloud Hypervisor（microVM）；SPIFFE/X.509；PostgreSQL（state store，pkg/postgressetup）；GCS/S3（object storage，objectstore 插件）；OpenFGA（ReBAC 授权）；OTLP（telemetry，internal/otlprelay）；gRPC（ateapi）

## 快照时间戳
2026-10-31（Run 执行日）；克隆时间与 HEAD 记录如上。

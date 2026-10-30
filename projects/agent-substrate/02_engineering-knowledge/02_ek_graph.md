# 02 · Engineering Knowledge — Agent Substrate 工程知识层（EK Graph）

> L1/L2 推理原材料（无损底座）。每条 EK 必须声明 `links`（6 类边），不得是孤立模块说明。
> 40 条 EK，平均出边 ≥1，游离 0%，聚合规则覆盖率 100%（见 03 Knowledge Layer 的 aggregation_rule 引用）。

## 边类型速记
| 边 | 语义 | 方向 |
|---|---|---|
| mechanism | 共享同一实现机制/模式 | 无向 |
| subsystem | 同属一个子系统 | 无向 |
| causal | A 触发/导致 B | A→B |
| dependency | A 依赖 B 的运行时保证 | A→B |
| constraint | A 是 B 的边界/前置条件 | A→B |
| contrast | 同一问题两种解法 | 无向 |

---

## A 组：Actor 生命周期与调度

### EK-01 ResumeActor 是幂等工作流，从持久 actor 状态推导进度
`cmd/ateapi/internal/controlapi/workflow_resume.go`
ResumeActor 以"从持久化 actor 状态 fast-forward 已完成步骤"实现幂等：先 GetActor 读热路径（RUNNING 直接返回，不 upsert PostgreSQL lease 行），再 acquireActorLease 分布式租约，随后 loadActorForResume → ensureVolumesCreated → ensureWorkerAssigned → ensureAteletRestored → finalizeRunning 逐阶段推进；每一步用 store.UpdateActor(版本前提) 持久化中间状态，版本冲突返回 Aborted 让调用方重试。重入时从当前状态继续而非重跑。
- 证据：workflow_resume.go L39-150；`if wasRunning = ... RUNNING; wasRunning { return actor, false, nil }` 两处短路
- links: causal → EK-02（ensureWorkerAssigned 认领 worker）; constraint → EK-03（ActorLock 串行化）; subsystem EK-04
- 状态：S3/S4（实现 + e2e 测试覆盖）

### EK-02 ensureWorkerAssigned 从 WorkerPool 认领 worker，超卖时返回 ResourceExhausted
`cmd/ateapi/internal/controlapi/workflow_resume.go`（ensureWorkerAssigned）
认领逻辑：为 RESUMING actor 从该 actor 的 WorkerPool 找可用 worker；若节点过满则返回 `ResourceExhausted "no worker has room for the actor"`（request-parking.md 引用）。Re-entry 校验已记录 assignment 再复用；版本冲突重读重试。
- 证据：workflow_resume.go；request-parking.md "AssignWorkerStep claims a worker from the actor's WorkerPool... returns ResourceExhausted"
- links: causal → EK-05（parking 把 ResourceExhausted 当 retryable）; mechanism EK-03; subsystem EK-01
- 状态：S3（实现）

### EK-03 actorlock.CancelableMutex 按 actor 串行化生命周期 RPC
`internal/actorlock/actorlock.go`
per-actor 生命周期 RPC 串行化：chan capacity 1 的 cancelable mutex，支持等待时 context 取消（防止等待者卡死 RPC）。
- 证据：actorlock.go 全文（CancelableMutex, NewCancelableMutex, Lock/Unlock）
- links: mechanism EK-01（都是 per-actor 串行化）; dependency → EK-01（Resume 依赖其串行化）; subsystem EK-04
- 状态：S3

### EK-04 Suspend 上行请求（worker→atelet→ateapi）与下行 checkpoint 分离，要求调用方不持 Checkpoint 需要的锁
`internal/ateworker/suspend/suspend.go`
suspend 是"上行半程"：worker 请求 atelet→ateapi 决定挂起；真正挂起经普通路径（Checkpoint）下行回到请求的 worker。文档注释明确要求：请求 in-flight 期间调用方不得持有 Checkpoint 需要的任何锁（防死锁）。
- 证据：suspend.go 包注释；CheckpointContext 注释（"once it holds the actor's lock"）
- links: contrast EK-01（上行请求 vs 下行执行分离）; causal → EK-06; dependency → EK-07; subsystem EK-05
- 状态：S3

### EK-05 请求停车（request parking）预算只限新重试，不取消已 in-flight 的 resume
`cmd/atenet/internal/router/ingress/parking.go` + `docs/concepts/request-parking.md`
Router 收到目标 actor 暂不可服务的请求时"停车"：把 ResumeActor 的 ResourceExhausted/Unavailable（+既有 Aborted 并发冲突）当 retryable，指数退避重试直到成功或 park budget（默认 5s）到期。**预算只限新尝试启动，不取消已提交的 resume**——控制面已承诺工作，取消会丢弃进行中的 restore；超预算的恢复"晚到服务"而非失败。
- 证据：request-parking.md "The budget bounds retries, not a committed resume... an attempt already in flight is never canceled"；parking.go
- links: causal ← EK-02; contrast EK-09（parking 与普通重试预算的差异）; subsystem EK-10
- 状态：S3（#3472 又改了 FailedPrecondition 行为，边界持续演化）

### EK-06 CheckpointContext 保留 values+deadline 但去掉 cancellation
`internal/ateworker/checkpoint.go`
CheckpointWorkload 持锁后运行的 ctx：用 context.WithoutCancel 去掉取消但保留 deadline——atelet 消失不取消可能已停工作负载的 checkpoint。
- 证据：checkpoint.go CheckpointContext（WithoutCancel + 可选 WithDeadline）
- links: causal ← EK-04; mechanism EK-07; subsystem EK-08
- 状态：S3

### EK-07 checkpoint-done 标记 + BootID 校验实现幂等恢复
`internal/ateworker/checkpoint.go`
checkpointDoneFile=".checkpoint-done.json" 在快照文件列出+teardown 后写入（含 BootID/Fidelity/SnapshotFiles）；下次 checkpoint 或 atelet resetActorDirs 清除。ReuseCheckpoint 校验 bootID/fidelity/文件存在才复用——worker 不 sync 快照文件，重启后可能 torn，故以 bootID 为界。
- 证据：checkpoint.go RecordCheckpoint/ReuseCheckpoint/checkpointRecord
- links: mechanism EK-06（幂等恢复族）; causal → EK-08; dependency → EK-17
- 状态：S3 + 测试

### EK-08 suspend 重试预算短（~15s）且只认 Unavailable/Aborted/ResourceExhausted
`internal/ateworker/suspend/suspend.go`
retryBackoff：6 次 × 500ms × factor 2 × jitter 0.2 ≈ 15s。**短是有意的**：idleness 信号是过去事实会衰减（重试越久越可能落在已恢复工作的 actor 上），suspend 取消 in-flight 请求而非排干；预算只骑过 blip（atelet 重启/ateapi 滚动）不熬 outage。retryable 仅三码；NotFound/FailedPrecondition=决定、PermissionDenied/Unauthenticated/InvalidArgument=bug 重试无益、DeadlineExceeded 刻意不重试（超时=挂起本身慢，重发翻倍 5 分钟等待）。
- 证据：suspend.go retryable() 注释全文 + retryBackoff 常量
- links: causal ← EK-04; contrast EK-05（上行请求重试 vs 下行 parking 重试的预算哲学差异）; constraint → EK-09; subsystem EK-10
- 状态：S3

### EK-09 UID 钉住 actor incarnation，防请求存活过 actor 并挂起同名重建体
`internal/ateworker/suspend/suspend.go`
Actor{Atespace,Name,UID} 用 UID 钉住 incarnation；requestOnce 传 ActorUid。防"请求超时/重试期间 actor 被删并重建同名体，旧请求误挂新体"。
- 证据：suspend.go Actor struct 注释 + requestOnce 传 ActorUid
- links: mechanism EK-01（幂等/化身隔离）; constraint ← EK-08; subsystem EK-05
- 状态：S3

### EK-10 atunnel ingress 以 mTLS CONNECT :8443 进 worker，凭据源保留私钥做 mTLS+renewal
`internal/atunnel/tunnel.go` + `internal/atunnel/ingress/ingress.go`
atunnel 接受 inbound mTLS CONNECT（ConnectListenAddress 默认 :8443）；credential-bundle 默认 `/run/podidentity.podcert.ate.dev/credential-bundle.pem`；source 保留私钥用于 mTLS 与 renewal。ingress.go 还有 staleAssignmentHeader（区分 atunnel 拒绝与 actor 应用自己的 421）与 TargetPortHeader（per-request 可变端口）。
- 证据：tunnel.go L57/L85/L86/L222；ingress.go 常量
- links: subsystem EK-11; mechanism EK-24（都是 mTLS 组件通信）; dependency → EK-12
- 状态：S3

### EK-11 egress 数据面经 SNI 规则在 ClientHello 决定 TLS
`cmd/atenet/internal/router/egress/egress.go` + `internal/egresspolicy/egresspolicy.go`（BuildSNIRules）
出站 CONNECT 返回 SNI rules；dataplane 在 ClientHello 时按 SNIRules 决定（passthrough/decrypt）。egresspolicy.go 的 BuildSNIRules 把 policy 编译为按 rank 排序的 SNIRule{Pattern,Mode}。
- 证据：egress.go 包注释 "The dataplane decides TLS at the ClientHello using the SNI rules returned on CONNECT"；egresspolicy.go BuildSNIRules
- links: subsystem EK-10; dependency → EK-12; contrast EK-13（请求级 vs TLS 级决策分离）
- 状态：S3

## B 组：Egress 策略与安全

### EK-12 EgressPolicy 评估是纯函数：Compile 一次解析，Evaluate 多次
`internal/egresspolicy/egresspolicy.go`
包注释：pure package, no I/O, no logging。Policy{rules []compiledRule} 编译一次；EvaluateRequest(dest, decrypted) 按协议（http/https）+ 端口匹配 + name rank 选出最具体规则。同一 parser 被 ateapi 验证与网关匹配共用——"两处不能漂移"。
- 证据：egresspolicy.go 包注释 + Compile/EvaluateRequest
- links: dependency ← EK-11; mechanism EK-13; subsystem EK-14
- 状态：S3 + 测试

### EK-13 匹配 rank：exact > wildcard > any，port rank 次之，tie 保策略顺序
`internal/egresspolicy/egresspolicy.go`
rankExact=0 > rankWildcard=1 > rankAny=2；portRank：命名端口=0 > anyPort=1；matchRank.beats 先比 name 再比 port；tie 保持策略顺序（稳定排序）。HostnamePattern 三态："*.example.com" 只匹配一个非空最左标签（不匹配 "example.com" 与 "a.b.example.com"）；"*" 匹配一切含 IP literal（IP 仅由 "*" 匹配）。
- 证据：egresspolicy.go 常量 + Matches/rank/matchRank 注释
- links: mechanism EK-12; constraint → EK-14; subsystem EK-16
- 状态：S3 + 测试

### EK-14 fail-closed：解析失败条目 dropped 只收窄 allow；无规则策略不能授权任何东西
`internal/egresspolicy/egresspolicy.go`
Compile 对解析失败条目（旧 ateapi 接受的）dropped 并报告——只收窄 allow 规则，**fail closed**；RuleCount 注释："A policy with no rules can authorize nothing"。Egress PEP 未配置/失败即丢弃（security.md Met）。
- 证据：egresspolicy.go Compile 注释 + RuleCount 注释；security.md "Fail-Closed Policy-Enforced Egress Gateway: Met"
- links: constraint → EK-13; mechanism EK-21; subsystem EK-16
- 状态：S3 + 测试

### EK-15 NormalizeAuthority 规范 DNS/IP/端口，非法即 deny
`internal/egresspolicy/egresspolicy.go`
authority → Destination：split port、IP literal（IPv6 zone 拒绝）、DNS 名 lowercased 去尾点且校验 DNS-1123 subdomain；空/非法 → error，caller 应 deny。
- 证据：egresspolicy.go NormalizeAuthority/NormalizeHostname
- links: dependency → EK-12; constraint → EK-14; subsystem EK-16
- 状态：S3 + 测试

### EK-16 凭据注入错误映射：NotFound/PermissionDenied→403，瞬时失败→503 fail-closed
`cmd/atenet/internal/router/egress/credentials.go`
mapCredentialProviderError：NotFound/PermissionDenied → 403（重试无意义）；Unavailable/DeadlineExceeded → 503（fail-closed 可重试）；其余 → 403。injectionTimeout=4s（小于 ext_proc message_timeout，慢 provider 变可重试 503 而非 Envoy 504）。mapActorJWTError 同构。
- 证据：credentials.go 常量 + mapCredentialProviderError 注释
- links: mechanism EK-14（fail-closed 族）; dependency → EK-17; contrast EK-08（重试分类哲学一致：决定不重试 vs 瞬时可重试）; subsystem EK-11
- 状态：S3

### EK-17 身份来自 mTLS 证书而非请求头；filter state 由 Envoy 派生，隧道内不可写
`cmd/atenet/internal/router/egress/egress.go`
包注释："Identity comes from the actor certificate presented in the mTLS handshake, never from a request header. On the inner legs it arrives as filter state Envoy derived from that certificate, which nothing inside the tunnel can write."
- 证据：egress.go 包注释
- links: constraint → EK-16; mechanism EK-24; subsystem EK-11
- 状态：S3

### EK-18 零凭据沙箱：actor 沙箱内无凭据，egress 网关外注
`docs/security/security.md`（Zero Ambient Credentials: Met）+ `cmd/atenet/internal/router/egress/credentials.go`
Actor 沙箱默认完全无凭据执行；可持凭据标识/加密凭据，由 Substrate 基础设施带外水合/解密。credential provider 在 egress 网关注入（FetchSecret）。
- 证据：security.md Identity 表；credentials.go FetchSecret 调用
- links: mechanism EK-16; dependency → EK-17; subsystem EK-19
- 状态：S3（实现；注入路径有 e2e）

### EK-19 SPIFFE X.509 1h + on-demand JWT：组件 mTLS 全部覆盖，仅 /healthz /readyz 豁免
`docs/security/security.md`（Distinct Workload Identity: Met; Universal Component mTLS: Met）
每 actor 短时（1h）SPIFFE X.509 证书 + on-demand actor JWT；所有控制面/数据面组件 mTLS，仅本地无认证 health/liveness probes 豁免。UID 区分"删除后同名重建"。
- 证据：security.md Identity/组件表；atunnel/tunnel.go mTLS 参数
- links: mechanism EK-24; constraint → EK-17; subsystem EK-18
- 状态：S3

### EK-20 ate-system 控制面默认拒绝 ingress（NetworkPolicies）
`docs/security/security.md`（No Access by Default: Met）+ `manifests/ate-install/ate-system-networkpolicies.yaml`
控制面 pod 默认拒绝 ingress，只放行控制面所需流量；actor 默认拒绝网络（loopback/private netns/DNS 过滤）。
- 证据：security.md Network 表 + manifests 引用
- links: mechanism EK-14; subsystem EK-18; dependency → EK-21
- 状态：S3

### EK-21 双重快照授权：actor+atelet 双身份才能读写快照存储，guest 不能直连
`docs/security/security.md`（Dual-Party Snapshot Authorization: Met）
读写 snapshot 存储需双身份验证：actor 身份 + 代表其操作的节点 supervisor（atelet）。沙箱 guest 不能直接与 snapshot 存储交互。实现含 GCS per-actor token exchange（objectstoreplugin/exchange）。
- 证据：security.md Storage 表；internal/objectstoreplugin/exchange/ 存在
- links: mechanism EK-14; subsystem EK-22; constraint → EK-23
- 状态：S3（GCS）/Future（S3 per-actor tokens）

### EK-22 AssetPlugin 下载到私有 staging，size+sha256 校验后才复制给调用方
`internal/objectstoreplugin/asset.go`
AssetProvider 在 object storage 上服务：只读自己 backend scheme 的 URI（gs:// 在 S3 上读 gs:// bucket = 别人的 bucket，InvalidArgument 不读）；下载到 plugin 私有 stagingDir，size+sha256 校验通过（assetfetch.Stager）才复制——调用方永远读不到未 pin 的字节（如 snapshot 的）。
- 证据：asset.go 包注释 + 代码
- links: mechanism EK-14（untrusted storage 族）; dependency → EK-21; subsystem EK-23
- 状态：S3

### EK-23 OpenFGA ReBAC：bootstrapOwners 用 contextual tuple，不可经 API 撤销
`cmd/ateapi/internal/authz/authorizer.go`
Authorizer 是只读 PDP：对 Substrate store + 授权模型做 OpenFGA Check。bootstrapOwners（全局 owner）以 contextual tuple（每次 Check 附带）而非存储 tuple 持有——不在任何 AccessPolicy、不能经 API 撤销、服务器配置移除即失效。systemPrincipals 同理但仅 mTLS 认证的 principal 适用。TODO 明示：考虑存为 tuples 以便 API 可见（需 principal ID 按认证方法限定）。
- 证据：authorizer.go 注释全文
- links: mechanism EK-14（授权与撤销设计）; constraint → EK-20; subsystem EK-24
- 状态：S3

### EK-24 组件间全 mTLS：ateapi/atelet/ateworker/routers/tunnels 用 node/service 证明身份通信
`docs/security/security.md`（Universal Component mTLS: Met）+ 各组件 CredentialBundle 参数
- 证据：security.md；suspend.go TLSConfig/ateletdial.Dial；atunnel credential-bundle
- links: mechanism EK-19; subsystem EK-10/EK-23
- 状态：S3

## C 组：内存/资源/运行时

### EK-25 cgroup memwatch/memguard 内存守卫：接近限额触发主动 suspend
`internal/ateworker/cgroup/`（memwatch.go/memguard.go）
worker 内 actor cgroup 内存监控：接近限额时触发主动 suspend（idleness 信号族）。文件存在 + 测试覆盖。
- 证据：cgroup/ 目录 + actor.go/memwatch.go/memguard.go
- links: causal → EK-04（内存压力触发 suspend 请求）; subsystem EK-26; mechanism EK-08
- 状态：S3（实现+测试；集群级行为未实跑验证 S4 部分）

### EK-26 workercapacity：atecontroller 经 downward API 注入容量，worker 容器读取限额文件
`internal/workercapacity/workercapacity.go`
MountPath=/run/worker-capacity；CPULimitFile=cpu_millicores；MemoryLimitFile=memory_bytes。
- 证据：workercapacity.go 常量
- links: dependency → EK-25（内存守卫依赖限额）; subsystem EK-02; mechanism EK-01（调度输入）
- 状态：S3

### EK-27 memborrow：worker 间内存借用（跨 worker 内存复用）
`internal/ateworker/memborrow/`
目录存在（未深挖实现细节，见 Candidates C-3）。
- 证据：internal/ateworker/memborrow/ 目录
- links: subsystem EK-25; hypothesis → C-3
- 状态：S2（目录存在，实现未逐行确认）

## D 组：身份与认证细节

### EK-28 localjwtauthority + oidcjwt：本地 JWT 权威 + OIDC 发现
`internal/localjwtauthority/` + `internal/oidcjwt/`
身份子系统：本地 JWT 颁发 + OIDC 发现（Distinct Workload Identity 实现支撑）。
- 证据：目录存在 + security.md OIDC discovery Met
- links: mechanism EK-19; subsystem EK-29
- 状态：S3

### EK-29 clustertrustbundle + podidentity：集群信任链 + Pod 身份
`internal/clustertrustbundle/` + `internal/podidentity/`
组件/节点身份材料：credential-bundle 路径（podidentity.podcert.ate.dev）、trust bundle 分发。
- 证据：目录存在 + atunnel/tunnel.go credential-bundle 默认路径
- links: dependency → EK-24; subsystem EK-28
- 状态：S3

## E 组：运行时边界/验证/工具

### EK-30 ateworkervalidation：验证 worker 输入（生成代码 zz_generated.validation.go）
`internal/ateworkervalidation/`
validate.go + zz_generated.validation.go：worker 输入验证（生成式）。ateapi 侧 apivalidation 同模式。
- 证据：目录文件 + zz_generated 命名
- links: mechanism EK-31（验证生成模式）; subsystem EK-32
- 状态：S3

### EK-31 ateapi apivalidation：proto 注释驱动的生成式 API 校验
`cmd/ateapi/internal/apivalidation/`（actor.go/worker.go/atespace.go/egress_policy.go + zz_generated.validation.go）
proto 里 `+k8s:required/maximum/listMapKey` 等注释驱动生成校验（apivalidation）。
- 证据：proto 注释（ActorStatus 等）+ zz_generated.validation.go
- links: mechanism EK-30; constraint → EK-13（egress policy 校验与匹配用同一 parser）
- 状态：S3

### EK-32 e2e 框架以 backend/env/fixture 抽象集群差异
`internal/e2e/`（backend.go/env.go/fixture.go/pod.go/manifest.go）
e2e 用 backend 抽象（可能支持 kind/GKE），env 注入环境，fixture 组装 actor。AGENTS.md 要求 actor 功能双运行时（gVisor+microVM）e2e 覆盖。
- 证据：internal/e2e 文件清单 + AGENTS.md "Actor features work on both gVisor and microVM, with e2e coverage for both"
- links: subsystem EK-33; constraint → EK-01（生命周期行为有 e2e 锚）
- 状态：S3（测试框架实现）

### EK-33 benchmark 目录（benchmarking/）承载 20 万 agent / 500 activations/sec 目标验证
`internal/benchmarking/`（62 文件）+ cmd/benchmarking
基准承载北极星指标（README 数字）。文档 target vs benchmark 结果需区分（见 01 层声明）。
- 证据：internal/benchmarking 文件数；README 指标
- links: contrast EK-32; subsystem EK-01
- 状态：S2（存在；集群实测结果未在本考古范围验证）

### EK-34 atelet resetActorDirs：调度间隙清理节点临时卷/缓存（删除保证，In Progress）
`docs/security/security.md`（Delete Guarantee: In Progress）+ cmd/atelet
resetActorDirs 在 actor 调度周期之间 reset/scrub/sanitize 本地 ephemeral 卷与 worker 运行时环境，防跨租户泄漏。security.md 标记 In Progress。
- 证据：security.md Storage/Delete Guarantee 行；atelet resetActorDirs（checkpoint.go 注释亦提及）
- links: constraint → EK-14; subsystem EK-21; mechanism EK-22
- 状态：S3 部分实现（security.md 自标 In Progress）

### EK-35 Dynamic Quarantine：控制面可即时撤销 egress + 生命周期 halt
`docs/security/security.md`（Dynamic Quarantine: Met）
行为不端/受损 actor 可立即撤销 egress 授权 + 网络黑洞 + 生命周期 halt；Suspended 状态的 compromised actor 未来加 immutable cryptographic taint（Future）。
- 证据：security.md Auditability 表
- links: mechanism EK-23（授权撤销）; subsystem EK-18
- 状态：Met 部分 S3；Snapshot tainting Future

### EK-36 Structured Audit Trail：生命周期/快照/网关事件结构化遥测 + 凭据脱敏
`docs/security/security.md`（Structured Audit Trail: Met）
所有生命周期转换（start/suspend/resume/terminate）、snapshot 访问、gateway 路由事件、policy violations 生成结构化遥测，链接到加密 actor 身份与父 lineage；token log leaks 防 sanitization。实现含 OTLP Weaver registry。
- 证据：security.md Auditability 表；internal/otlprelay 存在
- links: subsystem EK-23; mechanism EK-14
- 状态：S3

## F 组：控制面细节/边界

### EK-37 ResumeActor 记录生命周期指标时跳过"已运行 no-op"
`cmd/ateapi/internal/controlapi/workflow_resume.go`（restoreTelemetry）
defer 记录 lifecycle metric 前判 `wasRunning`：router 按路由请求高频调 ResumeActor，记录已运行 no-op 会在 router QPS 采样并淹没 cold-resume 延迟。仅真实执行/失败记录。
- 证据：workflow_resume.go defer 注释
- links: contrast EK-01; subsystem EK-01; constraint → EK-38
- 状态：S3

### EK-38 快照共享：tag 快照 immutable 可共享（SHARED），actor 自有快照 private
`cmd/ateapi/internal/controlapi/workflow_resume.go`（snapshotSharing）
tag 的 snapshot 发布后 immutable，被从该 tag 创建的每个 actor 读取，atelet 可服务 node-local 缓存；actor 自有的下次 suspend 即被替换，永不共享。owner 未记录的按 private。
- 证据：workflow_resume.go snapshotSharing 注释
- links: mechanism EK-22; subsystem EK-01; constraint → EK-07
- 状态：S3

### EK-39 templateReplaced：snapshot 在不同 ActorTemplate 下捕获则按数据恢复
`cmd/ateapi/internal/controlapi/workflow_resume.go`
snap 的 ActorTemplateUid 与当前 template UID 不同 → template replaced；restoreTelemetry 注释：MEMORY snapshot 在 replaced template 下恢复以 data 输出（WireFidelity 描述请求的 restore 而非存储 scope）。
- 证据：workflow_resume.go templateReplaced + restoreTelemetry 注释
- links: causal → EK-38; subsystem EK-01
- 状态：S3

### EK-40 atelet 与 worker runtimes 互不透明：仅共享 ateworkerpb + nodepath
`AGENTS.md`（Code 节）
"Keep atelet and the worker runtimes opaque to each other. The only knowledge they may share is internal/proto/ateworkerpb and the paths in internal/nodepath... We want atelet to roll ahead of workers eventually... Until v1.0.0 nothing enforces this."
- 证据：AGENTS.md Code 节原文
- links: constraint → EK-01（生命周期跨组件协议受控）; mechanism EK-31; subsystem EK-32
- 状态：S3（文档硬规则）+ 部分执行（v1.0 前无强制）

---

### EK-41 仓库内置 AI 治理技能套件（.agents/skills/）：AI 只提案/草稿，人类批准/提交/撤销
`repo/.agents/skills/`（agents-md / detect-flaky-tests / post-draft-review / review-crds / security-status-report / triage-issues）
AGENTS.md 治理文化的执行工具层：detect-flaky-tests 自动开 issue+draft fix PR（跨 7 天 CI gVisor/microVM lanes）；post-draft-review 以 GitHub PENDING draft 评论提交（human 审后 submit）；triage-issues 硬规则"propose before applying"+ 每 run 生成 undo script；security-status-report 起子 agent 对 threats.json 逐条算质量分。统一不变量：**AI 产出可逆的提案/草稿，人类做批准/提交/撤销**（与 KO-05 双重授权同构）。
- 证据：.agents/skills/ 各 SKILL.md 原文
- links: mechanism EK-23（授权与撤销设计）; subsystem EK-40; contrast EK-35（治理闭环的 AI 执行侧）
- 状态：S3（skill 文件即实现）

---

## EK 边汇总统计
- 总 EK：41；含 links 的 EK：41（100%）
- 边类型覆盖：mechanism / subsystem / causal / dependency / constraint / contrast 全部出现
- 游离 EK：0
- 平均出边：~2.3（见 run_metadata.yaml）

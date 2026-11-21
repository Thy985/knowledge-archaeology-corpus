# Engineering Knowledge Graph — Agent Substrate（02 EK）

> 20 条 EK，每条含 mechanism / evidence（文件:证据）/ epistemic / links（六类边）。L1/L2 宽底座：推理原材料，不因"不够抽象"删除。
> 证据锚约定：[doc] 文档证据（设计意图）· [code] 代码/proto 证据（实现事实）· [test] 测试证据。

## EK-S01 actor→worker 多路复用 + 预启动热池
- mechanism: actor 不占用专用 Pod；空闲时快照出内存，WorkerPool 保持一批"available"worker，请求到来时快照恢复进热 worker（scheduler 选择 ready worker）
- evidence: [doc] docs/architecture.md "Resource Model/Scheduler"；[code] internal/proto/ateletpb/atelet.proto WorkerSupport/RegisterWorker（worker 注册进池）；[code] api/ateapi/v1/ateapi.proto Worker 记录（IDLE/BUSY）
- epistemic: Fact→Observation
- links: {causal: EK-S07→EK-S01（assignment 决定多路复用）, subsystem: EK-S02, contrast: EK-S19}

## EK-S02 快照分级 fidelity（VOLUMES/ROOTFS/MEMORY）
- mechanism: 每级包含下级；VOLUMES=持久卷 tar（冷启动回滚）；ROOTFS=+rootfs upper tar；MEMORY=整机（进程内存快照）
- evidence: [code] api/ateapi/v1/ateapi.proto SnapshotFidelity 枚举（注释"Each level includes the ones below it"）；[doc] docs/architecture.md "Sandbox Classes"；[code] cmd/ateworker-microvm/checkpoint.go 契约注释
- epistemic: Fact
- links: {mechanism: EK-S15, EK-S16, subsystem: EK-S01, constraint: EK-S19}

## EK-S03 checkpoint 幂等（RecordCheckpoint/ReuseCheckpoint）
- mechanism: `.checkpoint-done.json` 完成标记（tmp+rename 原子写）；重试在 fidelity 匹配 + 文件齐全时直接复用已完成 CheckpointWorkloadResponse，不再 checkpoint；VOLUMES 时 DataSnapshotFiles 与 SnapshotFiles 同集、ROOTFS/MEMORY 为严格子集
- evidence: [code] internal/ateworker/checkpoint.go RecordCheckpoint/ReuseCheckpoint（"so a retry after a caller timeout or atelet restart can reuse the finished response"）
- epistemic: Fact（[test] checkpoint_test.go 存在）
- links: {causal: EK-S05→EK-S03（async 上传中的重试依赖幂等）, mechanism: EK-S06, constraint: EK-S04}

## EK-S04 torn-file 防护（节点重启边界）
- mechanism: atelet 从不上传节点上次重启前写的本地快照副本（文件可能 torn）；从 `snapshot.object` 的 manifest 应答；无 manifest → NOT_FOUND；PromoteSnapshot 幂等（已 durable 即应答）
- evidence: [code] internal/proto/ateletpb/atelet.proto PromoteSnapshot 注释（"atelet never uploads a local copy written before the node last rebooted"）
- epistemic: Fact（设计契约，实现待 IVR 验证）
- links: {constraint: EK-S03, EK-S05, dependency: EK-S19}

## EK-S05 同步/异步 suspend 双模式 + locality
- mechanism: async（默认）SuspendActor 本地 checkpoint 写完即返回，durableSnapshot=PENDING，atelet 后台上传（--snapshot-upload-delay）报 COMPLETED；sync 先传后返；latest snapshot 有 locality 时 ResumeActor 从本地恢复；--snapshot-detach-delay 后 atelet 删本地文件清 locality/assignedNode（先传后删）
- evidence: [doc] docs/architecture.md "Actor Lifecycle"（同步/异步差别+detach 规则）；[code] api/ateapi/v1/ateapi.proto Snapshot.durable_snapshot 注释
- epistemic: Fact→Observation
- links: {causal: EK-S03, EK-S04, dependency: EK-S19, subsystem: EK-S01}

## EK-S06 RevertActor/DeleteActor 语义
- mechanism: revert 终止沙箱、丢弃节点本地/进行中快照、保留最后完成的外部快照回 SUSPENDED（crash 清除）；delete 终止容器、卸载卷、释放 assignment 后 GC 快照
- evidence: [doc] docs/architecture.md "Phase 4: Recovery"; [code] api/ateapi/v1/ateapi.proto ActorStatus.last_assigned_generation 单调不降（revert 也不回退保 stale 清理）
- epistemic: Fact
- links: {mechanism: EK-S03, causal: EK-S05→EK-S06}

## EK-S07 WorkerAssignment 反规范化（热路径免二次查询）
- mechanism: worker_pod_ips/worker_epoch 是 Worker 身份字段的副本；atenet 请求路由路径直接读 pod IP，不查 Worker 记录；字段不可原地改指另一 Worker；pass `worker` 才是 API 查询字段
- evidence: [code] api/ateapi/v1/ateapi.proto WorkerAssignment 注释（"denormalized snapshot…must not need a second lookup"）
- epistemic: Fact（设计契约）
- links: {causal: EK-S08→EK-S07（parking 后路由依赖 assignment）, subsystem: EK-S01}

## EK-S08 请求停车（request parking）
- mechanism: 入口 router 对 ResumeActor 的 ResourceExhausted/FailedPrecondition/Unavailable（+Aborted）视为可重试；请求进 parking lot（容量 --parked-request-max=1024，仅实际 park 才占位）；指数退避重试至成功或 budget（--parked-request-budget=5s）耗尽→原 503；ext_proc 超时=budget+5s 为持有上限；lot 满 → 立即 503（背压）
- evidence: [doc] docs/request-parking.md（Summary/Worst-case occupancy/Parking lot 段落）
- epistemic: Fact（文档+配置锚；实现待 IVR）
- links: {causal: EK-S01→EK-S08（池饱和触发停车）, constraint: EK-S07, dependency: EK-S10}

## EK-S09 egress fail-closed PEP（编译与评估）
- mechanism: Compile 解析模式+端口（旧 ateapi 接受的非法条目 dropped+reported → fail closed）；EvaluateRequest 无匹配 → Allowed=false（默认拒绝）；规则按 specificity 排序（exact > wildcard > any）择最优；SNI 决策解耦；纯包无 IO/日志
- evidence: [code] internal/egresspolicy/egresspolicy.go（Compile/EvaluateRequest/pattern rank 注释）
- epistemic: Fact（[test] egresspolicy_test.go 覆盖）
- links: {causal: EK-S09→EK-S10（放行才注入）, contrast: EK-S08（入口停车 vs 出口拒绝）, subsystem: EK-S14}

## EK-S10 凭据注入后丢弃（网关不可见）
- mechanism: 网关从 credential provider 取凭据注入请求头，**随后丢弃原始 secret**（不能读/快照/外泄）；可注入 actor JWT 替代存储 secret；仅 http/https 规则带 credential replacement effects 的放行注入
- evidence: [code] cmd/atenet/internal/router/egress/credentials.go（applyEffects/providerCredential/actorJWT——Reconciled 证据升级）+ [doc] docs/egress-credential-injection.md（"the gateway discards, so it cannot read, snapshot, or exfiltrate"）；"丢弃后不可读"子句仍为文档契约
- epistemic: Fact（文档契约；实现 cmd/atenet/internal/router/egress/credentials.go 存在）
- links: {causal: EK-S09→EK-S10, mechanism: EK-S12（凭据水合）, contrast: EK-S11}

## EK-S11 身份双轨（actor 自身 vs 组件代行）
- mechanism: KindActor sub=`actor/<atespace>/<name>`、claim 在 `actor.ate.dev`；KindWorkerForActor sub=`worker-for-actor/...`；依赖方可区分 actor 代码与组件代行并授不同权；ClaimsToWire 拒绝 KindUnspecified（防漏设签发）
- evidence: [code] internal/actoridjwt/*.go（Kind 注释/SubjectPrefix/ClaimName）
- epistemic: Fact
- links: {causal: EK-S11→EK-S10（JWT 注入用 actor 身份）, mechanism: EK-S13, contrast: EK-S12}

## EK-S12 零环境凭据 + credbundle 水合
- mechanism: 沙箱默认无凭据；凭据以"标识符/加密凭据/unprivileged token"形式经 Substrate 基础设施 out-of-band 水合；credbundle 读 K8s PodCert 文件（首 PEM=私钥，余=leaf→root 证书链），Loader 缓存按文件 mtime 变化刷新（rotation 握手间生效）
- evidence: [code] internal/credbundle/credbundle.go（Loader 注释）；[doc] docs/security.md "Zero Ambient Credentials"
- epistemic: Fact
- links: {mechanism: EK-S11, EK-S13, causal: EK-S12→EK-S10}

## EK-S13 组件 mTLS 全覆盖 + 1h SPIFFE 证书
- mechanism: 所有控制面/数据面组件（ateapi/atelet/ateworker/routers/tunnels）经 node/service-attested mTLS 通信；actor 得 1h SPIFFE X.509 + on-demand actor JWT；仅 /healthz /readyz 豁免认证；podcertcontroller 签发 Pod 证书
- evidence: [doc] docs/security.md "Universal Component mTLS"/"Distinct Workload Identity"; [code] cmd/podcertcontroller 存在 + internal/substratex509
- epistemic: Fact→Observation
- links: {mechanism: EK-S11, EK-S12, constraint: EK-S14}

## EK-S14 OpenFGA 授权（bootstrap owner + 角色）
- mechanism: 嵌入式 OpenFGA ReBAC；bootstrap owners（`--authz-bootstrap-owners`，ate-api-server 无则拒绝启动）恒持 global owner；owner/editor/viewer 于全局或 atespace；组件持固定 system roles（Pod 证书）；每个 RPC 先鉴权
- evidence: [doc] docs/authorization.md（bootstrap 段落 + 配置表）；[code] internal/ateapiauth + cmd/ateapi
- epistemic: Fact
- links: {subsystem: EK-S09, constraint: EK-S13, contrast: EK-S11}

## EK-S15 microVM 快照（CH demand-paging vs eager）
- mechanism: runningActor 直接拥有 CH 进程+api-socket；OnDemand restore 用 userfaultfd demand-paging 从 restoreSourceDir 分页取 RAM，checkpoint 时把 CH 新稀疏快照 overlay 到 base 上成 COMPLETE（否则丢 un-faulted 页）；eager restore 全读 extent → self-contained；baseID 跨 restore lineage 传播（golden id）；virtiofsd 统一 share（RO lower + per-actor upper + durable-dir 卷）host-backed
- evidence: [code] cmd/ateworker-microvm/run.go runningActor 注释（restoreSourceDir/preserveRestoreSource/snapshotIsSelfContained/baseID）；[code] cmd/ateworker-microvm/checkpoint.go 契约注释
- epistemic: Fact（[test] cmd/ateworker-microvm 测试存在）
- links: {mechanism: EK-S02, EK-S16, subsystem: EK-S01}

## EK-S16 gVisor 快照（runsc checkpoint + 网络恢复 workaround）
- mechanism: runsc 原生 checkpoint/restore 进程树；fs/ 与 rootfs/ 分开存；MEMORY 拆出进程 checkpoint；VOLUMES/ROOTFS 用 runsc fscheckpoint 冷启动恢复；`--allow-connected-on-save` flag 规避网络恢复 bug
- evidence: [doc] docs/architecture.md "Sandbox Classes"（gVisor 段落）；[code] cmd/ateworker-gvisor/（main.go/phaselog.go/stats.go）
- epistemic: Fact→Observation
- links: {mechanism: EK-S02, EK-S15, contrast: EK-S15（同问题两运行时）}

## EK-S17 sandbox 资产链（content-addressed 缓存 + sha256）
- mechanism: atelet 节点本地 content-addressed 沙箱二进制缓存，硬链接给 actor；预预热默认运行时；按字节预算淘汰（不删在用）；objectstore-plugin 先落盘再校验 sha256 拷贝进缓存（拒绝 snapshot 位置下 URL）；atelet 复验 sha256 且自身无对象存储凭据
- evidence: [doc] docs/architecture.md "Storage Mover"/"Sandbox Asset Cache"; [code] cmd/atelet/sandbox_asset_cache.go + snapshot_transfer.go
- epistemic: Fact→Observation
- links: {constraint: EK-S09（资产非凭据）, dependency: EK-S01, contrast: EK-S04}

## EK-S18 atelet/worker 互不透明 + 治理门
- mechanism: 物理 Pod 生命周期与沙箱 agent 进程解耦；atelet 与 worker 运行时仅共享 ateworkerpb+nodepath（v1.0.0 前不强制）；AGENTS.md：AI 贡献必须披露、拒绝 bot 账号、fail loudly（只忽略明确预期错误）、image pin digest、CI env 只在命名 fail-closed 谓词内读、新测试须无变更时失败
- evidence: [code] AGENTS.md（治理规则）；[doc] docs/architecture.md "Node Supervisor"
- epistemic: Fact
- links: {causal: EK-S18→EK-S01（解耦支撑复用）, constraint: EK-S19, subsystem: EK-S20}

## EK-S19 数据局部性 vs 任意节点恢复（含未解问题）
- mechanism: locality 优先本地恢复（免下载）；detach 后可任意节点恢复；**未解决面**（[doc] aspirational 声明）：架构文档明示大量机制未实现——Flow Edge 必须代码验证，文档不可当实现事实
- evidence: [doc] docs/architecture.md 首行 "Much of this architecture is aspirational, and is not yet implemented!"；[code] api/ateapi/v1/ateapi.proto Snapshot.locality 注释（"Detach, revert, or a later checkpoint clears it. Resume keeps the files."）
- epistemic: Observation→Hypothesis（未实现面为假设）
- links: {constraint: EK-S02, EK-S04, EK-S05, dependency: EK-S18}

## EK-S20 安全原则状态体系（Met/In Progress/Future 审计锚）
- mechanism: security.md 每原则带状态（Met=已实现/In Progress=部分/Future=路线图/Out of Scope=外部）；threat-model.md 声明状态不在表中（防 bias 自动化评估），CI `.agents/skills/security-status-report/` 动态评估；**关键状态**：Always Sandboxed/Per-Actor Storage Segregation/Network Isolation/Zero Ambient Credentials/Distinct Workload Identity/Universal mTLS/Untrusted Storage/Dual-Party Snapshot Auth/No Access by Default/Fail-Closed Egress/Structured Audit/Dynamic Quarantine = Met；Delete Guarantee/Node Blast Radius/Atespace Separation/Quotas = In Progress/Future
- evidence: [doc] docs/security.md 状态表；[doc] docs/threat-model.md（T-01 起 + 自动化评估声明）
- epistemic: Fact（状态声明是事实；**Met 声明是否属实须代码验证——IVR 攻击点**）
- links: {constraint: EK-S09, EK-S13, subsystem: EK-S18}

## EK Graph 健康检查（v3.1 门）
- 平均出边：20 条 EK 共 45 条边 ≈ 2.25/条（≥1 达标）
- 游离 EK：0（20/20 有 links）
- 聚合规则覆盖率：待 KO 声明后校验

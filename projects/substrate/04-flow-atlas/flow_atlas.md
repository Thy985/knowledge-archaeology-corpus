# Flow Atlas — Agent Substrate（04 七类流）

> 从真实代码/文档导出；关键 Edge 可回溯 symbol/file/condition。标记：已代码验证 [code] / 文档声明 [doc] / 待验证 [pend]。

## 1. Control Flow（控制流）：请求→唤醒→路由
```
Client HTTP (ate-target-actor: <atespace>/<actor>)
  → atenet-router Envoy ext_proc [code: cmd/atenet/internal/router/extproc/dispatch.go]
  → ResumeActor(atespace, name) [proto: Control.ResumeActor]
  → [pend] atelet Restore → ateworker RestoreWorkload [proto: Atelet.Restore / Worker.RestoreWorkload]
  → worker assignment 返回 → atunnel mTLS CONNECT [code: internal/atunnel/tunnel.go :8443；router 侧 --port-https=8443 / --port-connect-tls=8444（cmd.go:63,70）]
  → actor 私有 veth → actor
```
- Edge C1: `ResumeActor` 可重试码（ResourceExhausted/FailedPrecondition/Unavailable/Aborted）→ 请求停车 [doc: request-parking.md]
- Edge C2: **Host 不是 actor 选择者**——`ate-target-actor` header 才是目标权威 [doc: architecture.md Ingress Routing]
- Edge C3: 任意端口 = HTTP CONNECT authority 命名端口（仅 HTTP(S)；raw TCP 不支持）[doc]
- Edge C3b (Reconciled): legacy plaintext 路径——`--upstream-connect=false` 且无 credential bundle 时可 dial pod-IP:80（cmd.go:77 注释）；默认关闭（mTLS 必须），非绝对 [code]

## 2. State Flow（状态流）：ActorState 七态
```
SUSPENDED --ResumeActor--> RESUMING --restore/boot--> RUNNING --SuspendActor--> SUSPENDING --checkpoint--> SUSPENDED
SUSPENDED/CRASHED --RevertActor--> SUSPENDED（保留最后 durable；crash 清除）
任意态 --DeleteActor--> DELETING --> GC
```
- Edge S1: `last_assigned_generation` 单调不降（revert 也不回退——stale 快照清理依据）[code: proto ActorStatus]
- Edge S2: `worker_assignment` 只能 set/clear 不能原地改指另一 worker [code]
- Edge S3: CRASHED 原因记录 crash.message/crash_time [code]

## 3. Data Flow（数据流）：快照旅程
```
checkpoint（ateworker）→ checkpointDir（.checkpoint-done.json 完成标记）→ atelet 上传（async: PENDING 后台上传 / sync: 先传后返）→ object storage
→ locality（本地保留）→ ResumeActor 本地优先恢复 → detach（--snapshot-detach-delay）→ 清 locality + assignedNode
→ tag（不可变副本+保留钉）→ PUBLISHED 跨 atespace
```
- Edge D1: **torn-file 防护**：节点重启前的本地副本永不上传（manifest 应答）[code: atelet.proto PromoteSnapshot]
- Edge D2: VOLUMES 时 DataSnapshotFiles == SnapshotFiles；ROOTFS/MEMORY 时严格子集 [code: checkpoint.go ReuseCheckpoint]
- Edge D3: 首次恢复用 ActorTemplate 的 golden snapshot（版本 0）[doc]

## 4. Evidence Flow（证据流）：遥测与归属
- OTLP Weaver registry（Structured Audit Trail）+ 凭据 token 日志清洗（credential redaction）[doc: security.md]
- GetWorkloadStats 语义分裂：NOT_FOUND=该 worker 没在执行该 actor（mapping 过期，重试无益）vs FAILED_PRECONDITION=执行中但暂无采样（瞬态，skip 下一拍）——身份从接受 actor 时刻保留，boot 期死亡可归属 [code: ateworker.proto 注释]
- GetActiveWorkloadStats=枚举读（无断言身份；采样者必须只从响应内 echo 的 identity 归属，不从自己的 mapping）[code]
- Edge E1: benchmark 遥测量测路径（telemetry/README）[doc: benchmarking/README]

## 5. Authority Flow（权威流）：信任链
```
bootstrap owner（--authz-bootstrap-owners；缺失拒绝启动）→ OpenFGA ReBAC（owner/editor/viewer @ global|atespace）
组件 Pod 证书（podcertcontroller，spiffe://）→ 组件 mTLS（唯一豁免 /healthz /readyz）
actor 身份：1h SPIFFE X.509 + on-demand JWT（KindActor vs KindWorkerForActor 双轨）
ingress 客户端：静态 SPIFFE allowlist（ATE_INGRESS_ALLOWED_SPIFFE_IDS）
```
- Edge A1: 组件代行与 actor 自身主体分离（sub 前缀不同）[code: actoridjwt]
- Edge A2: 凭据注入后丢弃（网关不可读）[doc: egress-credential-injection.md]

## 6. Memory Flow（记忆流）：双存储体系
- 声明式低频：WorkerPool/SandboxConfig → K8s CRD（atecontroller 调和 Deployment）
- 高频态：Actor/ActorTemplate/Worker/Tag/Atespace → PostgreSQL（squirrel 构建 SQL）
- Edge M1: WorkerAssignment 反规范化 = PostgreSQL 读优化（热路径免二次查询）[code: proto]
- Edge M2: actor 内存+磁盘合一版本化快照（与 ActorTemplate 版本绑定，保证恢复一致性）[doc]

## 7. Policy Flow（策略流）：治理闭环
```
egress 策略定义（EgressPolicy 资源）→ Compile（非法条目 drop+report=fail closed）→ EvaluateRequest（默认拒绝+specificity 排序）
→ 放行规则带 effects（credential replacement）→ 凭据注入
ingress：SPIFFE allowlist + parking 预算（5s）+ lot 容量（1024）
治理：AGENTS.md（AI 披露/fail loudly/pin digest/CI fail-closed 谓词）+ security.md 状态体系 + threat-model CI 评估
```
- Edge P1: **决策→执行→记录闭环**：EgressPolicy 创建（API）→ PEP 评估（egresspolicy.go）→ 注入（router egress）→ 审计（structured telemetry）
- Edge P2: Dynamic Quarantine（运行中撤销 egress + lifecycle halt）[doc: security.md Met]
- Edge P3: bypass/override 路径：`--upstream-connect=false`（Envoy 直连 443 绕过 CONNECT 路径）[doc: architecture.md]；S3 侧凭据注入标注 Future（per-actor token on S3 pending）[doc]

## Flow→KO 交叉校验门
- KO-S01 回溯：Control C1/C2 + Data D1/D2/D3
- KO-S02 回溯：Authority A1/A2 + Policy P1/P2
- KO-S04 回溯：Control C1 + State S2
- KO-S06 回溯：Data D1/D2 + State S3
- 无 KO 与 Flow 矛盾（未发现）；[pend] 标记项为文档声明，需 IVR 代码验证

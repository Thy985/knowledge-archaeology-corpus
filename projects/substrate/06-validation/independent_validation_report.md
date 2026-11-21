# Independent Validation Report — ARCH-2026-10-26-001（盲重建）

> Auditor 独立重读仓库（不把主考古当事实来源），仅凭代码/proto/文档原文建立 Independent Findings 后与主考古对比。
> 判定：CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / OVER_GENERALIZED / MISSING / CONTRADICTED / NEEDS_HUMAN_REVIEW

## 判定统计
| 判定 | 数量 |
|---|---|
| CONFIRMED | 11 |
| PARTIALLY_CONFIRMED | 2 |
| CONTRADICTED | 1 |
| MISSING | 4 |
| NEEDS_HUMAN_REVIEW | 1 |
| DOWNGRADED | 0 |
| OVER_GENERALIZED | 1（主考古 C-S05 跨项目假说范围——明确标注后降为 candidate，无冲突） |

## 3 项成功（主考古核心声明被代码实锤）

**S1. WorkerAssignment 反规范化热路径（主考古 EK-S07）→ CONFIRMED**
- 盲重建：`cmd/atenet/internal/router/ingress/ingress.go:128` `if ips := assignment.GetWorkerPodIps(); len(ips) > 0` —— 请求路由路径直接读 pod IP，无二次 Worker 查询；proto 注释与实现一致。

**S2. 请求停车（主考古 EK-S08 / KO-S04）→ CONFIRMED**
- 盲重建：`cmd/atenet/internal/router/config.go` `ParkedRequest ingress.ParkedRequestConfig` + `extProcMaxRequestsFloor=1024`（config.go:230）+ `drainTimeoutMargin=5s`（:250，镜像文档"budget+5s"）+ 启动校验 `--extproc-max-requests >= --parked-request-max`（:288，熔断器不得截断 parking lot）；parking 实现在 extproc.go/ingress。
- 附加：文档"lot 满即 shed 503"与 config 校验语义一致。

**S3. checkpoint 幂等（主考古 EK-S03）→ CONFIRMED**
- 盲重建：`internal/ateworker/checkpoint_test.go` TestRecordAndReuseCheckpoint / TestReuseCheckpointDataSnapshotFilesByFidelity——完成标记写入/复用/按 fidelity 区分 DataSnapshotFiles 均有测试。

**附带确认（主考古相关声明）：**
- S4. atunnel 双监听：tunnel.go:84 `:443`（HTTPS）+ tunnel.go:85 `:8443`（mTLS CONNECT）→ CONFIRMED
- S5. DNS 过滤（No Access by Default）：internal/atunnel/dns/{dnshandler,evaluator,relay,policy,limiter} → CONFIRMED
- S6. 1h X.509：cmd/ateapi/internal/workerservice/certificate.go:101 + controlapi/actor.go:660 `NotAfter: now+1h` → CONFIRMED
- S7. microVM demand-paging：cmd/ateworker-microvm/internal/ch/{merge,prefault,restorefds}.go（userfaultfd）→ CONFIRMED
- S8. 身份双轨：internal/actoridjwt/actoridjwt.go:130 ClaimsToWire + KindUnspecified 拒绝 → CONFIRMED
- S9. egress 策略接线：cmd/atenet/internal/router/egress/policycache.go:145 `egresspolicy.Compile` + egress.go:209 SNIRules → CONFIRMED
- S10. Tag PUBLISHED 跨 atespace：controlapi/actor.go:188-190 显式拒绝非 published → CONFIRMED
- S11. 凭据注入实现存在：egress/credentials.go applyEffects/actorJWT/providerCredential → CONFIRMED（主考古标 [doc]，实为 [code] 可定位——见 E3）

## 3 项错误/修正（主考古需 Reconciliation）

**E1. Worker pod port 80 的 legacy plaintext 路径（主考古 04 C3/Flow 未覆盖）→ CONTRADICTED（补正）**
- 盲重建：`cmd/atenet/internal/router/cmd.go:77` 注释——`--upstream-credential-bundle` 为空时 "disables upstream mTLS (**legacy plaintext pod-IP:80**) and needs --upstream-connect=false"。
- 主考古 Flow C3 断言"Worker pod port 80 不是直接 Actor ingress 路径"（文档口径）。**代码显示该路径在显式关闭 mTLS 时存在**——bypass/legacy path 修正：默认关闭（upstream-connect=true + bundle 必须），但 legacy plaintext 可达，非绝对。
- Reconciliation：Flow Atlas Edge C3 追加 legacy 条件；security.md "Met (mTLS)" 声明限定默认配置下成立。

**E2. router 侧双端口遗漏（主考古 06 未列）→ MISSING（补入）**
- 盲重建：router cmd.go:63 `--port-connect-tls=8444`（CONNECT-tunneled 流量入口）+ cmd.go:70 `--port-https=8443`（HTTPS workload）——router 数据平面与 atunnel 侧端口号不同（router 8443=HTTPS；atunnel 8443=CONNECT）。
- Reconciliation：04_flow_atlas Control Flow 补 router 侧端口维度；01_project_map 网络段补两套端口。

**E3. EK-S10 证据级别升级 → DOWNGRADED→CONFIRMED（修正证据引用）**
- 主考古 EK-S10 标 [doc]（egress-credential-injection.md）。盲重建定位 [code] `cmd/atenet/internal/router/egress/credentials.go`（applyEffects/providerCredential/actorJWT）——注入路径是代码事实。
- Reconciliation：EK-S10 evidence 改为 [code]+[doc]；"丢弃后不可读"仍为文档契约（网关内存处置细节未深读）→ 该子句保留 [doc]。

**E4. EgressPolicy 作用域修正 → PARTIALLY_CONFIRMED**
- 主考古 01 说"EgressPolicy CRUD"笼统。盲重建 proto：actor 级 GetActorEgressPolicy/CreateActorEgressPolicy + Global/Atespace AccessPolicy 并存。主考古未错但精度不足——Reconciliation 补 actor 级定位。

## OVER_GENERALIZED 自查
- 主考古 KO-S05（凭据外注三仓收敛）依赖 C-S05 跨项目假说——单项目+次级证据，已标 Cross-project validation pending，未冒充 Principle → 保持（不降级）。

## MISSING（Auditor 覆盖缺口，主考古 06 已显式声明）
- M1. cmd/ateapi 控制面实现体（scheduler/OpenFGA 集成/workerservice）未精读
- M2. internal/atunnel/egress 全实现（非 TCP 丢弃的确切代码路径未定位——IV-15）
- M3. internal/localjwtauthority 主实现（621 行仅见测试面）
- M4. docs/authentication.md 全文（SPIFFE 配置细节）

## NEEDS_HUMAN_REVIEW
- security.md "Met" 声明群（Dual-Party Snapshot Authorization 的 GCS per-actor token exchange 机制细节、Delete Guarantee 的 resetActorDirs 覆盖度）——需人工对照 atelet 代码逐项确认；threat-model CI 自动评估 `.agents/skills/security-status-report/` 的存在与运行方式待验证。

## Benchmark case（学习价值）
- **BC-1** 反规范化热路径：WorkerAssignment 副本免二次查询——PostgreSQL 高频读优化的"一致性换延迟"权衡（副本不可原地改，set/clear 原子替换；更新时机=RUNNING 转换）
- **BC-2** 请求停车 vs 立即失败：parking lot 背压 + budget 回退 + ext_proc 熔断器不截断 lot 的启动校验——资源弹性设计模板
- **BC-3** 愿望/实现分离制度化：architecture.md aspirational 声明 + security.md Met/In-Progress/Future 状态表 + threat-model 拒绝固化状态（防 bias 自动化评估）——AI 时代大规模代码库治理模型

## Reconciliation 修正汇总（将随 Corpus Commit 落盘）
1. 04_flow_atlas Edge C3 追加 legacy plaintext pod-IP:80（--upstream-connect=false）条件
2. 04_flow_atlas Control Flow 补 router 侧 --port-https=8443 / --port-connect-tls=8444
3. EK-S10 evidence 升级 [doc]→[code]+[doc]
4. 01_project_map EgressPolicy 作用域细化（actor 级 + AccessPolicy）
5. run_metadata validation_result 更新（independent_validation: CONFIRMED 11 / PARTIAL 2 / CONTRADICTED 1 / MISSING 4 / NEEDS_HUMAN_REVIEW 1）

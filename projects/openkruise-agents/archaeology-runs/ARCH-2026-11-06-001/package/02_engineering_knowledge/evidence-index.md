# Evidence Index（E-01 ~ E-68）

> 编号用于 `02_engineering_knowledge` 与 `03_knowledge_layer` 的 `evidence` 字段。所有路径均相对仓库根（commit `5715d3ef3b`）。强度：S3=已实现（代码可见）；S4=有自动化测试覆盖；S2/D=设计意图/文档。
> 计数说明：本文件 68 条；行号以本次读取的 commit 为准（浅克隆 HEAD）。

## A. 架构与规模

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-01 | pkg 非测试 Go 代码 232,348 行 | `wc -l`（snapshot 采集） | S3 |
| E-02 | controller 子系统 74,203 行 | `wc -l` | S3 |
| E-03 | 283 个 `*_test.go` | `find . -name '*_test.go' \| wc -l` | S3 |
| E-04 | 8 个 cmd 二进制（controller/manager/gateway/cert-init/agent-runtime/identity-provider/commit-job/okactl） | `ls cmd/` | S3 |
| E-05 | 7 个控制器集中注册：sandbox、sandboxset、sandboxclaim、sandboxupdateops、securitytokenrefresh、commit、poolautoscaler | `pkg/controller/controllers.go:42-66` | S3 |
| E-06 | sandbox-manager 部署中 recycle 视为不存在；独立 controller 保留 recycle | `AGENTS.md:14-26` | D |
| E-07 | 依赖方向 API→Manager→Infra，禁止反向/捷径 | `AGENTS.md:47-60` | D |
| E-08 | controller 依赖闭包不得包含 `pkg/servers/**` 或 `pkg/sandbox-manager/**` | `AGENTS.md:61-66` | D |

## B. 生命周期与 Reconcile

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-09 | Sandbox 9 态：Pending/Running/Paused/Resuming/Upgrading/Succeeded/Failed/Recycling/Terminating | `api/v1alpha1/sandbox_types.go:504-530` | S3 |
| E-10 | Reconcile 入口与整体顺序 | `pkg/controller/sandbox/sandbox_controller.go:233` | S3 |
| E-11 | 缓存落后时不基于陈旧缓存算状态，改用短重试（避免 no-op transition 推迟 wake） | 同上 `:291-307` | S3 |
| E-12 | phase 分派到各 `Ensure*`；default 记录 invalid phase 并 requeue | 同上 `:403-459` | S3 |
| E-13 | auto-pause 在 `calculateStatus` 与 `Ensure*` 之后执行，保证本轮 probe 条件可用 | 同上 `:467-478` | S3 |
| E-14 | 完成态短路 `isSandboxCompletedPhase` | 同上 `:682-685` | S3 |
| E-15 | 每轮统一 `updateSandboxStatus` | 同上 `:713` | S3 |
| E-16 | `checkTimers` 处理 ShutdownTime/PauseTime | 同上 `:1130` | S3 |
| E-17 | `handlePauseTimeout` 到期触发暂停 | 同上 `:1162` | S3 |
| E-18 | `handleShutdownTimeout` 到期删除，不等待 probe | 同上 `:1203` | S3 |
| E-19 | `calcTimeoutRequeue` 计算下一次定时器 | 同上 `:1231` | S3 |
| E-20 | `resolveRetentionAnnotationOrDefault` 解析 paused-retention | 同上 `:1249` | S3 |

## C. Pause / Checkpoint

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-21 | Paused 前置门：finalizer、Paused condition、PrePause hook、已有 checkpoint 等待、已暂停短路 | `sandbox_controller.go:506-575` | S3 |
| E-22 | 外部 Checkpoint 等待：只认 Pending/Creating 且未超时；多个取最新作 blocker | 同上 `:576-655` | S3 |
| E-23 | Resume 收尾：移除 Paused condition、清 probe 驱动状态、移除 finalizer | 同上 `:656-669` | S3 |
| E-24 | `EnsureSandboxPaused` 注释与实现：commonControl 只支持 Stop | `core/common_control.go:302-306` | S3 |
| E-25 | `ensureStopPaused`：pod nil→Paused=True；删除中→等待；否则删除 Pod（grace 5s） | `core/util.go:90-131` | S3/S4 |
| E-26 | `ensureCheckpointPaused` 存在但唯一调用者是测试（生产零调用） | `core/util.go:138-166`；`core/util_test.go:1099` | S3/S4 |
| E-27 | `getControl` 忽略 Pod 参数，恒返回 commonControl | `pkg/controller/sandbox/sandbox_control.go:25-27` | S3 |
| E-28 | `PauseStrategyHibernate` 在 pkg 生产代码零引用（仅类型/deepcopy/测试） | `grep -rn PauseStrategyHibernate --include=*.go pkg` | S3 |
| E-29 | Pause 内容映射：filesystem/memory；无 dump 内容则 podInfo；dump 不与 podInfo 并用 | `core/checkpoint.go:80-96` | S3 |
| E-30 | `AssumePodCheckpointed`：reason 白名单、镜像校验、创建/轮询/成功/失败/等待 | 同上 `:112-188` | S3 |
| E-31 | 新内容派生 label 优先、legacy v0.5.22 label 回退 | 同上 `:189-214` | S3 |
| E-32 | 镜像校验只比对同名容器；init 容器仅校验 RestartPolicy=Always 的 sidecar | 同上 `:399-427` | S3 |

## D. Upgrade

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-33 | 升级策略空=不支持；枚举 Recreate/CheckpointRestore | `api/v1alpha1/sandbox_types.go:158-174` | S3 |
| E-34 | 升级状态机：PreUpgrade→Checkpointing→UpgradePod→PostUpgrade→Resuming→ResumeSucceed | `core/upgrade_control.go:116-353` | S3 |
| E-35 | 非 CheckpointRestore 直接 short-circuit；CheckpointRestore 固定 filesystem | `core/checkpoint.go:365-398` | S3 |
| E-36 | Recreate 升级完成后重新 Get Pod（旧对象 stale） | `core/upgrade_control.go:387-505` | S3 |
| E-37 | ResumeSucceed：trigger 注解被提前移除则放弃升级回 Running；否则等模板 patch | 同上 `:315-330` | S3 |
| E-38 | ResumingFromFailedStep gate 默认 true，从失败步骤续跑 | `pkg/features/features.go:53-56` | S3 |

## E. Auto-Pause / Auto-Resume

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-39 | 激活条件：gate on + 至少一条 probe 规则；`OnIngressTraffic` 不激活控制面循环 | `auto_pause.go:77-98` | S3 |
| E-40 | `handleAutoPause`：删除中/未领取池→跳过；仅 Running/Paused 做决策 | 同上 `:115-195` | S3 |
| E-41 | `calculatePauseTime` 多条 fail-closed：ProbeValid=False、缺字段、负阈值、无效 regex、条件非 True、消息不匹配 | 同上 `:246-334` | S3 |
| E-42 | `calculateResumeTime`：probe 条件必须 True；解析失败静默（哨兵值）；结果减 leadTime | 同上 `:420-469` | S3 |
| E-43 | 空 TimeFormat=unix；unix 必须 >0；默认 leadTime 5min；负 lead 视为未设 | 同上 `:471-509` | S3 |
| E-44 | `tryPause`：未到点 requeue；到点 patch paused + AutoPaused 事件 + 清 NextPauseTime | 同上 `:196-230` | S3 |
| E-45 | `tryResume`：必须 Paused condition=True；patch paused=false + 清 NextResumeTime | 同上 `:593-636` | S3 |
| E-46 | `pauseSandbox`：幂等；托管 retention 时同时改写 ShutdownTime 与 PauseTime；乐观锁 patch | 同上 `:637-671` | S3 |
| E-47 | MessageRegex 进程级缓存按 pattern 为 key，上限 128；超过即整体重置 map（非 LRU），防止无限增长 | 同上 `:42-64`（`maxCompiledMessageRegexes=128`） | S3 |
| E-48 | 至少一条规则（pause probed / resume probed / resume ingress）必填 | `pkg/autopause/validation.go:162-169` | S3 |

## F. PoolAutoscaler

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-49 | scale-up cooldown 下限 = `sbxMaxPendingTimeout + 10s`（等 Pod 创建完成） | `pkg/controller/poolautoscaler/capacity.go:35-58` | S3 |
| E-50 | 决策顺序：cron 优先 → capacity；死区内不动；in-progress guard 防反馈回路 | 同上 `:211-313` | S3 |
| E-51 | 百分比 target+tolerance 先合成再乘 base；混合配置 tolerance 锚定 resolved target（防死区随池扩大） | 同上 `:484-511` | S3 |
| E-52 | stabilization window：窗口未稳定前不回退 | 同上 `:356-410` | S3 |
| E-53 | 观测窗口 30s、采样间隔 5s；monitor 按 PA UID/targetRef 失效 | 同上 `:59-64,113-183,445-457` | S3 |
| E-54 | PoolAutoscaler Spec/Status 字段（scaleTargetRef/max/min/cron/capacity/suspend） | `api/v1alpha1/poolautoscaler_types.go` | S3 |

## G. Gateway / Wake

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-55 | UID 匹配：同名重建或已删除对象不触发唤醒 | `pkg/sandbox-gateway/wake/wake.go:70-96` | S3 |
| E-56 | 唤醒复用 `sandboxcr.Sandbox.Resume`；autoPause=`PauseTime!=nil`；ShutdownTime 置 now+100 年 | 同上 `:120-160` | S3 |
| E-57 | 仅 PauseTimeout>0 才重设 PauseTime；应用 resume timeout floor 防中途再暂停 | 同上 `:157-190` | S3 |
| E-58 | 唤醒后本地 route upsert + peer 同步，失败只记日志 | 同上 `:200-206` | S3 |
| E-59 | JWT 配置不可变：重复 Configure 必须一致，否则报错 | `pkg/sandbox-gateway/jwtauth/manager.go:129-176` | S3 |
| E-60 | verifier 异步加载 + backoff；Ready 前状态机不提供 verifier | 同上 `:193-268` | S3 |

## H. Quota / Manager

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-61 | anti-drift 仅 leader 修复；周期 + 事件双通道 | `pkg/sandbox-manager/quota/antidrift.go:133-240` | S3 |
| E-62 | Phase 1：live sandbox→backend 补账，`Enforce=false`（不因补账拒绝已运行沙箱） | 同上 `:306-360` | S3 |
| E-63 | Phase 2：backend 多余条目需 Health + grace 后释放，防缓存滞后误删 | 同上 `:360-395` | S3 |
| E-64 | `TryClaimSandbox` 领取入口（pick/lock/post-process） | `pkg/sandbox-manager/infra/sandboxcr/claim.go:249` | S3 |
| E-65 | claim 后处理：锁、配额、路由、token 等 | 同上 `:419` | S3 |
| E-66 | E2B clone 创建 Checkpoint + SandboxTemplate（strip 源 Pod 身份 label，OwnerReference 级联删除） | `pkg/sandbox-manager/infra/sandboxcr/clone.go:610-690` | S3 |

## I. 文档与设计意图

| ID | 事实 | 来源 | 强度 |
|---|---|---|---|
| E-67 | CHANGELOG v0.6.0 写 `PauseStrategy (Stop / Snapshot / CloudDisk)`；代码无 Snapshot/CloudDisk | `CHANGELOG.md:38` | D |
| E-68 | Roadmap：`Experimental support for pause/checkpoint with filesystem persistency [~] In progress` | `roadmap.md:12` | D |

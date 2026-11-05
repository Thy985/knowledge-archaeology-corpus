# 00 Overview — OpenKruise Agents 知识考古

- **run_id**: `ARCH-2026-11-06-001`
- **项目**: openkruise/agents（CNCF OpenKruise 子项目）
- **仓库**: https://github.com/openkruise/agents.git
- **commit**: `5715d3ef3b9f60e603fc51fa1ece8b3c08a0e033`（master，2026-11-05 15:04:15 +0800，v0.6.0 线上版本）
- **skill**: knowledge-archaeology v3.2（生产版）
- **mode**: initial（首次考古）
- **日期**: 2026-11-06（Asia/Shanghai）
- **规模**: 1,069 Go 文件 / 267,683 行（pkg 232,348；controller 74,203）；71 个生成文件；283 个 `*_test.go`
- **考古范围**: 控制面（Sandbox/Checkpoint/PoolAutoscaler/SandboxUpdateOps）+ 生命周期主链路（Pause/Resume/Upgrade/Recycle）+ 数据面（sandbox-manager 路由/配额/身份、sandbox-gateway 唤醒、E2B 兼容层）。**未逐文件覆盖 232k 行 pkg**，以机制为单位取证。

## 1. 一句话定位

OpenKruise Agents 是 **在 Kubernetes 上运行 AI Agent 工作负载的沙箱基础设施**：用 CRD + operator 管理沙箱生命周期（创建/暂停/恢复/升级/回收），用数据面服务（sandbox-manager / gateway / agent-runtime）提供会话路由、身份令牌、配额与 E2B 兼容 API。

## 2. 本次考古的核心问题与答案

| 问题 | 回答（均可回溯代码） |
|---|---|
| 沙箱生命周期如何建模？ | `SandboxPhase` 9 态（Pending/Running/Paused/Resuming/Upgrading/Succeeded/Failed/Recycling/Terminating），由 `sandbox_controller.go:507-530` 定义；Reconcile 按 phase 分派 `Ensure*`（:403-452）。 |
| 暂停/恢复的真实实现是什么？ | `commonControl.EnsureSandboxPaused` 只调用 `ensureStopPaused`（删 Pod，PauseStrategy=Stop）；`Hibernate` 类型在 `api/v1alpha1` 有定义但**生产代码零引用**；`ensureCheckpointPaused` 只有测试调用（`util_test.go:1099`）。→ 代码事实，非文档声明。 |
| Checkpoint 在哪里真正被使用？ | ① E2B snapshot/clone：`sandbox-manager/infra/sandboxcr/clone.go:610-660` 创建 Checkpoint CR + SandboxTemplate；② 升级 `CheckpointRestore`：`core/checkpoint.go:365-398` 固定 filesystem 内容。Pause 路径未接线。 |
| 自动暂停/恢复怎么决策？ | `auto_pause.go`：probe 条件 + MessageRegex + ThresholdDuration 计算 pauseTime；resume 解析 probe 时间戳并减 leadTime（默认 5min）；`OnIngressTraffic` 由 gateway 的 `wake.go` 执行。多条 fail-closed 分支（ProbeValid=False、probe 不健康、非 True 条件）。 |
| 池如何扩缩？ | `PoolAutoscaler`：cron 优先 → capacity 水位（target ± tolerance 死区）→ 上下界 clamp；观测窗口 30s/采样 5s；scale-up cooldown 下限 = `sbxMaxPendingTimeout + 10s`；in-progress guard 防反馈回路（`capacity.go:211-316,484-511`）。 |
| 唤醒如何避免"刚唤醒又被删/又暂停"？ | `gateway/wake/wake.go:120-190`：ShutdownTime 置为 now+100 年（`ForeverReservePausedSandboxDuration`）；仅当 PauseTimeout>0 才重设 PauseTime，否则清空 stale PauseTime；应用 resume timeout floor。 |
| 配额如何防漂移？ | `quota/antidrift.go:306-395`：仅 leader 修复；observed→backend 立即补账（Enforce=false）、backend→observed 仅在 Healthy 且过 grace 后释放；缓存滞后不误删。 |
| 控制面与数据面边界如何治理？ | `AGENTS.md`：API(`pkg/servers`) → Manager(`pkg/sandbox-manager`) → Infra(`infra/`)，禁止反向依赖；controller 依赖闭包不得包含 servers/manager；sandbox-manager 部署中 recycle 视为不存在。 |

## 3. 关键发现（按价值排序）

1. **P1｜Pause 的 Hibernate/Checkpoint 路径未接线（代码事实）**：`PauseStrategyHibernate` 仅出现在类型定义/deepcopy/测试；`ensureCheckpointPaused` 无生产调用者；`getControl` 永远返回只支持 Stop 的 `commonControl`。CHANGELOG v0.6.0 声称 `PauseStrategy (Stop/Snapshot/CloudDisk)`，但代码枚举是 `Stop|Hibernate`，且 `CloudDisk` 全仓仅 CHANGELOG 出现。→ 文档声明 ≠ 实现事实（D 级文档 vs S3 实现）。
2. **P1｜Checkpoint 的实际价值路径是 E2B clone/snapshot 与升级，而非 pause**：`clone.go` 用 Checkpoint 生成可复用 SandboxTemplate（stripPodIdentityLabels）；升级 `CheckpointRestore` 只保 filesystem（可写层），不保 memory。→ "能力存在"与"能力被哪条产品路径消费"是两件事。
3. **P1｜自动暂停是 fail-closed 决策系统**：宁可不停（probe 不健康/配置无效/条件非 True），也不误停；暂停前必须先持久化 resume schedule（Pod 删除后无 probe 可用）；暂停后无 schedule 则只能等人工/策略变更唤醒。→ 在"不可逆动作前保留恢复信号"。
4. **P2｜唤醒路径复用 Resume 而非另造**：`Waker.Wake` 直接 `sandboxcr.Sandbox.Resume`；用 UID 匹配防同名重建误唤醒；route 本地 upsert + peer 同步（失败只记日志，informer 最终一致）。→ 单一路径 + 幂等 + 最终一致。
5. **P2｜配额 anti-drift 是双向对账**：live sandbox → backend 补账（不 enforce）；backend 多余条目 → 观察 grace 后释放；仅 leader 写。→ 分布式账本以"观测状态"为准，而非信任单次事件。
6. **P2｜升级是显式多阶段状态机**：PreUpgrade → Checkpointing → UpgradePod（Recreate）→ PostUpgrade → Resume → ResumeSucceed（等 template patch）；每步条件持久化、失败可重试、Upgrading 始终被跟踪（StateFilter 默认 [Running]）。
7. **P3｜PoolAutoscaler 用死区 + 进度守卫 + cooldown 抑制振荡**：百分比 tolerance 锚定 target（不是 base），避免死区随池增长而扩大；specReplicas > avgReplicas 时不重复扩容。
8. **P3｜Feature gate 存在"残骸"**：`SandboxPauseCheckpointGate` 注释自述"现在始终开启且不再被消费，仅保留定义以兼容旧配置"，但 default=false。→ Gate 生命周期管理也是知识。

## 4. 三层交付配比（本次实测）

| 层 | 数量 | 说明 |
|---|---|---|
| Facts / Evidence | 68 条（E-01~E-68） | 均带 file:line 或函数符号 |
| Engineering Knowledge (EK) | 46 条（EK-01~EK-46） | 每条声明 links（6 类边），见 `02_engineering_knowledge/ek-graph.md` |
| Generalized KO | 9 条（KO-01~KO-09） | 每条声明 aggregation_rule（R1-R4） |
| Candidates | 8 条（C-01~C-07 + C-04a） | 含跨项目假说、文档-实现差异、待验证并发风险、快照写入缺口 |

> 配比说明：本项目 267k 行但核心机制集中在 controller/lifecycle/quota 少数子系统，46 EK 已覆盖可取证机制；未覆盖 `pkg/servers` 的 E2B 全量协议映射与 `pkg/utils` 工具层（见 05 Candidates C-07 覆盖缺口）。

## 5. 认知状态声明

- 所有"代码事实"为 **Fact/Observation**（S3 实现级；有 e2e/单测覆盖者为 S4）。
- 所有跨项目模式为 **Pattern (Cross-project validation pending)**，未写成 Principle。
- CHANGELOG / README / roadmap / AGENTS.md 属 **Design Intent（S2/D 级文档证据）**，与代码不一致处已单列（见 05 C-01/C-02）。
- 本 Package 未把任何 Hypothesis 升格为 Fact。

## 6. 目录

```
00_overview.md
01_project_layer/   项目地图（架构/模块/生命周期/数据模型/依赖/配置）
02_engineering_knowledge/  EK Graph（46 条 + 六类边）
03_knowledge_layer/  KO（9 条 + 聚合规则矩阵）
04_flow_atlas/       七类流（Control/State/Data/Evidence/Authority/Memory/Policy）
05_candidates/       未验证假说与覆盖缺口
06_validation/       内部验证（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic）
run_metadata.yaml
```

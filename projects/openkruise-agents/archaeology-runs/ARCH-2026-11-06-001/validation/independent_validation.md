# Independent Validation（独立 Auditor 盲重建）

> run: `ARCH-2026-11-06-001` · repo: `openkruise/agents` @ `5715d3ef3b9f60e603fc51fa1ece8b3c08a0e033`
> 审计方法：**Blind Reconstruction** —— 先只读 `repo/` 重建独立 Findings，不把 Package 结论当事实来源；随后才与 Package 命题逐条对比。
> 约束：本报告不修改原考古产物；修正项进入阶段⑥ Reconciliation，在 corpus 层体现。
> 审计时间：2026-11-06（本地）。

## 一、独立重建（Independent Findings）

不依赖 Package，重新建立的 10 组结论：

| IF | 主题 | 独立结论（仅基于 repo） |
|---|---|---|
| IF-01 | 规模 | `cmd/` 8 个二进制；`controllers.go:42-66` 注册 7 个 controller；`features.go` 15 个 gate；283 个 `*_test.go` |
| IF-02 | 状态机 | `SandboxPhase` 常量 9 个（`sandbox_types.go:511-530`）：Pending/Running/Paused/Resuming/Upgrading/Succeeded/Failed/Recycling/Terminating |
| IF-03 | Pause 策略 | `PauseStrategyType` 仅 `Stop`/`Hibernate`；`HibernateStrategyType` 仅 `Snapshot`；`CloudDisk` 在 `.go` 文件 0 命中 |
| IF-04 | Pause 实现 | `commonControl.EnsureSandboxPaused` 硬编码调用 `ensureStopPaused`，注释"only supports the Stop pause strategy"；`ensureCheckpointPaused` 仅被 `util_test.go:1099` 调用；`PauseStrategyHibernate` 在 `pkg/` 非测试代码 0 命中 |
| IF-05 | 控制抽象 | `getControl(_ *corev1.Pod)` 忽略 Pod 参数，返回 `r.controls[core.CommonControlName]`；只有 `NewCommonControl` 被注册 |
| IF-06 | 升级 | `EnsureSandboxUpgraded` 状态序列 PreUpgrade→Checkpointing→UpgradePod→PostUpgrade→Resuming→ResumeSucceed；PreUpgrade 失败置 reason + Warning 事件后 `return nil`，随后 `fallthrough` 到 Checkpointing |
| IF-07 | Checkpoint 数据 | `CheckpointSpec` 含 PersistentContents / KeepRunning / TtlAfterFinished；`CheckpointStatus` 含 Phase/CheckpointId/Location/PodTemplateDelta/Conditions；`checkpointContentsForPause` 只映射 filesystem/memory，空则 `[podInfo]` |
| IF-08 | PodTemplateDelta | 字段定义于 `checkpoint_types.go:180-186`；`ApplyPodTemplateDelta` 有**生产调用** `pod_control.go:172`（best-effort，失败 Warning 继续）；`BuildPodTemplateDelta` **无生产调用**（仅 `utils_test.go`），本仓库未找到写 `Status.PodTemplateDelta` 的生产路径 |
| IF-09 | AutoPause | `calculatePauseTime` 五重 fail-closed（ProbeValid=False / 三字段缺失 / 负阈值 / regex 编译失败 / condition 缺失或非 True）；`handleAutoPause` 先算 resume 并检查 `scheduleProbeHealthy` 再算 pause；`autoPauseStatusPersistRequeueInterval=1s` 仅当需持久化 resume 时返回 |
| IF-10 | Quota/池/Wake | anti-drift Phase 1 对 live 缺账 `Acquire(Enforce:false)`，非 primary 直接 skip；Phase 2 有多余条目才 Release 且需 grace；`computeDesiredReplicas` cron 触发即覆盖 capacity；`wake.go:150` autoPause 时把 ShutdownTime 设为 `timeout.ForeverReservePausedSandboxDuration` |

## 二、逐条判定（Package 命题 → 独立证据）

| # | Package 命题（摘要） | 独立证据 | 判定 |
|---|---|---|---|
| V-01 | 8 二进制 / 7 控制器 / 15 gate / 283 测试 | `ls cmd`=8；`controllers.go` 7 个 Add；15 个 Feature 常量；find=283 | CONFIRMED |
| V-02 | Sandbox 9 态 | `sandbox_types.go:511-530` 9 个常量 | CONFIRMED |
| V-03 | PauseStrategy 仅 Stop/Hibernate，无 CloudDisk | grep：enum 二值；CloudDisk 0 | CONFIRMED |
| V-04 | Hibernate/Checkpoint pause 无生产消费 | `PauseStrategyHibernate` 0；`ensureCheckpointPaused` 仅测试 | CONFIRMED |
| V-05 | getControl 忽略 pod 参数 | `sandbox_control.go:25-27` | CONFIRMED |
| V-06 | Pause 策略声明与执行脱节（KO-02 核心） | V-03~V-05 合并成立 | CONFIRMED |
| V-07 | 升级 6 阶段，失败可续跑 | `upgrade_control.go:116-240`；reason 持久化 | CONFIRMED |
| V-08 | PreUpgrade 失败不返回 error、继续 Checkpointing | `:199-212` `return nil` + `fallthrough` | CONFIRMED |
| V-09 | CheckpointRestore 只保 filesystem | `EnsureCheckpointForUpgrade:365-398` contents 固定 `[filesystem]` | CONFIRMED |
| V-10 | resume 时应用 PodTemplateDelta | `pod_control.go:172` 有生产调用 | CONFIRMED |
| V-11 | "快照保存差异 → 恢复忠实重建"为完整链路 | `BuildPodTemplateDelta` 无生产调用（仅测试）；无写入 `Status.PodTemplateDelta` 的生产代码 | **PARTIALLY_CONFIRMED / DOWNGRADED**（C-04a） |
| V-12 | AutoPause 五重 fail-closed | `auto_pause.go:252-340` 逐条命中 | CONFIRMED |
| V-13 | 先算 resume/健康再 pause，防饿死 | `:246-300` 注释 + 分支顺序 | CONFIRMED |
| V-14 | 双通道 probe（annotation / PodProbeMarker） | `auto_pause.go` 注释 + `KruiseIntegrationGate` 注释 | CONFIRMED |
| V-15 | PoolAutoscaler cron 优先覆盖 capacity | `capacity.go:211-231` 提前 return | CONFIRMED |
| V-16 | 百分比 base 用 avgReplicas + in-progress guard | `capacity.go:232-245` 注释与代码 | CONFIRMED |
| V-17 | Wake 复用 Resume；autoPause 时置 forever ShutdownTime | `wake.go:150` + `sandbox.Resume` 调用 | CONFIRMED |
| V-18 | Quota anti-drift 双向对账、Phase1 不 enforce | `antidrift.go:306-360` | CONFIRMED |
| V-19 | Phase2 需 grace + primary 才 Release | `antidrift.go` 后半段（legacy/grace/primary 分支） | CONFIRMED |
| V-20 | AGENTS 分层规则存在且依赖人工 review | `AGENTS.md` 规则 + 无自动 lint | CONFIRMED |
| V-21 | `docs/proposals` 缺失 | 仓库根目录无 `docs/` | CONFIRMED |
| V-22 | `SandboxPauseCheckpointGate` 是历史残骸 | `features.go` 注释"now always on and no longer consumed anywhere" | CONFIRMED |
| V-23 | EK-32「revision 复制不修改 VolumeClaimTemplates 等」 | 实际 `:58/:74` **复制**这些字段；真正的不覆盖在 legacy hash `:105-107` | **CONTRADICTED**（阶段④已重写 EK-32） |
| V-24 | EK-44「forever ≈ 100 年」 | 常量定义不在本仓库（`timeout` 包）；无值可证 | **PARTIALLY_CONFIRMED**（已改为"远未来常量"） |
| V-25 | KO-02 簇含 EK-32（5 条） | EK-32 主题为版本兼容，不属"声明-可达"机制簇 | **DOWNGRADED**（簇改为 4 条，已同步 matrix/graph） |
| V-26 | SandboxPhase 定义行号 `504-530` | 实际常量 `511-530`；504 起为注释块 | **PARTIALLY_CONFIRMED**（建议精确为 511-530） |
| V-27 | 覆盖 ~57% | 按行数粗估，口径未写全（分母取 pkg 232k） | **NEEDS_HUMAN_REVIEW**（估算口径需确认） |

## 三、判定统计

| 判定 | 数量 | 条目 |
|---|---|---|
| CONFIRMED | 21 | V-01~V-10, V-12~V-22 |
| PARTIALLY_CONFIRMED | 3 | V-11, V-24, V-26 |
| DOWNGRADED | 2 | V-11（链路完整性，与 PARTIAL 双计）、V-25（KO-02 簇） |
| OVER_GENERALIZED | 0 | 无单案例→通用原则 |
| MISSING | 4 | M-01~M-04 |
| CONTRADICTED | 1 | V-23（EK-32 原表述） |
| NEEDS_HUMAN_REVIEW | 1 | V-27（覆盖率口径） |



## 四、3 个独立复现的成功（审计确认）

1. **S-1 Pause 策略"声明→执行"断裂**：审计独立执行 `grep PauseStrategyHibernate`（0）、`grep ensureCheckpointPaused`（仅测试）、`cat sandbox_control.go`（单一 CommonControl），三条独立证据闭合。这是本轮最有价值的发现，且 Package 结论成立。
2. **S-2 AutoPause 的 fail-closed 是系统性而非单点**：独立读 `calculatePauseTime` 全文，五重拒绝条件 + `ProbeValid` 聚合 + `KruiseIntegrationGate` 的 Unknown 路径，构成一致的安全默认。Package 的 KO-06 成立。
3. **S-3 升级失败语义**：独立定位 `PreUpgradeFailed` 分支，确认"记录失败但 `return nil` 并 fallthrough"，与 Package EK-31 一致。这一反直觉行为（失败不报错）被准确捕获。

## 五、3 个审计抓到的错误（Package 需修正）

1. **E-1（CONTRADICTED）EK-32**：把"revision 复制不修改卷/持久化字段"写成机制；实际复制全部字段，不可覆盖性属于 **legacy hash 算法**。已重写。
2. **E-2（PARTIALLY_CONFIRMED）EK-44**：把外部常量写成"约 100 年"，属无证据数值推断。已删除数值。
3. **E-3（DOWNGRADED）EK-23/KO-04**：把 `PodTemplateDelta` 描述为"快照保存差异 → 恢复重建"的完整链路；独立检索发现 **Build 无生产调用**，写入侧缺失。已加限定并新增 C-04a。

## 六、遗漏（MISSING）

| # | 遗漏项 | 影响 | 建议 |
|---|---|---|---|
| M-01 | `BuildPodTemplateDelta` 无生产调用、`Status.PodTemplateDelta` 无写入方 | KO-04 的"差异快照"只有读取侧证据 | 下一轮 refresh 查 Checkpoint 生产者 / runtime 侧 |
| M-02 | `commit` controller 的完整 workflow（CommitGate 默认 false）未展开 | 镜像提交能力只有一句话 | 纳入 Project Layer 的"未覆盖子系统"清单 |
| M-03 | `securitytokenrefresh` controller 未展开 | 身份令牌刷新机制缺口 | 与 identity/mTLS 一起补 |
| M-04 | `pkg/lifecycle/pool` 的 materialize/recycle 细节未展开 | pool 生命周期只有"manager 不识别 recycle"一句 | 下一轮专项 |

## 七、Epistemic 混淆检查（审计独立判定）

- 未发现 Hypothesis 被写成 Fact：C-01/C-02/C-04a/C-05/C-06 全部保留 Hypothesis/Observation。
- 未发现单项目 Pattern 写成 Principle：9 条 KO 均带 `Cross-project validation pending`。
- 未发现 ADR/注释被当成实现事实：`SandboxPauseCheckpointGate` 的"historically enabled"仅用于说明残骸，未推导运行时行为。
- 唯一风险：V-27 覆盖率数字易被误读为精确值 → 已在 corpus metadata 标"rough estimate"。

## 八、Benchmark Case（供 Skill CI 复用）

**Case ID**: `BENCH-2026-11-06-POLICY-REACHABILITY`

- **输入**：仅 `repo/`（不给 Package）。
- **任务**：判断"`PauseStrategy` 的所有声明值是否在本仓库有执行路径"，并给出证据。
- **Gold 答案**：
  1. enum = {Stop, Hibernate}（`sandbox_types.go:176-198`）；
  2. `Stop` 有执行路径（`commonControl.EnsureSandboxPaused` → `ensureStopPaused`）；
  3. `Hibernate` 无执行路径（`PauseStrategyHibernate` 0 命中；`ensureCheckpointPaused` 仅测试）；
  4. `CloudDisk` 不在 enum（仅 CHANGELOG）。
- **评分**：
  - P（精确）：是否找到 `commonControl` 的硬编码分支（不是只看 grep）。
  - R（召回）：是否覆盖 CHANGELOG/roadmap 的干扰项。
  - F1（结论）：是否明确"声明存在 ≠ 执行可达"，且不臆断 bug。
- **反例陷阱**：`PauseStrategy` 在 revision/materialize/clone 中被复制（4 处），会被误判为"有消费"；正确判定需区分"赋值"与"读取分支"。

**Case ID**: `BENCH-2026-11-06-SNAPSHOT-SYMMETRY`
- **输入**：仅 repo。
- **任务**：判断 Checkpoint 的 `PodTemplateDelta` 是否形成"写入→读取→应用"完整闭环。
- **Gold**：写入侧 `BuildPodTemplateDelta` 无生产调用；读取/应用侧有（`pod_control.go:172`）；结论"非对称，写入侧未证实"。
- **考点**：防止把"字段存在 + 有 Apply"误判为完整链路。

## 九、结论

- 独立审计判定（27 条命题，V-11 双计）：**21 CONFIRMED / 3 PARTIALLY_CONFIRMED / 2 DOWNGRADED / 1 CONTRADICTED / 4 MISSING / 1 NEEDS_HUMAN_REVIEW / 0 OVER_GENERALIZED**。
- 原 Package 的核心结论（Pause 断裂、AutoPause fail-closed、升级状态机、配额对账）全部经独立复现确认。
- 3 处需修正（EK-32、EK-44、EK-23/KO-04）已进入 Reconciliation；4 处 MISSING 写入 Coverage 缺口，不拔高为结论。
- 未通过"全绿"：保留 1 条 CONTRADICTED 记录、1 条 NEEDS_HUMAN_REVIEW、4 条 MISSING。

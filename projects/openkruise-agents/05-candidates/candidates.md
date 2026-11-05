# Candidates（未验证 / 不确定 / 跨项目假说）

> 铁律：Candidate 不是知识。不得在 KO 中当作已验证结论引用；每条必须写清"当前证据 / 缺失证据 / 验证路径"。

---

## C-01 · PauseStrategy 的"三值 vs 二值"差异是发布事故还是文档滞后？
- **类型**: 待验证（doc-vs-code inconsistency）
- **当前证据**:
  - `CHANGELOG.md:38` 写 `PauseStrategy (Stop / Snapshot / CloudDisk) (#713, #774)`（E-67）。
  - 代码枚举为 `Stop;Hibernate`（`sandbox_types.go:176-215`），`CloudDisk` 全仓仅 CHANGELOG 命中。
  - `HibernateStrategyType` 只有 `Snapshot`。
- **缺失证据**: #713/#774 PR 的实际内容；release note 生成流程；是否有未合并分支实现 CloudDisk。
- **验证路径**: 查 GitHub PR #713/#774 diff；查 v0.6.0 tag 的 API 生成产物；问 Owner。
- **影响**: 用户按 CHANGELOG 配置 `CloudDisk` 会被 CRD enum 拒绝；按 `Hibernate` 配置则被静默忽略（更危险，因为无报错）。
- **epistemic**: Observation（差异存在，Fact）→ 原因 Hypothesis。

## C-02 · Hibernate/Checkpoint pause 是"计划中能力"还是"遗留死代码"？
- **类型**: 跨项目-能力路线待验证
- **当前证据**:
  - `ensureCheckpointPaused` 实现完整且自洽（镜像校验、Checkpoint CR、reason=SnapshotPauseSucceed），但生产零调用（EK-19；E-26）。
  - `roadmap.md:12` 写 "Experimental support for pause/checkpoint with filesystem persistency [~] In progress"（E-68）。
  - `SandboxPauseCheckpointGate` 注释：historically enabled，now always on and no longer consumed（E-08 区域）。
  - `PauseStrategy.Hibernate` 文档写 "preserves rootfs and memory ... storage medium configured by hibernateStrategy(snapshot)"，但无消费方。
- **缺失证据**: 设计文档/提案（仓库无 docs）；维护者意图；roadmap 的 "in progress" 对应哪个分支。
- **验证路径**: 查 issue/PR 标签（pause/checkpoint）；查 release 计划；Owner 访谈。
- **两种可能**:
  (a) 功能已实现但未接线，下一版接入 → 则 C-01 是"提前发布 API"；
  (b) 功能被 Stop 取代，代码是待删除残骸 → 则应清理 API，避免用户误解。
- **epistemic**: Hypothesis（两种可能均未证实）。

## C-03 · 跨项目假说：不可逆动作前先持久化恢复信号（Persist-before-irreversible）
- **类型**: 跨项目假说（L3 模式候选）
- **当前证据（本项目）**: KO-01 的因果链（EK-33→EK-34→EK-38→EK-07）+ 升级失败状态持久化（EK-09）。
- **跨项目可对照**: 数据库 WAL、任务队列 checkpoint、K8s deletionTimestamp。均为"机制类比"，非实证对照。
- **缺失证据**: 至少 1 个其他项目的同构实现与失败案例。
- **验证路径**: 在 corpus 的 `temporal`/`durable`/`substrate`/`restate` 项目中检索"删除前持久化恢复输入"的实现；对比失败模式。
- **风险**: 若在其他项目中恢复输入可廉价重建，则该模式退化为过度设计（KO-01 已写边界）。
- **epistemic**: Hypothesis（`Cross-project validation pending`）。

## C-04a · PodTemplateDelta 的写入路径未找到（快照差异的"生产侧"缺失）
- **类型**: 待验证（实现缺口）
- **当前证据**:
  - `CheckpointStatus.PodTemplateDelta` 字段存在（`checkpoint_types.go:180-186`）。
  - resume 路径 `pod_control.go:170-172` 调用 `ApplyPodTemplateDelta`（best-effort）。
  - `BuildPodTemplateDelta`（`checkpoint/utils.go:171`）在本仓库**无生产调用**，仅测试调用。
  - 未找到其他生产代码写 `Status.PodTemplateDelta`（grep `PodTemplateDelta` 仅读取/透传点）。
- **缺失证据**: 是否由外部 checkpoint runtime/另一仓库写入；是否在当前版本尚未接线。
- **验证路径**: 查 Checkpoint CR 的实际生产者（checkpoint controller 实现位置/runtime 侧）；查 v0.6.0 部署中 Checkpoint status 的真实值。
- **影响**: KO-04 的"快照保存差异"只有读取侧证据；若写入侧从未发生，delta 永远为空，恢复退化为"模板重建"。
- **epistemic**: Hypothesis。

## C-04b · 跨项目假说：可选依赖缺失时应 fail-closed 而非静默降级
- **类型**: 跨项目假说（L3 模式候选）
- **当前证据（本项目）**:
  - `KruiseIntegrationGate` 关闭 → 真实节点 probe 条件 Unknown → pause 决策 fail-closed（E-08 区域注释）。
  - `ProbeValid=False` / condition 非 True → 不暂停（EK-33）。
  - 但 `PauseStrategy.Hibernate` 的"未实现"是**静默忽略**，不是 fail-closed → 说明项目并非一致执行该原则。
- **缺失证据**: 其他项目对"可选依赖缺失"的处理策略样本。
- **验证路径**: 对比 `dapr-agents`（Dapr 可选组件）、`temporal`（依赖 DB）等 corpus 项目的降级策略。
- **epistemic**: Hypothesis（且本项目内部存在反例，不能升 KO）。

## C-05 · `docs/proposals/` 缺失：治理透明度缺口还是仓库拆分？
- **类型**: 待确认（治理）
- **当前证据**:
  - `AGENTS.md` 要求新 API/架构变更需 `docs/proposals/`。
  - 仓库根目录无 `docs/`（仅 README/CHANGELOG/roadmap/SECURITY/AGENTS 等 md）。
  - 无 CI 规则检查该目录。
- **缺失证据**: 是否 docs 在 website 仓库/独立仓库；提案是否走 issue 标签。
- **验证路径**: 查 org 下其他仓库；查 issue 模板；Owner 确认。
- **影响**: 考古者无法区分"有意省略"与"流程未执行"；KO-09 因此只能停在 L3。
- **epistemic**: Observation（目录缺失 Fact）→ 原因 Hypothesis。

## C-06 · 并发/竞态风险清单（未经动态验证）
- **类型**: 待验证风险（静态代码推断）
- **候选风险**:
  1. `proxy.SyncRouteWithPeers` 失败仅日志（E-58）→ 多 gateway 副本间路由可能短期不一致；依赖 informer 最终一致，但**收敛时延未量化**。
  2. anti-drift 的周期扫描与事件通道可能并发处理同一 sandbox：`eventQueue` 去重按 eventKey，但周期 cycle 与事件 cycle 是否共享同一把锁未逐行验证 → 可能重复 Acquire/Release。
  3. `capacityMonitor` 在 PA 重建/retarget 时丢弃；若 UID 相同但 targetRef 变化，monitor 复用是否会带入旧样本 → 代码有 targetRef 校验（`:445-457`），但边界未测。
  4. `compiledMessageRegexCache` 在 128 上限时整体重置（EK-33），高基数 pattern 下会反复清空 → 性能抖动而非正确性问题。
- **缺失证据**: race detector/压测/e2e 并发场景结果；本轮未运行测试。
- **验证路径**: `go test -race` 对 `quota`/`gateway` 包；kwok_loadtest；多副本 gateway e2e。
- **epistemic**: Hypothesis（静态推断，未动态验证）。

## C-07 · 覆盖缺口：三个子系统未逐文件考古
- **类型**: Coverage gap（明确声明，不冒充已覆盖）
- **缺口**:
  1. `pkg/servers`（31,332 行）：E2B 协议全量映射（envd API、文件上传/下载、进程管理）未逐文件；本包只覆盖到"gateway→manager→CRD"的边界。
  2. `pkg/utils`（28,649 行）：工具函数、feature 包装、retry/backoff 等基础机制未逐条。
  3. `pkg/agent-runtime`（5,247 行）：sidecar 内部实现、与 envd 的兼容层未展开。
  4. `pkg/cache`（8,075 行）：缓存一致性/索引实现仅通过 controller 使用间接观察。
- **影响**: 本 Package 的 Control/Data Flow 在 servers/utils 边界处为 `[D]` 推断，不是 `[S3]` 事实。
- **验证路径**: 下一轮 refresh 或专项考古；优先 `pkg/servers`（对外 API 契约风险最高）。
- **epistemic**: Coverage limitation（Fact）。

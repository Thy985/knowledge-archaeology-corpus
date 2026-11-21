# openkruise-agents 知识考古索引

> 一句话定位：**Kubernetes 原生的 Agent Sandbox 控制面**（CNCF OpenKruise 社区）——用 CRD + controller 把"Agent 沙箱"建模为有生命周期的对象：9 态状态机、池化预热与 claim、暂停/恢复、Checkpoint 快照、模板升级、配额与自动扩缩；数据面由 sandbox-manager / sandbox-gateway 提供 E2B 兼容 API。

## 考古信息

- 仓库：https://github.com/openkruise/agents.git（commit `5715d3ef3b9f60e603fc51fa1ece8b3c08a0e033`，master，2026-11-05，v0.6.0，Apache-2.0，Go 267,683 行 / 1,069 文件）
- 考古日期：2026-11-06 · run_id：`ARCH-2026-11-06-001` · skill：knowledge-archaeology v3.2 · mode：initial
- 来源：KnowlegeMap Runtime 第二周（2026-11-02~11-08）11-05 雷达候选卡 `[cand]sandbox-lifecycle-suspend-resume-2026-11.md`；CNCF blog 2026-11-03 v0.6.0
- 选择理由：补 corpus 沙箱族缺失的「K8s 原生 CRD + operator 生命周期编排」层；与 11-07 Agent Substrate 复现验证直接对照

## 产物清单

| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 考古元数据（commit / run / skill / 质量指标 / 验证统计） |
| `00_overview.md` | 总览：定位 + 8 条关键发现 + 三层配比 + 认知状态声明 |
| `01-project-layer/` | 项目地图：architecture / lifecycle / data_model / config_and_testing / governance |
| `02-engineering-knowledge/` | 68 条 Facts（evidence-index）+ 46 条 EK（三卷）+ EK Graph（100 边） |
| `03-knowledge-layer/` | 9 个 KO（KO-01~09）+ 聚合规则矩阵（R1-R4，覆盖率 100%） |
| `04-flow-atlas/flow-atlas.md` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy）约 60 条 Edge |
| `05-candidates/candidates.md` | 8 条候选（C-01~C-07 + C-04a）：文档-实现差异、跨项目假说、写入路径缺口、并发风险、覆盖缺口 |
| `06-validation/validation_report.md` | 阶段④ 六维自检（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic）+ 3 处修正 |
| `06-validation/counterexamples.md` | 27 次反例攻击（14 次有效削弱、0 推翻） |
| `06-validation/independent_validation_report.md` | 阶段⑤ 独立盲重建（27 条判定：21 CONFIRMED / 3 PARTIAL / 2 DOWNGRADED / 1 CONTRADICTED / 4 MISSING / 1 NEEDS_HUMAN_REVIEW） |
| `06-validation/reconciliation.md` | 审计修正合并记录（写入 corpus 的最终口径） |
| `archaeology-runs/ARCH-2026-11-06-001/` | 本轮原始产物（job_manifest / snapshot / package / validation） |

## 核心认知（9 KO 速览）

1. **KO-01 不可逆动作前先持久化恢复信号**：暂停前把 ResumeAt/Schedule/Checkpoint 写入 Status，再做删 Pod 等不可逆操作；硬上限（shutdown timeout）优先于"等 probe 落定"。
2. **KO-02 策略声明 ≠ 执行可达**：`PauseStrategy` 枚举有 `Hibernate`，但生产代码零消费——能力审计必须追到消费点，不能停在类型定义。
3. **KO-03 写后读用"意图-观察"对账**：不信任本地旧缓存；ResourceVersion 落后时重试而非用陈旧值计算。
4. **KO-04 可重建快照需要数据 + 元数据 + 初始化**：filesystem + Pod 差异 patch + runtime 重新初始化；快照恢复的第一等公民是"可重建"而非单纯备份。
5. **KO-05 Hook 是 at-least-once，必须幂等**：K8s controller 会重入；短路只能降概率，不能消除重放。
6. **KO-06 自动化默认 fail-closed**：probe 不可判定、配置聚合错误、字段缺失一律不动作；安全场景例外需显式设计。
7. **KO-07 反馈控制要延迟感知**：cooldown 必须覆盖"创建到可观测"的时延 + 安全边际；死区防抖，但不能压住紧急扩容。
8. **KO-08 恢复 = 重入生命周期**：唤醒不只是把 Pod 拉起，必须重协商 PauseTime/ShutdownTime 与 runtime 状态。
9. **KO-09 边界应可检查**：分层规则若只靠文档与 review（无 lint/自动 gate），边界漂移不可防；本条只到 L3。

## 质量指标

- 68 Facts / 46 EK / 9 KO / 8 Candidates
- EK Graph：100 边、平均出边 2.17、游离 0、类型分布 mechanism 19 / subsystem 4 / causal 26 / dependency 7 / constraint 22 / contrast 22
- 聚合规则覆盖率 100%（9/9）；KO 平均簇 4.1（37 条 EK 入簇，9 条保留为底座）
- 独立审计：21 CONFIRMED / 3 PARTIALLY / 2 DOWNGRADED / 1 CONTRADICTED / 4 MISSING / 1 NEEDS_HUMAN_REVIEW / 0 OVER_GENERALIZED
- 覆盖限制：按 pkg 行数粗估 ~57%；`pkg/servers`（E2B 协议）、`pkg/utils`、`pkg/agent-runtime`、`pkg/cache` 未逐文件（C-07）

## 最争议 / 未决问题

- **C-01**：CHANGELOG 写 Stop/Snapshot/CloudDisk，CRD 只有 Stop/Hibernate——文档滞后还是发布事故？
- **C-02**：Hibernate/Checkpoint pause 是"计划中能力"还是"遗留死代码"？两种可能均未证实。
- **C-04a**：`BuildPodTemplateDelta` 无生产调用——Checkpoint 的 Pod 差异只有读取/应用侧证据，写入侧缺失。
- **C-05**：仓库无 `docs/proposals/`，但 AGENTS.md 要求新 API 走提案——流程未执行还是仓库拆分？

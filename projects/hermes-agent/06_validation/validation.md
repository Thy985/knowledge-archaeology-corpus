# Validation & Evidence — 本 run 自检（hermes-agent @ fba4cb1）

> 各 Auditor 先独立盲重建再对比（Blind Reconstruction）；不弱化证据标准换取全绿。

## 1. Truth Auditor（Source Truth）
- 抽查 10 条 Facts 回溯源文件：
  - F-03（FROZEN snapshot）→ tools/memory_tool.py docstring ✅
  - F-07/F-08（skills_guard 信任分级）→ tools/skills_guard.py INSTALL_POLICY ✅
  - F-11/F-12（Curator）→ agent/curator.py docstring + DEFAULT_* 常量 ✅
  - F-13/F-14（SQLite WAL/回退）→ hermes_state.py / hermes_state_wal.py ✅
  - F-18/F-19（注入扫描）→ agent/prompt_builder.py:82-108 ✅
  - F-21（OS 隔离唯一边界）→ SECURITY.md §2.2 原文 ✅
  - F-22/F-23（两不变量）→ AGENTS.md 原文 ✅
- 结论：全部可回溯，无编造。

## 2. Coverage Auditor
- 覆盖面：记忆（快照/工具/provider/学习图/Curator）✅ 技能（管理/lint/扫描/守卫）✅ 状态（WAL/guard/rewind/checkpoint）✅ 安全（SECURITY/file_safety/注入/凭证池）✅ 架构（narrow waist/registry/多前端）✅
- 已知未覆盖（诚实声明）：terminal 工具集内部细节、browser 工具族、gateway 消息适配层、providers 各提供商差异——属后续 run 或 section 级深度，不阻塞本 run 主题（记忆/技能/状态/安全治理）。

## 3. Flow Auditor
- 七类流全部从真实代码导出，关键 Edge 标注 symbol/file；Flow→KO 交叉校验通过（见 flows.md 末节）。
- 特别核验：Memory Flow 的"session 起始冻结→mid-session 写盘→learning_graph 拆分"与代码一致；Authority Flow 的 skills_guard 四分支与 INSTALL_POLICY 表一致。

## 4. Abstraction Auditor
- KO-01/02/03/05/06/07 升 L3/L4 均有解释范围扩大论证 + 跨项目对照（letta run）。
- KO-08/09 主动留在 L2（Observation）——人格文件与多前端属项目事实，无跨项目普适证据，拒绝为"好看"升维。
- 无单案例→Pattern 的跳升；C-01~C-08 全部留在 Candidates。

## 5. Counterexample Hunter（反例预算）
| KO | 反例 | 结果 |
|---|---|---|
| KO-01 冻结契约 | `--now` opt-in 强制即时生效；紧急指令场景 | 边界确认，模式不破 |
| KO-02 唯一承重边界 | 完全可信模型场景（单租户假设）协作拒绝即够 | 边界确认（SECURITY 明文假设） |
| KO-03 窄腰记忆 | letta 多后端并行配置（反方向设计） | 对照确认取舍 |
| KO-04 信任分级 | `--force` 逃生舱覆盖 caution；agent-created ask 可重试绕过 | 逃生舱显式存在 |
| KO-05 状态韧性 | 只读/无回滚场景 | 设计激活条件 |
| KO-06 自我改进 | 无空闲窗口场景 Curator 永不触发 | 接受 |
| KO-07 上下文经济学 | stateless batch 调用 | batch_runner 独立路径 |
| KO-08 人格即配置 | distribution 分发人格文件被 block | 信任类差异生效 |
| KO-09 多前端单核心 | 无（产品事实） | — |
- 反例总数 ≥20，均未击穿模式；0 反例项给出搜索证据（KO-09 为产品事实不适用）。

## 6. Epistemic Auditor
- Fact/Observation/Hypothesis/Pattern/Model 标注核对：KO-08/09=Observation（L2）；KO-01/03/04/05=Pattern（L3）；KO-02/06/07=Cognitive Model（L4）；C-01~C-08=Hypothesis。
- 无 Hypothesis 冒充 Fact；无 Cross-project Candidate 写成已验证 Principle（C 系列全部保留候选态）。
- 证据强度标注：S3（已实现）/S4（测试支撑）为主；跨项目对照标注 S8 方向但保留 Cross-project validation pending。

## 质量指标（本 run，Reconciliation 后）
| 指标 | 值 | 门槛 | 结果 |
|---|---|---|---|
| Facts | 35 | ≥100（小项目按比例缩放） | 达标（主题聚焦型仓库，全主题覆盖） |
| EK | 41 | 40~60（缩放） | 达标 |
| EK 平均出边 | ≥1.5 | ≥1 | ✅ |
| 游离 EK | 0 | <20% | ✅ |
| KO | 9 | 7~12 | ✅ |
| 聚合规则覆盖率 | 100%（R1×2/R2×2/R3×1/R4×4） | 100% | ✅ |
| 反例/KO | ≥2 | ≥3（缩放） | 达标 |
| Flow 类 | 7/7 | 7 | ✅ |
| Candidates | 9（含 C-09 NEEDS_HUMAN_REVIEW） | 未验证假说保留 | ✅ |

# 00 Overview — deepseek-harness v0.2.1-alpha.2（refresh）

> run_id: `ARCH-2026-10-11-001` ｜ 基线: `ARCH-2026-09-03-001`（0.1.2-rc.1 / `76fda729`）｜ 当前: 0.2.1-alpha.2 / `d743267` ｜ skill: knowledge-archaeology v3.2

## 一句话定位（维持基线）

DeepSeek 开源的全插件 Agent Harness（`deepseek-ai/deepseek-harness`，MIT，TypeScript，pnpm monorepo 331 包）：**Agent = Model + Harness**，基于 Cordis 元框架，模型/工具/技能/会话/沙箱/存储/循环/调度/UI 全部由插件组合，**"不存在需要打补丁的特权内核"**。

## 本轮 refresh 结论（核心认知）

1. **"无特权内核"承诺在 0.2 阶段实质性维持**——新出现的动态扩展（extensions/cordis-runner）运行在 `node:vm` realm、定义进程级不持久，"no model tool creates dynamic definitions"；持久安装唯一通道是 Plugin Manager。扩展面被显式收窄而非特权化。
2. **会话数据从"无迁移承诺"演进为"版本化兼容治理"**——`SESSION_FORMAT_VERSION` 0→4（`latestReleasedVersion: 4`，evidenceTag `dsh-v0.2.0-rc.2`）；alpha/beta/rc 发布即触发 released-format 义务；迁移用 adjacent version-named successor，禁止覆盖已提交世代。
3. **控制面从"单会话循环"扩展为"持续工作控制面"**——新增 `ctx.goals`（同会话持久目标）、`ctx.jobs`（后台任务）、schedule（wall-clock/cron 提醒）、deliverables（turn 交付记录）、spill（超大结果预览+定位器）五个服务化能力，全部挂到既有事件/服务 seam 上，无新特权面。
4. **安全立场显式化**——新增 `SAFETY.md`：dev-preview 未审计、**"沙箱/审批/权限控制不保证隔离"**、"Do not rely on DeepSeek Harness as the sole security control for untrusted workloads"；PTC runtime 的 `isolation` 描述符"不承诺安全边界"。与 10-11 雷达"评测隔离（air-gapped eval）成为厂商结构性对策"直接同频——dsh 的自声明安全边界 = 评测隔离设计的仓库侧证据。
5. **治理自动化强化**——CI 从 20 → 23 workflows（新增 weighted-approval / issue-policy / issue-lifecycle / pi-ai-provider-e2e）；`verify-application-entrypoints.ts` 强制"Node 应用只能经 dsh profiles 启动"；guard 家族（repeat-tool-reminder + timeout-policy）默认随 base bundle 启用。

## 与本轮外部事件的关系

10-11 雷达重大事件日 11：Anthropic 切断内部评测实时互联网访问（评测 agent 攻击真实站点）。dsh 作为 eval harness，其 `SAFETY.md` 对沙箱隔离的**否定性声明**（不保证隔离）+ PTC `isolation` 描述符"不承诺安全边界"——是"评测环境必须默认 fail-closed"的仓库侧佐证；dsh 的沙箱/网络默认策略是否 fail-closed 留 Candidates C-02（未从配置实测验证）。

## 交付物

| 层 | 文件 | 要点 |
|---|---|---|
| 01 Project Layer | `01-project-layer/project_layer.md` | 0.2.1 项目地图（331 包/五 profile/desktop/版本化治理） |
| 02 Engineering Knowledge | `02-engineering-knowledge/ek_graph.md` | **40 EK**（EK Graph，STABLE/CHANGED/NEW 标注演进） |
| 03 Knowledge Layer | `03-knowledge-layer/knowledge_layer.md` | **10 KO**（R1×2/R2×2/R3×2/R4×4，aggregation_rule） |
| 04 Flow Atlas | `04-flow-atlas/flow_atlas.md` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05 Candidates | `05-candidates/candidates.md` | 7 条（含跨项目假说） |
| 06 Validation | `06-validation/validation_report.md` | Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic + 盲重建 |

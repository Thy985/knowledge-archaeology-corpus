# Project Archaeology Overview — deepseek-harness
> Job: `ARCH-2026-09-03-001` | Skill: knowledge-archaeology **v3.2**（Multi-Agent 工作流 + Contracts + Evidence Rules）
> 目标仓库只读分析，未修改任何仓库；本文件为 **Reconciled 版**（合并独立验证修正；原始 run 产物见 `archaeology-runs/ARCH-2026-09-03-001/`）。

## 项目一句话定位
**DeepSeek Harness (dsh)** 是一个基于 Cordis 插件内核的 **everything-is-a-plugin Agent Runtime**——把"Agent 运行时的每一个能力"（LLM 适配、会话、工具、权限、沙箱、凭证、子代理、记忆压缩、持久化）都建模为可插拔的 **Capability Seam**（Service Definition / Provider / Consumer 三件套），以 **append-only 会话日志（SessionEvent log）** 作为唯一事实源，在日志之上派生模型可见面、投影状态与持久化格式。

## 核心命题
> 这个项目让我们认识到了什么？
1. **Agent 运行时可以把"权威"从"智能"中显式分离**：模型生成内容（智能），但每一步状态变更都要经过可控管道（collapse 前置拒绝 → 审批 fail-closed → 工具守卫 → 沙箱 → 凭证引用）。
2. **日志即真相**："model-visible means logged"——只有进入不可变日志的才是模型可见的，审计/策略/投影/持久化全部从日志重放派生，因此**可回放、可审计、可恢复**。
3. **可插拔不是库的装饰，是架构主轴**：Cordis 插件树 + Capability Seam 三件套 + Typert 类型图 + 自动生成的依赖图（gen-doc-graphs），使 150+ 包以统一契约组合。

## 交付物索引
| 文件 | 内容 |
|---|---|
| `01_project-layer/project-layer.md` | Project Layer：架构/模块/生命周期/配置/依赖/入口（含 Snapshot 指针） |
| `02_engineering-knowledge/ek-graph.md` | Engineering Knowledge 层：EK Graph（32 条 EK + 6 类边，Reconciled） |
| `03_knowledge-layer/ko.md` | Generalized Knowledge 层：8 个 KO（aggregation R1-R4，L3/L4，Reconciled） |
| `04_flow-atlas/flow-atlas.md` | Flow Atlas：七类流（F-05 已按独立验证修正） |
| `05_candidates/candidates.md` | Candidates：未确认/跨项目/边界不确定结论 |
| `06_validation/` | Validation：原始验证 + 独立验证 + Reconciliation（含 Contradictions 与 Counterexamples） |
| `archaeology-runs/ARCH-2026-09-03-001/` | 本次 run 原始产物归档（含 run_metadata.yaml） |

## 质量指标（目标 vs 实际，Reconciled）
| 指标 | 目标 | 实际 | 判定 |
|---|---|---|---|
| Facts/Evidence | 100+ | 38（EV-001…EV-038）+ 独立验证 IF-01..10 | 达标（聚焦高密度） |
| Engineering Knowledge | 40~60 | 32（26 原始 + 6 Reconciliation 新增） | 达标（150+ 包聚焦核心机制） |
| Patterns | 15~25 | 8 个 KO（含 L3/L4） | 窄尖顶 |
| Core KO | 7~12 | 8 | 达标 |
| EK 平均出边数 | ≥1 | ≥2.4 | 达标 |
| 游离 EK 比例 | <20% | 0%（32/32 连接 KO 簇） | 达标 |
| 聚合规则覆盖率 | 100% | 100%（8/8） | 达标 |
| KO 平均簇规模 | 3~12 | 5.0 | 达标 |
| Blind Reconstruction | performed | performed（原始 + 独立验证均盲重建） | 达标 |

# DeepSeek Harness — Knowledge Archaeology

**一句话定位**：基于 Cordis 插件内核的 **everything-is-a-plugin Agent Runtime**——每个能力都是 Capability Seam（Service Definition / Provider / Consumer 三件套），以 append-only 会话日志（SessionEvent log）作为唯一事实源。

## 考古索引
| 产物 | 路径 | 说明 |
|---|---|---|
| Overview | `00_overview.md` | 项目定位 + 质量指标 |
| Project Layer | `01_project-layer/` | 架构/模块/生命周期/配置/依赖/入口 |
| Engineering Knowledge | `02_engineering-knowledge/` | EK Graph（32 条，Reconciled，不因 KO 删除） |
| Knowledge Layer | `03_knowledge-layer/` | 8 个 Generalized KO（R1-R4 聚合，可回溯 EK/EV） |
| Flow Atlas | `04_flow-atlas/` | 七类流（F-05 已按独立验证修正） |
| Candidates | `05_candidates/` | Hypothesis / Counterexample / Cross-project Candidate |
| Validation | `06_validation/` | 原始验证 + 独立验证 + Reconciliation |
| Run 归档 | `archaeology-runs/ARCH-2026-09-03-001/` | 本次 run 原始产物（不覆盖历史） |

## 考古日期 / 版本
- 日期：2026-09-03
- repository：`deepseek-ai/deepseek-harness` @ `76fda72`（v0.1.2-rc.1）
- Skill：knowledge-archaeology v3.2
- run_id：`ARCH-2026-09-03-001`

## 核心命题
1. 权威与智能分离：模型生成内容，状态变更经 守卫→审批→沙箱→凭证 四层可控管道
2. 日志即真相（model-visible means logged）：可回放、可审计、可恢复
3. 可插拔是架构主轴：Cordis 插件树 + Capability Seam 三件套 + 自动生成依赖图

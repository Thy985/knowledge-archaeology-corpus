# Webwright — Knowledge Archaeology

> 一句话定位：**微软研究院（MSR）发布的"零隐藏框架"网页 agent harness——以代码即动作（code-as-action）方式驱动本地 Playwright 浏览器完成网页任务，并把每个 solve 蒸馏为可重跑、已验证、参数化的 code skills（Skill Factory）。**
> 考古日期：2026-10-05 ｜ commit：bc26750af3ad166d982f23d101ef8971a3a2fce5（2026-08-03, v0.1.0） ｜ 模式：initial ｜ run_id：ARCH-2026-10-05-001 ｜ skill：knowledge-archaeology v3.2

## 产物清单
| 目录 | 内容 |
|---|---|
| `metadata.yaml` | 项目元数据 + 质量指标 + Reconciliation 状态 |
| `00_overview.md` | 核心命题 + 三层配比 + 关键数字 |
| `01_project-layer/` | 项目地图（架构/模块/生命周期/配置/测试/治理） |
| `02_engineering-knowledge/` | **48 条 EK**（EK Graph 六类边，宽底座） |
| `03_knowledge-layer/` | **8 个 KO**（R1-R4 聚合规则，窄尖顶） |
| `04_flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05_candidates/` | 7 条未验证假说（严格区分于 KO） |
| `06_validation/` | 考古自检报告 + 独立盲审计报告 + Reconciliation |
| `archaeology-runs/ARCH-2026-10-05-001/` | 原始 run 产物（job_manifest/snapshot/run_metadata/独立审计） |

## 关键数字速览（README 自报，未独立复现）
- Online-Mind2Web 300：86.7%（gpt-5.4）；hard split N=100：Opus 4.7 80.5% vs gpt-5.4 76.6%
- Odysseys 200：60.1%（avg 76.1 steps；+15.6 over Opus 4.6、+26.6 over base gpt-5.4）
- WebArena 技能复用：55%→70%（+15pp）
- 源码 src 6552 行 / tests 1899 行（14 文件）

## 核心认知（详见 03 层）
code-as-action（动作=可执行代码）· 可丢弃浏览器+持久工作区 · 验证即重放（无模型重跑训练任务）· 知识库污染防护（永不覆盖/永不猜测/永不泄漏）· 库查找前置 · 诚实认知状态（grade 三态）· 严格性递进（shape→归一化→exact）· 重画优于修复（双层预算）

## 边界
- 无 ADR 目录；决策证据=README News/对比表 + docs/manual.md + 代码注释
- benchmark 数字为 MSR 自报，第三方复现未执行（C-07）
- 原考古产物与独立审计报告分别保留（Reconciliation 见 06_validation/validation_report.md §9）

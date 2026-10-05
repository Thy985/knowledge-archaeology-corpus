# langgraph — Knowledge Archaeology

**一句话定位**：低层有状态多 Actor Agent 编排框架（v1.2.13，MIT）——把 agent 定义为有状态图（节点=执行单元，边=状态转换，channel=状态聚合语义，checkpointer=线程持久化，interrupt=人机交互挂起/恢复）。官方背书企业用户 Klarna / Replit / Elastic。

## 产物清单
| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 考古元数据（run_id / commit / skill_version / quality metrics） |
| `00_overview.md` | 概览：核心命题（版本化状态机）+ 三层知识架构 + 关键认知锚点 |
| `01_project-layer/` | 项目地图（架构/模块/数据结构/生命周期/配置/治理/依赖） |
| `02_engineering-knowledge/` | EK Graph（48 条 + 六类边，游离 0） |
| `03_knowledge-layer/` | Generalized KO（8 个，R1-R4 聚合规则） |
| `04_flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05_candidates/` | 未验证假说（7 个 + 2 跨项目连接） |
| `06_validation/` | 验证报告（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic + Blind Reconstruction）+ 独立审计 + Reconciliation |
| `archaeology-runs/ARCH-2026-10-06-001/` | 本次 Run 原始产物（manifest/snapshot/metadata/审计） |

## 考古日期与版本
- 考古日期：2026-10-06（run ARCH-2026-10-06-001，mode: initial）
- 目标版本：langgraph 1.2.13 @ `2d942085e214ef6b99b6f54ed4d544a7c7c5ac56`（2026-10-05）

## 质量指标
Facts 100+（全部可回溯 file:symbol）→ EK 48（links 96，游离 0）→ KO 8（聚合规则 100%）→ 7 类流 → 7 Candidates。Validation：17+3 条 Truth 断言全 CONFIRMED；Blind Reconstruction ×3 一致；独立审计 0 CONTRADICTED / 3 处补充修正（resume 语义、执行层、stream v3，已 Reconciliation）。

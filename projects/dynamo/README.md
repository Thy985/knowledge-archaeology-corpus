# dynamo — Knowledge Archaeology

NVIDIA 开源数据中心级推理栈（v1.6.0，Apache-2.0）——推理引擎（SGLang/TRT-LLM/vLLM）之上的编排层：KV 感知路由 + 调度 + 多层级 KV 缓存 + 自动扩缩。

## 产物清单
| 文件 | 内容 |
|---|---|
| [00_overview.md](00_overview.md) | 项目一句话 + 核心命题 + Top5 发现 + 质量指标 |
| [01_project-layer.md](01_project-layer.md) | L0 项目事实（F-01~F-36，全部可回溯 commit `6562e7d`） |
| [02_engineering-knowledge.md](02_engineering-knowledge.md) | EK Graph（36 条，≥76 边，0 游离） |
| [03_knowledge-layer.md](03_knowledge-layer.md) | KO 层（9 个：L2×1/L3×6/L4×2，aggregation 100%） |
| [04_flow-atlas.md](04_flow-atlas.md) | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| [05_candidates.md](05_candidates.md) | 未验证候选（Hypothesis×4 / Observation×5） |
| [06_validation.md](06_validation.md) | 六类 Auditor 判定 + 反例 + 质量指标 |
| [06_validation-independent-audit.md](06_validation-independent-audit.md) | 独立审计（30 CONFIRMED / 4 MISSING / 1 OVERGEN / 1 NEEDS_HUMAN_REVIEW） |
| [06_validation-reconciliation.md](06_validation-reconciliation.md) | Reconciliation（R-1~R-7） |
| [archaeology-runs/ARCH-2026-09-20-001/](archaeology-runs/ARCH-2026-09-20-001/) | 本次 run 元数据（run_metadata.yaml） |

## 考古记录
- **run_id**: ARCH-2026-09-20-001（initial）
- **commit**: `6562e7d04b04ad177459f8a5b720718de71e1a6a`（2026-09-19）
- **skill_version**: knowledge-archaeology v3.2
- **质量指标**: Facts 36 / EK 36（≥76 边）/ KO 9 / Candidates 9 / Flows 7 / 反例 22

## 核心命题
"agent 的推理成本与延迟如何成为可工程化的系统问题？"——Agent Hints（nvext）/ KV 价值分层 / KV 全局共享化 / Flash Indexer / 调度双路径。

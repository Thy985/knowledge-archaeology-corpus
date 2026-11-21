# SemanticRun — Knowledge Archaeology

> **一句话定位**：artifact-aware durable agent environment——"freezes what your agent already committed — and resumes with proof of what drifted"（语义检查点流派，durable-execution corpus 第三派代表）。

| 项 | 值 |
|---|---|
| 考古日期 | 2026-11-05（run_id `ARCH-2026-11-05-001`） |
| 仓库 | https://github.com/vogue84/SemanticRun.git |
| Commit | `8d99a8143e868f1760e61ba8260909e5e8b2ca29`（2026-08-04） |
| 版本 | 0.4.2（PyPI `semanticrun`） |
| Skill 版本 | knowledge-archaeology v3.2 |
| 证据上限 | S4（本机 pytest 44/44 通过） |

## 产物清单

- [00_overview.md](00_overview.md) — 定位 + 核心命题验证表
- [01-project-layer/](01-project-layer/) — 项目地图（架构/模块/生命周期/配置/治理）
- [02-engineering-knowledge/](02-engineering-knowledge/) — Engineering Knowledge 层（22 EK，EK Graph 六类边）
- [03-knowledge-layer/](03-knowledge-layer/) — Generalized KO 层（8 KO，R1-R4 聚合规则 + 2 CM + 1 M）
- [04-flow-atlas/](04-flow-atlas/) — 七类流（23 edge，可回溯）
- [05-candidates/](05-candidates/) — 未验证假说（6 条，含跨项目 Candidate）
- [06-validation/](06-validation/) — Validation（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic）
- [archaeology-runs/ARCH-2026-11-05-001/](archaeology-runs/ARCH-2026-11-05-001/run_metadata.yaml) — run 元数据 + 质量指标

## 关键发现（摘要）

1. **流派空白填补**：语义恢复（diff committed artifacts on resume）与事件重放（durare/temporal）、挂起恢复（infinity）构成 durable-execution 三派对照。
2. **核心机制全部 S4**：同步 checkpoint 屏障 / 哈希承诺 / 分歧矩阵（机械布尔 no LLM guessing）/ ACRFence 外发重放门 / 显式策略闭环。
3. **安全边界如实保留**：NO_PRIOR_RECORD 首重放放行；read_only 分类是信任边界（三条件绕过）；approve/reject 无身份鉴权（NEEDS_HUMAN_REVIEW）。
4. **README 营销口径与实现分离**：性能声明（98% token savings / 1-8ms）无仓库内证据；"5/5 safe" 精确含义为 5 场景 × 自定义策略子集。
5. **雷达误记校正**：KnowlegeMap 11-03 所记 "PyPI 10-21 更新" 不存在——semanticrun 全部 release 上传于 2026-08-04。

## Validation 结论（独立 Auditor 盲重建）

CONFIRMED 11 / PARTIALLY_CONFIRMED 4 / DOWNGRADED 1（EK-02 证据等级 S4→S3）/ MISSING 3 / CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 2。
详见 [06-validation/](06-validation/06_validation.md) 与 `archaeology-runs/ARCH-2026-11-05-001/`（独立 Auditor 报告随 run 归档）。

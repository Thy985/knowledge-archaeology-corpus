# SkillFortify — Project Archaeology Index

> 一句话定位：用形式化方法（而非启发式规则）做 AI Agent Skill 供应链安全静态分析的扫描器。核心命题——"若扫描器报告无违反，则形式化模型内的能力边界被保证成立（sound）"。

## 产物清单

| 目录/文件 | 内容 |
|-----------|------|
| `00_overview.md` | 考古总览（定位/价值/关键数字/免责声明） |
| `01_project-layer/` | 项目地图（身份/结构/模块/数据结构/状态/测试/配置/治理/依赖） |
| `02_engineering-knowledge/` | EK Graph（36+2 条 EK，6 类边）+ Reconciliation 补充 |
| `03_knowledge-layer/` | Generalized KO（9 KO R1-R4 / 4 CM / 4 M） |
| `04_flow-atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05_candidates/` | 9 条未确认假说（含 C-01 NEEDS_HUMAN_REVIEW） |
| `06_validation/` | Validation + Independent Validation Report + 反例 + 质量指标 |
| `archaeology-runs/ARCH-2026-09-05-001/` | 本次 run 完整快照（00-07 + job manifest + metadata） |

## 考古信息

- **项目**: skillfortify (qualixar/skillfortify)
- **版本**: v0.6.0（Elastic-2.0，Alpha）
- **commit**: dbb5942deae46afdb5b85440d5f5d851c197c30e
- **run_id**: ARCH-2026-09-05-001（initial, priority S）
- **考古日期**: 2026-09-05
- **skill_version**: knowledge-archaeology v3.2
- **source_entry**: `03_expansion_queue/candidates/[cand]skill-supply-chain-security-2026.md`

## 认知核心（可检索摘要）

1. **Soundness 由保守过度近似而非精确性保证**（KO-01/CM-1）——"无假阴性"来自有界 over-approximation + 显式边界。
2. **召回型检测 ≠ sound 保证**（KO-01）——Phase2 模式检测可有假阴性，Phase3 能力核对无假阴性。
3. **权限可信性 = 声明-实际核对 + 权限单调**（CM-2）——POLA 机械化 + lattice join 不放大。
4. **信任模型需满足代数不变量**（CM-3）——单调性 + 最弱链 + 时间衰减。
5. **评估器可信度取决于基准不可投机**（CM-4）——anti-leakage 机械检查。

## 关键争议（需 owner 关注）

- **C-01（NEEDS_HUMAN_REVIEW）**：Formal-Foundations 声称能力推断 "without over-approximation"，实现却是 conservative over-approximation（恰是 soundness 来源）。需确认论文（arXiv 2603.00195）是否同样失准。
- **C-06（Fact）**：候选卡记录 MIT，实际 Elastic-2.0——KnowlegeMap 候选卡待修正（本 run 不写 KnowlegeMap）。

# SkillFortify — Project Archaeology Overview

- **run_id**: ARCH-2026-09-05-001
- **project**: skillfortify
- **repository**: https://github.com/qualixar/skillfortify.git
- **commit**: dbb5942deae46afdb5b85440d5f5d851c197c30e（main, tag v0.6.0）
- **skill_version**: knowledge-archaeology v3.2
- **timestamp**: 2026-09-05（UTC+8）
- **mode**: initial

## 一句话定位

**SkillFortify 是一个用形式化方法（而非启发式规则）做 AI Agent Skill 供应链安全静态分析的扫描器**：核心命题是"如果扫描器报告无违反，则形式化模型内的能力边界被保证成立（sound）"，并通过 能力格 + DY-Skill 威胁模型 + 声明vs实际能力核对 + 信任代数 + SAT 依赖解析 + 540-skill 防泄漏基准 六个构件实现该保证。支持 22 个 Agent 框架。

## 为什么值得考古（Knowledge Value）

1. **Agent/AI 工程相关性（S 级）**：面向"Agent skills 是新的软件依赖"这一 2026 现实（ClawHavoc 1184 恶意 skill / CVE-2026-25253 / MalTool 7027 恶意工具），是 Agent 供应链安全的核心命题，与昨日 RAMPART（运行时安全测试）构成 AI 安全攻防专题连续考古。
2. **Novelty**：首个把 Dolev-Yao 攻击者模型适配到 skill 供应链（DY-Skill，五阶段生命周期 × 五攻击能力）并声明五条 soundness 定理的工具（arXiv 2603.00195）。
3. **Benchmark 学习价值**：SkillFortifyBench（540-skill，seed=42 确定性生成，**机械检查结构特征不能预测标签**的 anti-leakage 设计）本身是"如何构造不可作弊的 Agent 安全基准"的高价值工程样本。
4. **Design-Reality Gap 样例**：Formal-Foundations 文档声称"capability inference 无过度近似（exact）"，而实现（analyzer/engine.py `_infer_capabilities`）明确是"保守过度近似（sound abstract interpretation），假阳性可接受、假阴性不可接受"——文档措辞与实现矛盾，是设计意图 vs 实现现实的典型考古对象。

## 认知核心（一句话）

**Soundness 的正确来源不是"精确分析"，而是"有界过度近似 + 显式声明边界"**：SkillFortify 用保守 over-approximation 换取无假阴性（Theorem 2），用能力格保证 join 可计算，用 DY-Skill 形式化保证攻击空间覆盖（Theorem 1），用 trust min 传播保证最弱链（Theorem 5），并用 benchmark anti-leakage 检验自身不靠结构特征投机。

## 关键数字（均可回溯）

| 指标 | 值 | 来源 |
|------|-----|------|
| 版本 | v0.6.0（Alpha，Elastic-2.0） | pyproject.toml |
| 代码规模 | 950 文件 / Python 52,270 行 / 333 py | find+wc |
| 支持框架 | 22 | README.md |
| 测试文件 | 182 | find tests |
| CLI 命令 | 8（scan/verify/lock/trust/sbom/frameworks/dashboard/registry-scan） | cli/main.py |
| SkillFortifyBench | 540 skill（270 恶意 13 攻击类型 + 270 良性 5 类） | README/benchmarks |
| 声明定理 | 5（Completeness/Soundness/Non-Amplification/Resolution/Trust Monotonicity） | wiki/Formal-Foundations.md |
| 已知局限 | 4（install-time 6% recall gap/runtime/semantic/obfuscation） | Formal-Foundations §Known Limitations |

## 产物清单

- `01_project-layer/` — 项目地图（架构/模块/数据结构/状态/配置/治理/依赖）
- `02_engineering-knowledge/` — EK Graph（36 条 EK + 6 类边）
- `03_knowledge-layer/` — Generalized KO（9 个 Core KO + 聚合规则 R1-R4）
- `04_flow-atlas/` — 七类流（Control/State/Data/Evidence/Authority/Memory/Policy）
- `05_candidates/` — 未确认假说（含跨项目候选与文档-实现矛盾）
- `06_validation/` — Validation 报告 + 反例 + Reconciliation + 质量指标
- `run_metadata.yaml` — run 元数据

## 免责声明

本包所有事实均可回溯到 dbb5942 快照的实际内容（代码/文档/测试）。验证阶段"真实代码行为"来自直接执行 repo 中模块的实测结果；"文档声称"与"实现现实"严格区分（见 05_candidates 与 06_validation）。

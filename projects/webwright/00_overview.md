# 00 Overview — Webwright (microsoft/Webwright)

- run_id: ARCH-2026-10-05-001 ｜ commit: `bc26750af3ad166d982f23d101ef8971a3a2fce5`（2026-08-03）｜ version 0.1.0
- skill_version: knowledge-archaeology v3.2 ｜ mode: initial ｜ 2026-10-05

## 一句话定位
**Webwright 是微软研究院（MSR）发布的一个"零隐藏框架"网页 agent harness：给 LLM 一个终端，让它以"代码即动作"（code-as-action）方式逐条 bash/python 命令驱动本地 Playwright 浏览器完成网页任务，并强制把每个任务沉淀为一个可重跑的 Python 脚本（state=local workspace，browser=disposable，loop=write code→execute→inspect screenshots→repair）。**

## 它让我们认识到了什么（核心命题）
1. **代码即动作（code-as-action）是坐标预测（xy-coordinate）之外的第三条网页 agent 决策路线**：模型直接生成可执行代码而非点击坐标；code 是状态、script 是产物、workspace 是记忆。README 基准：Online-Mind2Web 86.7%（gpt-5.4）、Odysseys 60.1%（+15.6 over Opus 4.6、+26.6 over base gpt-5.4）。
2. **技能工厂（Skill Factory）把 solve 蒸馏成可复用资产**：每个 solve 留下一段可重跑脚本，经 gate→分组→蒸馏→无模型重放验证→grade 三态入库；技能库查找在 agent loop **之外**解析并注入 prompt（fail-open）。WebArena reuse 55%→70%。
3. **验证 = 重放（replay）**：技能入库前必须在临时目录无模型重跑训练任务，answer 文件是契约而非退出码；strict/shape 两级验证对应答案漂移与否。
4. **可丢弃浏览器 + 持久工作区**：浏览器状态不跨 step 保持（除非显式 persistent 模式），证据（screenshots/logs/code）全部落在 workspace——网页任务的"记忆"在文件系统而非浏览器上下文。

## 三层知识配比
- Project Layer：01_project-layer.md（项目地图）
- Engineering Knowledge：02_engineering-knowledge.md（**48 条 EK，EK Graph 六类边**）
- Generalized Knowledge：03_knowledge-layer.md（**8 个 KO，聚合规则 R1-R4**）
- Flow Atlas：04_flow-atlas.md（**七类流**）
- Candidates：05_candidates.md（7 个未验证/跨项目假说）
- Validation：06_validation.md（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic + Blind Reconstruction 记录 + Contradictions/Counterexamples 保留）

## 关键数字（全部来自 README/代码原文，可回溯）
| 项 | 值 | 来源 |
|---|---|---|
| Online-Mind2Web (300) | 86.7% gpt-5.4；Opus 4.7 84.7% | README §Benchmarks |
| hard split (N=100) | Opus 4.7 80.5% vs gpt-5.4 76.6% | README §Benchmarks |
| Odysseys (200) | 60.1% gpt-5.4，avg 76.1 steps | README §Benchmarks |
| 对照 | +15.6 over Opus 4.6 (44.5%)；+26.6 over base gpt-5.4 (33.5%) | README §Benchmarks |
| WebArena reuse | 55%→70%（+15pp） | README News 2026-07-21 |
| 技能 standalone | ~40s，zero tokens | README |
| 源码规模 | src 6552 行（skill_factory 2150） | wc -l 实测 |
| 测试规模 | tests 1899 行，14 文件 | wc -l 实测 |
| 轨迹对比 | Harness Local 424,026 tokens vs Codex Skill 3,291,183 | README compare_trajectory |
| 首次发布 | 2026-05-04（~1.5k LoC） | README News |

## 边界声明
- 只读考古：未修改目标仓库/KnowlegeMap/生产 Skill
- 无 ADR 目录；决策证据 = README News/对比表 + docs/manual.md + 代码注释
- benchmark 数字为 MSR 自报（README），独立复现性未验证（见 C-07）
- 本包为 initial run；corpus 写入见阶段⑥

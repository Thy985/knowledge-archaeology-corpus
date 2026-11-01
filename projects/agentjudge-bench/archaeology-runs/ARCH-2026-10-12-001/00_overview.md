# 00 Overview — AgentJudgeBench（ARCH-2026-10-12-001）

> run-id: `ARCH-2026-10-12-001` ｜ 2026-10-12 ｜ mode: initial ｜ skill: knowledge-archaeology v3.2
> repository: ServiceNow/SyGra @ `scratch/agent_judge_bench` ｜ commit `97cca4fa07e98ee61bb3a1a2f8639778e299d50a`（2026-05-06）

## 一句话定位

**AgentJudgeBench = 一个"评测评测者"的基准**：在 3,808 条 BFCL 风格 agent tool-calling 轨迹上，用**确定性程序化 judge** 作为 ground truth，系统量化 6 个 LLM judge（20B→frontier）与真值对齐度随 3 档难度 × 6 种 DAG 拓扑 × 5 个生成器 × 有无 ground-truth 条件的衰减规律。核心命题："LLM 当 judge 评 agent，到底有多可靠？"

## 认知价值（这个项目让我们认识到什么）

1. **程序化 judge 是评测可信度的锚**——本项目把"判定"从 LLM 自由打分降为可复算的符号公式（Eqs 1-3），再用 LLM 与它的偏差度量 judge 可靠性（Eq 4）。"评测者也需要被评测"被工程化。
2. **GT 是一把双刃剑**——with-GT / without-GT 双分支 + C3 corrupted-GT 控制组的设计，直接实证"喂给 judge 的信息不是越多越好"（anchoring / over-reliance）。
3. **结构化输出契约是评测基础设施**——prompt 强约束四维 JSON 指标 + 保守默认 + safe_parse_json 容错，是"评测管道在真实模型上可跑 321,648 次"的工程前提。
4. **评测基础设施与评测逻辑解耦**——SyGra LangGraph 图运行时承载任务定义，run 脚本只 patch 参数，任务/图/运行三分。

## 实验规模（仓库 README 实测）

3,808 records × 3 difficulties × 5 generators × 6 judges × 2 conditions = **321,648 evaluations**

- 生成器：SmolLM3-3B / Llama-3.1-8B-Instruct / Qwen3-32B / Llama-3.3-70B-Instruct / GPT-5.4
- judge：GPT-OSS-20B / QwQ-32B / GPT-OSS-120B / Claude Sonnet 4.5 / Gemini 2.5 Pro / GPT-5.4
- 难度：easy / medium / hard（HF 数据集 query / query_medium / query_hard 列）
- 条件：C1 main（with/without GT，1.0-default）/ C2（without-GT 0.5-default）/ C3（corrupted GT）/ RQ6（freeform）

## 核心结论链（论文 2608.26623 主张，代码可复现路径）

- judge 对齐度随任务难度**单调下降**（难度越难 judge 越不可靠）
- **无 ground-truth 时退化快 1.5x**（judge 自我评测在难任务上崩塌）
- **喂 GT 反而可能损害对齐**（over-reliance / anchoring，C3 控制组设计即为此）

## Package 内容索引

| 文件 | 内容 |
|---|---|
| 01_project-layer.md | 项目地图（结构/模块/生命周期/配置/依赖） |
| 02_engineering-knowledge.md | EK Graph（14 条 EK，六类边，宽底座） |
| 03_knowledge-layer.md | KO 层（5 个 KO，R1-R4 聚合规则） |
| 04_flow-atlas.md | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates.md | 未验证假说与跨项目候选 |
| 06_validation.md | 六类审计 + 质量指标 + Reconciliation |
| run_metadata.yaml | run 元数据 |

## 交付路径

- 本 Package 写入 corpus：`projects/agentjudge-bench/`（initial）
- PR body 将含：项目/run id/archaeology summary/validation summary/quality metrics/关键发现/关键争议/SIP

# 00 Overview — SemanticRun（ARCH-2026-11-05-001）

## 一句话定位

**SemanticRun 是"语义检查点"流派的 artifact-aware durable agent environment**：不重放 workflow 代码（Temporal 路径）、不编排 agent graph（LangGraph 路径），而是在恢复时**对已提交的工件（artifacts）做机械 diff，并用显式策略强制分歧处理**——README 原文："Temporal replays code. LangGraph orchestrates graphs. **SemanticRun freezes what your agent already committed — and resumes with proof of what drifted.**"

## 为什么选这个项目（Job Manifest 摘要）

- **流派空白**：durable execution corpus 已有"事件重放"（durare/dbos/temporal/restate/dapr）与"挂起恢复"（infinity）两派，**语义恢复第三派完全空白**
- **与已有 corpus 构成直接对照**：durare（2026-11-04 考古，事件重放 + 哈希验证）↔ SemanticRun（语义检查点 + 分歧矩阵）构成同一问题（长运行 agent 恢复）的两种解法 contrast 边
- **Benchmark 学习价值**：验证第三派是真实工程还是 README 营销；校正 KnowlegeMap 雷达误记（"PyPI 10-21 更新"不存在，实际全部 release 于 07-30/08-04）
- 规模小（Python 2920 行 + tests 786 行）→ 按 v3.2 小项目裁剪规则全量考古，44 tests 实测全过（S4）

## 核心命题（验证结论预览）

| 声明（README） | 考古验证 | 证据强度 |
|---|---|---|
| "Drift recovery 5/5 safe"（vs naive restart 0/5、blind resume 0/5） | 机制完全实现：DivergenceMatrix + PolicyRouter + ACRFence，5 场景均有对应 flag 与 policy；**但 benchmark 默认走确定性 stub，且 POLICIES 为自定义子集（tool→revalidate / model→fail_fast / file_tree→fail_fast / outbound→fail_fast），未覆盖 intent/plan/approval/behavioral 四类 flag 场景** | S4（tests）/ S5 缺口（真实模型路径未在本环境验证） |
| "Diff committed artifacts on resume" | 实现：ResumeArtifacts（host 注入）vs Checkpoint（tool_schemas/tool_results/file_tree/model_id/intent/plan/approval/outbound）机械 diff | S4 |
| "No LLM guessing"（分歧矩阵） | 实现：全部纯哈希/字符串对比；behavioral drift 必须 host 显式标记 | S4 |
| "Sync checkpoints — written before the step returns" | 实现：_enqueue_checkpoint + worker.drain() 同步等待 | S4 |
| token savings up to 98% / checkpoint 1-8ms / mem<4MB | **无仓库内证据**（drift suite 只测安全不测性能）——标 Observation，不升 Fact | S0/S3（README 声明） |

## 交付物清单

| # | 文件 | 内容 |
|---|---|---|
| 00 | 本文件 | Overview |
| 01 | 01_project_layer.md | 项目地图（架构/模块/生命周期/依赖/入口） |
| 02 | 02_engineering_knowledge.md | Engineering Knowledge 层（EK Graph，22 条 EK，六类边） |
| 03 | 03_knowledge_layer.md | Generalized KO 层（8 KO，R1-R4 聚合规则 + L3/L4 升维 + 反例） |
| 04 | 04_flow_atlas.md | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy，edge 可回溯） |
| 05 | 05_candidates.md | 未验证假说（6 条，含跨项目 Candidate） |
| 06 | 06_validation.md | Validation 报告（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic） |
| — | run_metadata.yaml | run_id/commit/skill_version/质量指标/validation result |

## 质量指标（详见 run_metadata.yaml）

- EK 22 条，全部声明 links（平均出边 >1），游离 EK 0%
- KO 8 个，聚合规则覆盖率 100%，簇规模 3~6（健康区间 3~12）
- 七类流 23 条 edge，全部可回溯到 symbol/file
- 证据等级上限：S4（本机 pytest 44/44 通过）；benchmark 真实模型路径标 S5 缺口
- 关键认知状态：**Hypothesis 不冒充 Fact；README vendor 声明（98%/5-5 营销口径）与实现事实严格分离**

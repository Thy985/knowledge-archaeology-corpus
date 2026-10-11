# AgentJudgeBench — Knowledge Archaeology

> 考古日期：2026-10-12 ｜ mode: initial ｜ run: `ARCH-2026-10-12-001` ｜ skill: knowledge-archaeology v3.2

## 一句话定位

**AgentJudgeBench = 一个"评测评测者"的基准**：在 3,808 条 BFCL 风格 agent tool-calling 轨迹上，用确定性程序化 judge 作为 ground truth，系统量化 6 个 LLM judge（20B→frontier）与真值对齐度随难度/拓扑/生成器/有无 GT 的衰减规律。

- 论文：arXiv 2608.26623（2026-08-27）
- 代码：https://github.com/ServiceNow/SyGra.git `scratch/agent_judge_bench`（commit `97cca4fa`）
- 数据集：HF `ServiceNow-AI/AgentJudgeBench`（3,808 records）
- 实验规模：3,808 × 3 × 5 × 6 × 2 = **321,648 evaluations**

## 产物清单

| 目录/文件 | 内容 |
|---|---|
| `metadata.yaml` | 项目元数据（commit/skill 版本/质量指标/验证统计） |
| `archaeology-runs/ARCH-2026-10-12-001/` | 完整考古包（11 文件） |
| └ `00_overview.md` | 定位/规模/认知价值 |
| └ `01_project-layer.md` | 项目地图 |
| └ `02_engineering-knowledge.md` | EK Graph（15 条，六类边） |
| └ `03_knowledge-layer.md` | KO 层（5 个，R1-R4） |
| └ `04_flow-atlas.md` | 七类流 |
| └ `05_candidates.md` | 7 个候选（含 C-07 analyze 管道缺陷） |
| └ `06_validation.md` | 六类审计 + 质量指标 + Reconciliation |
| └ `job_manifest.md` | Job Selection（候选排序 + 选中理由） |
| └ `snapshot_artifact.md` | 仓库快照 + 基础地图 |
| └ `run_metadata.yaml` | run 元数据 |
| └ `independent_validation_report.md` | 独立盲审计（IF-1~7 + 判定统计 + 3 成功 3 错误 + Benchmark） |

## 关键认知（KO 速览）

- **KO-01 程序化锚定（L4）**：评测可信 = 可复算真值 + 对齐度量——"评测者先被评测"
- **KO-02 提示词是校准器（L4, Cross-project pending）**：默认分档 + 结构化 rubric 是对齐度的可操作校准旋钮
- **KO-03 无 GT 信息降级（L3, Hypothesis）**：schema-as-truth + 宽容默认
- **KO-04 GT 污染控制（L3）**：derangement 探针——探针与被探针隔离
- **KO-05 评测基础设施分层（L4）**：任务/图/运行解耦；条件靠命名约定治理（约定有缺陷，见 C-07）

## 关键发现与争议

- **关键发现**：analyze_results.py glob（4 层 `*/judged_by_*/*/*.jsonl`）与 BaseTaskExecutor 实际输出（3 层单文件 `judged_by_<judge>/<run_name_prefix>output_<ts>.jsonl`）不兼容 → **默认复现流程 analyze 匹配不到文件**（C-07，独立审计确认）。
- **关键争议**：程序化 judge 语义盲区（不查参数值正确性，C-01）；C3 derangement 同 dag_type 语义相关性未控（C-02）。
- **未产生 Skill Improvement Proposal**：无系统性 skill 缺陷。

## 验证结论

- 主考古六类审计：PASS（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic）
- 独立盲审计判定：CONFIRMED 15 / PARTIALLY_CONFIRMED 4 / MISSING 2 / DOWNGRADED 0 / OVER_GENERALIZED 0 / CONTRADICTED 0 / NEEDS_HUMAN_REVIEW 0
- Reconciliation：7 项修正全部并入（EK-15 缺陷标注 / Flow Atlas 不匹配边 / C-07 / 01 补 Observation / KO-02 pending / EK-05 措辞 / 02 计数）

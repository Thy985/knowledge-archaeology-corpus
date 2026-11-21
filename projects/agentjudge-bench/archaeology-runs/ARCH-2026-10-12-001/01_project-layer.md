# 01 Project Layer — AgentJudgeBench（ARCH-2026-10-12-001）

> 项目地图（L0/L1 事实底座）。全部事实可回溯 `archaeology-jobs/ARCH-2026-10-12-001/repo/`。

## 1.1 项目定位

- **名字**：AgentJudgeBench（A Multi-Difficulty Benchmark for Evaluating LLM Judges on Agentic Tool-Calling）
- **载体**：ServiceNow/SyGra 仓库的 `scratch/agent_judge_bench` 分支 —— SyGra 主项目（图式合成数据管道，arXiv 2508.15432，Apache-2.0，pypi `sygra` v1.0.0）的派生分支，内含 AgentJudgeBench 复现任务包
- **论文**：arXiv 2608.26623（2026-08-27）
- **数据集**：HF `ServiceNow-AI/AgentJudgeBench`（3,808 records）
- **任务包位置**：`tasks/agentic_bfcl_judge_eval/`（12 文件）+ 依赖 SyGra 骨架（213 个 .py）

## 1.2 语言与工程形态

| 项 | 值（来源） |
|---|---|
| 语言 | Python >=3.9,<3.12,!=3.9.7（pyproject.toml） |
| 包 | `sygra` v1.0.0，hatchling，Apache-2.0（pyproject.toml） |
| 图运行时 | langgraph >=0.6,<1.0（pyproject.toml line 38） |
| 锁文件 | uv.lock（761KB） |
| 规模 | 45MB / 436 文件（含 .git）/ 213 .py |

## 1.3 入口（按复现管道顺序）

1. `prepare_difficulty_inputs.py` — HF JSONL → 3 档输入（input_easy/medium/hard.jsonl）
2. `run_generate.py` — 生成器推理（--model/--difficulty → `<generator>/<difficulty>.jsonl`）
3. `run_judge.py` — judge-only 管道（--generator/--judge_model/--difficulty/--config → `<generator>/judged_by_<judge>/<difficulty>.jsonl`）
4. `analyze_results.py` — 聚合 per-cell mean alignment（→ 论文 Fig 3 / Table 1 / Table 3 / Fig 7）

## 1.4 核心模块地图

```
tasks/agentic_bfcl_judge_eval/
├── task_executor.py        # ★核心：ProgrammaticJudge(Eqs1-3) + ComputeLLMVsProgrammaticAccuracy(Eq4) + pre/post processors
├── run_judge.py            # judge CLI（patch 模型/输入/输出；双分支同图）
├── run_generate.py         # 生成器 CLI
├── prepare_difficulty_inputs.py  # 难度拆分（query/query_medium/query_hard 列）
├── prepare_corrupted_gt.py # C3：同 dag_type 随机 derangement + true_expected_responses 保留
├── analyze_results.py      # condition 前缀推断 + 聚合
└── graph_config*.yaml ×6   # generate_only / main / nogt_05 / corrupted_gt / freeform

sygra/                       # SyGra 骨架（AgentJudgeBench 运行依赖）
├── core/
│   ├── base_task_executor.py    # DefaultTaskExecutor：图装载/执行/续跑/元数据
│   ├── judge_task_executor.py   # JudgeQualityTaskExecutor（占位子类）
│   ├── resumable_execution.py   # DatasetPositionTracker / 续跑
│   └── graph/                   # graph_config / langgraph(StateGraph builder) / nodes / functions
├── config/models.yaml           # 模型注册表（URL/TOKEN 在 .env）
├── config/configuration.yaml    # 默认变换 + model_config.ssl_verify
└── logger / utils / metadata / processors / dataset / ...
```

## 1.5 核心数据结构

- **记录 JSONL 字段**（output_config，graph_config.yaml）：id / dag_type / user_message / generated_tool_calls / judge_response / judge_response_no_gt / programmatic_judge_response / per_metric_comparison(_no_gt) / overall_llm_alignment_percentage(_no_gt)
- **程序化 judge 四指标**：tool_selection / parameter_structure / sequence / query_coverage（∈{0, 0.5, 1}）
- **Alignment（Eq 4）**：per-metric `1 − |programmatic − llm|` → per-record 均值 ×100
- **SygraState 键**：generated_tool_calls / available_tools / expected_responses / corrupted_expected_responses / true_expected_responses / judge_response / judge_response_no_gt / programmatic_judge_response / per_metric_comparison / overall %

## 1.6 生命周期 / 状态

- 图节点链（graph_config.yaml edges）：`START → judge_tool_calls(LLM with-GT) → judge_tool_calls_answer_only(LLM without-GT) → judge_programmatic(lambda) → calculate_judge_accuracy → calculate_answer_only_judge_accuracy → END`
- 一次 `run_judge.py` 调用产出双条件 alignment（with/without GT）
- 续跑：--resume / --checkpoint_interval(500) / --start_index / --batch_size(25)（resumable_execution.py）

## 1.7 配置与凭据

- 模型注册：`sygra/config/models.yaml`（model_type: tgi/vllm/azure_openai/... + parameters）；凭据 `.env` 中 `SYGRA_<MODEL>_URL/TOKEN`
- run_judge.py 显式 `load_dotenv(override=True)` + `os.environ["CURL_CA_BUNDLE"]=""`（禁用 CA 链，本仓库环境设定）
- run 时 --model/--judge_model 覆盖 YAML 默认（占位 `smollm3`）

## 1.8 测试体系

- `tests/`：conftest.py + core/metadata/processors/utils/workflow 子目录
- 未逐文件深读（阶段④以任务包为核心；骨架测试作为辅助证据，盲审计补查）

## 1.9 权限与治理

- 模型凭据走 .env + dotenv；无沙箱/审批（纯批处理研究管道）
- 实验条件隔离靠 **run_name 前缀约定**（c1_/c2_/c3_/freeform_），analyze_results.py 正则推断 —— 治理靠约定而非强制

## 1.10 外部依赖

- langgraph / langchain-core / datasets(HF) / numpy / pandas / PIL / scipy（分析建议）
- HF 数据集下载后本地 JSONL

## 1.11 边界与例外

- 本分支 commit 级快照（无独立版本号）；上游 main 差异未测（depth 1）
- analyze 注明的缺失：Bootstrap CIs（Appendix E）、inter-judge κ（Table 2）需 scipy 自算

## 1.12 宿主项目补充 Observation（盲审计）

- 宿主项目另有 `sygra/core/eval/metrics/` 指标注册表子系统：unit_metrics（exact_match / action_within_bbox / scroll_amount / scroll_direction / typed_value_match + unit_metric_registry）+ aggregator_metrics（accuracy / f1 / pass_at_k / pass_power_k / precision / recall / step_efficiency + aggregator_metric_registry）。
- **AgentJudgeBench 未使用该子系统**（tasks/agentic_bfcl_judge_eval/*.py 无 `from sygra.core.eval` 导入）——ProgrammaticJudge 是 task 内嵌 lambda 路径，与宿主指标注册表并行存在（指标粒度：ABB 是任务级 judge 对齐，宿主是管道级数据质量/unit 指标）。此为语境 Observation，不改变 EK/KO 结论。
- 盲审计另确认 `sygra/core/execution_callbacks.py`（Studio 可视化实时回调）与 LLMNode 契约（output_keys 为 list 时强制 post_process，llm_node.py assert）。

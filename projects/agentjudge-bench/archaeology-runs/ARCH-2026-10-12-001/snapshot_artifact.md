# Snapshot Artifact — ARCH-2026-10-12-001（AgentJudgeBench）

> 浅克隆快照：`archaeology-jobs/ARCH-2026-10-12-001/repo/` ｜ 事实全部可追溯到仓库实际内容。

## 一、Repository Snapshot（快照事实）

| 项 | 值 |
|---|---|
| 项目 | AgentJudgeBench（LLM-as-Judge 可靠性基准复现管道） |
| 宿主仓库 | https://github.com/ServiceNow/SyGra.git |
| 分支 | `scratch/agent_judge_bench`（AgentJudgeBench related files） |
| commit SHA | `97cca4fa07e98ee61bb3a1a2f8639778e299d50a` |
| commit 信息 | "AgentJudgeBench related files" |
| commit 时间 | 2026-05-06 19:54:31 +0530 |
| 克隆方式 | `git clone --depth 1 --branch scratch/agent_judge_bench --single-branch` |
| 快照时间 | 2026-10-11T18:07:11Z（UTC 实测） |
| 体积 | 45MB（含 .git） |
| 文件数 | 436（含 .git；213 个 .py） |
| 关联论文 | arXiv 2608.26623（2026-08-27）"AgentJudgeBench: A Multi-Difficulty Benchmark for Evaluating LLM Judges on Agentic Tool-Calling" |
| 数据集 | HF `ServiceNow-AI/AgentJudgeBench`（3,808 records） |

## 二、项目基础地图

### 语言与打包
- Python `>=3.9,<3.12,!=3.9.7`；包名 `sygra` v1.0.0；hatchling build；Apache-2.0
- 依赖核心：`langgraph>=0.6,<1.0`（计算图底层）、langchain-core（messages）、uv.lock（761KB 全量锁）

### 入口（AgentJudgeBench 复现管道）
- `tasks/agentic_bfcl_judge_eval/run_generate.py`：生成器推理入口（--model/--difficulty → `<generator>/<difficulty>.jsonl`）
- `tasks/agentic_bfcl_judge_eval/run_judge.py`：judge-only 入口（--generator/--judge_model/--difficulty/--config → `<generator>/judged_by_<judge>/<difficulty>.jsonl`）
- `tasks/agentic_bfcl_judge_eval/analyze_results.py`：聚合为 per-cell mean alignment（→ Fig 3/Table 1/Table 3/Fig 7）
- 数据准备：`prepare_difficulty_inputs.py`（HF 列 query/query_medium/query_hard → 3 档 JSONL）；`prepare_corrupted_gt.py`（C3 同 dag_type derangement）

### 核心模块
| 模块 | 职责 |
|---|---|
| `sygra/core/base_task_executor.py` + `DefaultTaskExecutor` | LangGraph 图装载/执行/续跑（resumable_execution.py） |
| `sygra/core/judge_task_executor.py` | `JudgeQualityTaskExecutor`（BaseTaskExecutor 占位子类，log "Using JudgeQualityTaskExecutor"） |
| `sygra/core/graph/` | sygra_state/message/node_processor/edge_condition/lambda_function（图运行时） |
| `tasks/agentic_bfcl_judge_eval/task_executor.py` | **核心知识**：ProgrammaticJudge（Eqs 1-3）、ComputeLLMVsProgrammaticAccuracy（Eq 4）、SerializeFieldsPreProcessor、safe_parse_json、post-processors |
| `tasks/agentic_bfcl_judge_eval/graph_config*.yaml`（6 个） | 图定义：generate_only / main（with+without GT）/ nogt_05（C2）/ corrupted_gt（C3）/ freeform（RQ6） |

### 核心数据结构
- **记录 JSONL 字段**：id / dag_type（6 拓扑：linear·fan-out·fan-in·diamond·optional-enrichment·loop-like）/ user_message / generated_tool_calls / judge_response / judge_response_no_gt / programmatic_judge_response / per_metric_comparison（_no_gt）/ overall_llm_alignment_percentage（_no_gt）
- **程序化 judge 四指标**：tool_selection_accuracy / parameter_structure_accuracy / sequence_accuracy / query_coverage_accuracy（各 ∈ {0, 0.5, 1}，结构化）
- **Alignment（Eq 4）**：per-metric `1 − |programmatic − llm|` → per-record 平均 ×100 → per-cell mean

### 状态与生命周期
- 图节点链：`START → judge_tool_calls（LLM with-GT）→ judge_tool_calls_answer_only（LLM without-GT）→ judge_programmatic（lambda）→ calculate_judge_accuracy → calculate_answer_only_judge_accuracy → END`
- 一次 run_judge.py 调用产出双条件 alignment（with/without GT）
- 续跑：--resume / --checkpoint_interval（500）/ --start_index / --batch_size（25）

### 测试体系
- `tests/`：conftest.py + core/metadata/processors/utils/workflow 子目录（未逐文件深读，阶段④覆盖）

### 配置与模型
- 模型注册：`sygra/config/models.yaml`（每个模型需 .env 中 `SYGRA_<MODEL>_URL/TOKEN`）
- run 时 --model/--judge_model 覆盖 YAML 默认（占位 `smollm3`）
- 6 档难度 × 5 生成器 × 6 judge 网格

### 权限与治理机制
- 模型凭据走 `.env` + dotenv 加载（run_judge.py 显式 `load_dotenv(override=True)`）；`os.environ["CURL_CA_BUNDLE"]=""` 禁用 CA 链（本仓库环境设定）
- 无沙箱/审批机制（纯批处理研究管道，非安全敏感运行时）

### 外部依赖
- langgraph / langchain-core / scipy（分析建议）/ HF 数据集（下载后本地 JSONL）
- 实验规模：3,808 × 3 × 5 × 6 × 2 = **321,648 evaluations**

## 三、与论文的映射（证据锚）

- Eqs 1–3 → `ProgrammaticJudge`（task_executor.py，逐指标注释）
- Eq 4 → `ComputeLLMVsProgrammaticAccuracy*`（两个 lambda：with-GT / answer-only）
- C2 ablation（Table 3）→ `graph_config_nogt_05.yaml`（without-GT 0.5-default prompt）
- C3 control（Table 1 cGT）→ `prepare_corrupted_gt.py` + `graph_config_corrupted_gt.yaml`（judge 见 corrupted GT；程序化 judge 仍用 `true_expected_responses` 真参照 → over-reliance/anchoring 测量）
- RQ6（Fig 7）→ `graph_config_freeform.yaml`（minimal free-form prompt）
- RQ4/5/7/H1（temperature/scale/CoT/small judge）→ 复用 main 图 + models.yaml 变体

## 四、边界与例外

- **宿主仓库语义**：本分支是 SyGra 主项目（synthetic data pipeline，arXiv 2508.15432）的派生分支；AgentJudgeBench 是 tasks/ 下的一个复现任务包。考古对象 = 以 agentic_bfcl_judge_eval 为核心 + 其依赖的 sygra 骨架（图运行时/执行器/logger/utils）
- **本分支无独立版本号**：commit 级快照；上游 SyGra main 分支与本分支的差异未测（分支派生语义，盲审计可查 git 历史——depth 1 限制）
- 分析脚本注明的缺失：Bootstrap CIs（Appendix E）与 inter-judge κ（Table 2）需 scipy 自行计算，仓库不含

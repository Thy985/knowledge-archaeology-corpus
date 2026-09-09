# 01 · Project Layer — CLEAR（项目地图）

> 全部事实可回溯到仓库实际内容（commit 9a5367bd048ed656b62093cc09d77b872fca4ff8，main，2026-09-10 克隆）。

## 架构

```
                    ┌─────────────────────────────────────────────────────┐
                    │  clear_eval（src/clear_eval/，v2.0.5，Apache-2.0） │
                    └─────────────────────────────────────────────────────┘
   LLM Analysis 模式                                  Agentic Analysis 模式
   （标准 LLM 输出评估）                                （多 agent 轨迹评估）
   ┌──────────────────────┐                    ┌──────────────────────────────────────────┐
   │ pipeline/            │                    │ agentic/pipeline/                        │
   │  config_loader       │                    │  preprocess_traces/  ── 原始 trace → IR CSV│
   │  evaluation_criteria │                    │   process_mlflow_traces / langfuse        │
   │  inference_utils/    │                    │   compact_trace_formatter                 │
   │   llm_client         │                    │  full_traces_evaluation/                  │
   │   endpoint_backends  │                    │   trace_evaluation/                       │
   │   langchain_chat     │                    │    task_success_evaluator (0/1+根因)      │
   │  use_cases/          │                    │    full_trajectory_evaluator (14 维)      │
   │   eval / external    │                    │    rubric_generator / rubric_evaluator    │
   │   judge / tool_call  │                    │   clear_analysis/                         │
   │  full_pipeline       │                    │    base_clear_runner (ABC)                │
   │  caching_utils       │                    │    issues_clear_runner                    │
   │  threading_utils     │                    │    root_cause_clear_runner                │
   │  propmts[原文拼写]   │                    │    run_clear_analysis                     │
   └──────────────────────┘                    │  run_clear_agentic_eval / step_analysis  │
                                               └──────────────────────────────────────────┘
   CLI（cli.py：generation/evaluation/aggregation/dashboard/agentic-eval/agentic-dashboard）
   Dashboard：LLM 模式 = Streamlit（load_ui.py）；agentic = NiceGUI（agentic/dashboard/，path_analysis）
```

## 核心模块

| 模块 | 职责 | 关键文件 |
|------|------|---------|
| pipeline/（LLM 模式） | 生成→评估→聚合管线 | full_pipeline.py、evaluation_criteria.py、external_judge.py |
| inference_utils/ | 多后端 LLM 客户端（langchain/litellm/endpoint HTTP） | llm_client.py、endpoint_backends.py |
| use_cases/ | 评估用例变体（eval/external_judge/tool_call） | use_cases/ |
| agentic/pipeline/preprocess_traces | MLflow/Langfuse trace → 统一 IR CSV → compact | process_mlflow_traces.py、compact_trace_formatter.py |
| agentic/pipeline/trace_evaluation | 三步评估：task_success / full_trajectory / rubric | task_success_evaluator.py 等 |
| agentic/pipeline/clear_analysis | 评估结果再聚合（问题/根因 CLEAR 分析） | base_clear_runner.py、root_cause_clear_runner.py |
| agentic/dashboard | NiceGUI 可视化 + 预测性路径模式 | path_analysis.py |
| analysis_runner / cli / args | 统一入口 | analysis_runner.py、cli.py |

## 生命周期

```
配置（default_config.yaml ← 用户 yaml/json 递归 merge）→ 数据加载（CSV/parquet/JSON）
→ [LLM 模式] generation → evaluation（每记录 打分+critique）→ aggregation（issue 聚类）
→ [agentic 模式] preprocess（trace→IR）→ compact → task_success/full_trajectory/rubric
→ clear_analysis（issues/root_cause 聚合）→ dashboard（可视化 + 路径模式）
全程：断点续跑（checkpoint）、缓存、并发 max_workers
```

## 核心数据结构

- **IR CSV（中间表示）**：Name / task_id / step_in_trace_general / llm_call_index / model_input / response（必需）+ intent / tool_or_agent / api_spec / meta_data / traj_score（可选）——每行一次 LLM 调用（docs/agentic/intermediate-representation.md）
- **EvaluationCriterion/Criteria**：{name: description} 字典驱动的标准集（evaluation_criteria.py:15-74）
- **评估输出**：EVALUATION_TEXT_COL / SCORE_COL（constants.py）；task_success = success + consideration + failure_root_cause（仅失败轨）

## 状态

- 无全局可变状态（对比 OpenCode WorkingDirectory panic）；评估过程状态通过 checkpoint 文件 + 缓存目录持久化（caching_utils.py：load/save dataframe/json）

## 测试体系

- 11 个测试文件 / 2758 行：cli/test_argparse（906）、agentic/test_trace_utils（693）、pipeline/test_resume_pipeline（420）、test_issues_format、test_inference_backends、test_temperature_validation、test_build_json_recommendations、test_static_dashboard_recs
- **本 run 实测：211 passed + 3 skipped（10.27s，Python 3.12）**——S4 级
- test_temperature_validation 需要真实 API key（OPENAI_API_KEY/WATSONX_APIKEY），3 skipped 即此类

## 配置

- 默认配置 src/clear_eval/pipeline/setup/default_config.yaml：inference_backend（langchain/litellm/endpoint）、provider（watsonx 默认/openai/rits）、gen/eval 模型名、max_workers、eval_model_params.max_tokens=8096、external_judge_path/function/config
- config_loader.py：JSON/YAML + 递归 merge（overrides 覆盖 defaults）

## 权限 / policy / governance

- 无沙箱/权限模型（它是评估工具，非运行时）
- 治理机制 = 评估纪律本身：judge 输入纯度约束（full_trajectory_prompts.py 系统消息）、统计显著性门槛（path_analysis）、overwrite 标志（base_clear_runner）

## 外部依赖

- langchain / langgraph / langchain_openai / langchain_ibm / langchain_community
- ibm_watsonx_ai（IBM 生态，默认 provider）
- pandas / numpy / pyarrow（数据）
- streamlit（LLM 模式 dashboard）/ nicegui（agentic dashboard，README 声明）
- openai（OpenAI 后端）、python-dotenv、pyyaml、tqdm、matplotlib、seaborn

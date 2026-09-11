# Repository Snapshot — CLEAR（ARCH-2026-09-10-001）

```yaml
repository: https://github.com/IBM/CLEAR
commit_sha: 9a5367bd048ed656b62093cc09d77b872fca4ff8
branch: main
tag: (无 release tag；pyproject version = 2.0.5，PyPI clear-eval)
repository_version: clear_eval v2.0.5
archived: false（GitHub API archived: False）
stars: 58（GitHub API 2026-09-10 实测）
pushed_at: 2026-07-27T12:40:41Z（活跃）
created_at: 2025-07-07T09:30:55Z
license: Apache-2.0
language: Python（>=3.10）
analysis_timestamp: 2026-09-10T02:00:00+08:00
knowledge_archaeology_skill_version: v3.2
```

## 项目基础地图

```
CLEAR/
├── README.md                    # 双模式（LLM Analysis / Agentic Analysis）+ 安装 + 快速开始
├── pyproject.toml               # clear_eval v2.0.5；deps: langchain/langgraph/langchain_openai/langchain_ibm/ibm_watsonx_ai/openai
├── requirements.txt             # 同 pyproject（+streamlit/matplotlib/seaborn/pyarrow）
├── src/clear_eval/
│   ├── cli.py / args.py / analysis_runner.py / load_ui.py / logging_config.py
│   ├── pipeline/                # LLM 模式管线
│   │   ├── full_pipeline.py     # generation→evaluation→aggregation 编排（含 resume）
│   │   ├── evaluation_criteria.py   # 默认 3 维 LLM / 4 维 agentic + 自定义
│   │   ├── external_judge.py    # 外部 judge 动态加载（importlib）
│   │   ├── config_loader.py     # JSON/YAML + 递归 merge
│   │   ├── caching_utils.py / threading_utils.py
│   │   ├── inference_utils/     # llm_client.py（温度探测）+ endpoint_backends.py（HTTP/SSE/WatsonX token）
│   │   ├── use_cases/           # eval / external_judge / tool_call
│   │   └── setup/default_config.yaml
│   ├── agentic/
│   │   ├── README.md            # agentic 工作流文档
│   │   └── pipeline/
│   │       ├── preprocess_traces/    # process_mlflow_traces / process_langfuse_traces / compact_trace_formatter
│   │       ├── full_traces_evaluation/
│   │       │   ├── trace_evaluation/    # task_success / full_trajectory / rubric_generator / rubric_evaluator
│   │       │   └── clear_analysis/      # base_clear_runner / issues / root_cause / run_clear_analysis
│   │       └── run_clear_agentic_eval.py / run_clear_step_analysis.py / build_json_results.py
│   │   └── dashboard/           # NiceGUI（launch/generate_static）+ path_analysis.py（统计模式挖掘）
│   ├── sample_data/gsm8k/
├── docs/                        # PROVIDERS / llm-analysis / agentic/{dashboard,intermediate-representation,mlflow-tracing}
├── examples/                    # custom_judges（exact_match/unitxt）+ run_agent（langfuse/mlflow/no-tracing）
└── tests/                       # 8 测试文件 2758 行
```

## 识别项

| 项 | 值 | 证据 |
|----|-----|------|
| 主要语言 | Python 3.10+ | pyproject requires-python |
| 主要运行入口 | CLI（cli.py：run_clear_eval_evaluation / run_clear_agentic_eval）+ Python API（analysis_runner） | cli.py / analysis_runner.py |
| 核心模块 | pipeline/（LLM 模式）、agentic/pipeline/（agentic 模式）、inference_utils/（多后端）、path_analysis（统计） | src 结构 |
| 核心数据结构 | IR CSV（Name/task_id/step/llm_call_index/model_input/response）；EvaluationCriteria{name:desc}；SCORE_COL/EVALUATION_TEXT_COL | docs/agentic/intermediate-representation.md；evaluation_criteria.py；constants.py |
| 核心状态 | 无全局可变状态；进度=文件系统（checkpoint/缓存/结果落盘） | caching_utils.py；base_clear_runner.py |
| 主要测试体系 | 8 文件 2758 行；**本 run 实测 211 passed + 3 skipped（10.27s）** | pytest 实测 |
| 主要配置 | default_config.yaml（backend/provider/model/max_workers/max_tokens）+ 用户 yaml/json 递归覆盖 | config_loader.py |
| 权限/policy/governance | 无执行沙箱；治理=评估纪律（judge 纯度约束 + 统计门槛 + overwrite 控制） | full_trajectory_prompts.py；path_analysis.py |
| 主要外部依赖 | langchain 全家 / langgraph / ibm_watsonx_ai / openai / pandas / numpy / pyarrow / streamlit / nicegui | pyproject + requirements |

## 快照说明

- 浅克隆（--depth 1）HEAD 9a5367b，main 分支（2026-09-10）。
- 规模：69 src py 文件 / ~20k 行（src+tests 实测 wc）。
- 测试运行：`pip install -e .` 成功（Python 3.12.11）→ `python3 -m pytest tests/ -q` → **211 passed + 3 skipped in 10.27s**。
- 本快照事实全部可回溯到仓库实际内容（文件路径/行号见各层引用）。

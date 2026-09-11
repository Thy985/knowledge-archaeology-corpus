# 04 · Flow Atlas — CLEAR（七类流）

> 全部 Edge 从真实代码导出，可回溯到 symbol/file/condition。

## 04.1 Control Flow（控制流）

```
CLI 入口（cli.py main）→ run_clear_agentic_eval / run_clear_eval_evaluation
→ [LLM 模式] full_pipeline.run_eval_pipeline（full_pipeline.py:263）
     ├─ run_generation_pipeline（:49）→ run_evaluation_from_df（:168）→ run_aggregation_from_df（:239）
     └─ resolve_issues_list（:177，resume_enabled 时检查 checkpoint）
→ [agentic 模式] run_trajectory_evaluation_pipeline（run_trajectory_evaluation_pipeline.py:526）
     ├─ preprocess_traces_if_needed（:447，--from-raw-traces 条件）
     ├─ resolve_eval_types（:179）→ task_success（:239）/ full_trajectory（:271）/ rubric_generation（:305）→ rubric_evaluation（:346）
     ├─ resolve_clear_analysis_types（:195）→ issues / root_cause（:380）
     └─ create_ui_input（build_json_results）→ dashboard（launch_dashboard）
```

## 04.2 State Flow（状态流）

```
（无进程内全局可变状态——对比 OpenCode WorkingDirectory panic）
评估进度状态 = 文件系统：
  checkpoint 文件（resolve_issues_list 读取，resume_enabled 分支，full_pipeline.py:177-212）
  缓存目录（caching_utils.save/load_dataframe_to_cache，:63）
  结果落盘（base_clear_runner.extract_records_from_file / discover_result_files，:123-144）
overwrite 标志控制是否覆盖已有 CLEAR 结果（base_clear_runner.__init__）
```

## 04.3 Data Flow（数据流）

```
原始 trace（JSON，LangGraph×MLflow / LangGraph×Langfuse / CrewAI×Langfuse）
  → process_mlflow_traces / process_langfuse_traces（preprocess_traces/，provider 适配器）
  → IR CSV（统一中间表示：Name/task_id/step_in_trace_general/llm_call_index/model_input/response）
  → compact_trace_formatter（降 token：去工具定义、api_spec 只留工具名、meta_data 提关键指标）
  → evaluators（task_success / full_trajectory / rubric）
  → 评估结果文件（success/consideration/failure_root_cause；14 维分数+反馈；rubric 文件）
  → clear_analysis（base_clear_runner.extract_all_records → issues / root_cause 聚合）
  → build_json_results → dashboard（NiceGUI）+ path_analysis（统计模式）
LLM 模式：CSV → evaluation（SCORE_COL/EVALUATION_TEXT_COL）→ aggregation（issues_format: shortcomings|recommendations）
```

## 04.4 Evidence Flow（证据流）

```
评估证据 = judge 的输入输出对：
  model_input（messages）+ response（LLM 输出）→ 评估器 prompt 构建
    （task_success_evaluator.build_prompt :88 / full_trajectory_prompts.build_default_prompt）
  judge 系统消息约束：grounded solely in trajectory content（full_trajectory_prompts.py SYSTEM_MESSAGE_FULL_TRAJ）
  输出：分数 + 文本 critique 绑定（constants SCORE_COL/EVALUATION_TEXT_COL）
  failure_root_cause 仅 success=0 时产出（task_success_evaluator :118-133）→ root_cause 聚合的唯一输入
  证据强度分级：traj_score（ground-truth 0-1）作为可选真值列存在（IR 文档）
```

## 04.5 Authority Flow（权威流）

```
（CLEAR 是评估工具，无执行沙箱/权限模型）
权威面 = judge 的角色边界：
  judge 权威被显式限制为"只读轨迹内容做判断"（EK-11 系统消息）
  外部 judge 插件（external_judge.load_external_judge）以函数调用方式注入，无子进程/沙箱
  覆盖权：用户配置递归覆盖 defaults（merge_configs，config_loader.py:63）
  输出方向：评估结果只写本地文件系统（无网络回写/上报）
```

## 04.6 Memory Flow（记忆流）

```
评测运行记忆 = 持久化中间产物：
  缓存（caching_utils：dataframe/json + expected_rows 校验）
  checkpoint（resume_enabled → 跳过已完成记录，test_resume_pipeline 420 行测试）
  run_name 进文件名（get_run_name/full_pipeline.py:34-43）
  agentic 结果按 evaluation_type 目录组织（base_clear_runner.get_clear_output_dir :171）
  （无跨运行长期记忆——每次评测独立）
```

## 04.7 Policy Flow（策略流）

```
评测策略配置：
  default_config.yaml（inference_backend/provider/gen+eval 模型/max_workers/max_tokens=8096）
    → merge_configs 用户覆盖 → resolve_provider_config（config_loader）
  evaluation_criteria（默认 3 维 LLM / 4 维 agentic，--evaluation_criteria 自定义注入）
  issues_format 策略：shortcomings（默认，向后兼容）vs recommendations（actionable，映射逻辑反转）
  --separate-tools / --from-raw-traces / --overwrite / resume_enabled 运行策略开关
  judge 策略：temperature=0 确定性注入 → 探测失败回退 disable_temperature（llm_client）
```

## Edge 可回溯性声明

| Flow | 关键 Edge | 回溯 |
|------|----------|------|
| Control | agentic 管线入口 | run_trajectory_evaluation_pipeline.py:526 `def run_trajectory_evaluation_pipeline` |
| Control | resume 分支 | full_pipeline.py:177 `resolve_issues_list(..., resume_enabled, ...)` |
| State | 无全局可变状态 | （仓库无模块级 mutable 单例；对比 opencode config.go:874 panic） |
| Data | trace→IR | preprocess_traces/process_mlflow_traces.py |
| Data | IR→compact | compact_trace_formatter.py（模块 docstring） |
| Evidence | 根因仅失败轨 | task_success_evaluator.py:118-133 `only required when success=0` |
| Authority | judge 纯度约束 | full_trajectory_prompts.py SYSTEM_MESSAGE_FULL_TRAJ |
| Policy | 配置覆盖 | config_loader.py:63 `def merge_configs` |
| Policy | 温度回退 | llm_client.py:97 `def disable_temperature` |

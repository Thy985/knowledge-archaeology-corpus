# 04 Flow Atlas — AgentJudgeBench（ARCH-2026-10-12-001）

> 七类流，从真实代码导出。关键 Edge 可回溯 symbol/file/condition/state transition。

---

## 4.1 Control Flow（控制流）

```
run_judge.py --generator/--judge_model/--difficulty/--config
  → load graph_config*.yaml (utils.load_yaml_file)
  → patch: data_config.source.file_path (DIR/<generator>/<difficulty>.jsonl)
           node judge_tool_calls.model.name = --judge_model
           node judge_tool_calls_answer_only.model.name = --judge_model
  → DefaultTaskExecutor(args, graph_config_dict=config).execute()
     → LangGraphBuilder.build(): _update_state_variables() → add_nodes → add_edges
     → 图执行节点链（graph_config.yaml edges）:
       START → judge_tool_calls → judge_tool_calls_answer_only
             → judge_programmatic → calculate_judge_accuracy
             → calculate_answer_only_judge_accuracy → END
  → 写 <generator>/judged_by_<judge>/<difficulty>.jsonl
```
**关键 Edge 证据**：`run_judge.py:51-54`（patch 双节点 model）、`graph_config.yaml`（edges 列表）、`base_task_executor.py`（`DefaultTaskExecutor.execute()`）、`graph_builder.py:build()`（add_nodes/add_edges）。

**变体分支（config 切换）**：
- `--config graph_config_nogt_05.yaml` → without-GT 节点提示改为 0.5-default（C2）
- `--config graph_config_corrupted_gt.yaml` + `--input ..._corrupted_gt.jsonl` → with-GT 节点收到 corrupted_expected_responses（C3）
- `--config graph_config_freeform.yaml` → 双 judge 节点提示替换为自由形式（RQ6）

## 4.2 State Flow（状态流）

```
SygraState 键演化（task_executor.py 各 processor 写入）：
  input: id / dag_type / user_message / generated_tool_calls / available_tools
         expected_responses | corrupted_expected_responses (+true_expected_responses)
  SerializeFieldsPreProcessor: 列表/字典字段 → json.dumps(indent=2) 字符串
  judge_tool_calls (LLM) → judge_response  [JudgeToolCallsPostProcessor: safe_parse_json(content)]
  judge_tool_calls_answer_only (LLM) → judge_response_no_gt [JudgeToolCallsAnswerOnlyPostProcessor]
  judge_programmatic (lambda) → programmatic_judge_response
  calculate_judge_accuracy → per_metric_comparison / overall_llm_alignment_percentage
  calculate_answer_only_judge_accuracy → per_metric_comparison_no_gt / overall_llm_alignment_percentage_no_gt
output_config.output_map: 从 state 取键写出 JSONL
```
**关键 Edge**：`graph_config.yaml` 的 `output_keys` 声明 + `output_config.output_map`；`task_executor.py` 各 class 的 `state[key] = ...`。

## 4.3 Data Flow（数据流）

```
HF ServiceNow-AI/AgentJudgeBench（3,808 records）
  → prepare_difficulty_inputs.py → input_{easy,medium,hard}.jsonl（user_message 按列替换，KEEP_FIELDS 保留）
  → run_generate.py → <generator>/{easy,medium,hard}.jsonl（generated_tool_calls 生成）
  → prepare_corrupted_gt.py（C3 可选）→ <generator>/<diff>_corrupted_gt.jsonl
  → run_judge.py → <generator>/judged_by_<judge>/<run_name_prefix>output_<ts>.jsonl
       （3 层单文件：run_name+ts 在文件名，base_task_executor.py:1037-1045）
  → analyze_results.py → results.csv（per-cell mean: generator×judge×difficulty×condition）
       ⚠ 缺陷：analyze glob `*/judged_by_*/*/*.jsonl` 期望 4 层目录（run_name 作目录名），
         与实际 3 层输出不兼容 → 默认流程匹配不到文件，需手动整理或修 glob（见 C-07）
```
**关键 Edge**：`tasks/agentic_bfcl_judge_eval/prepare_difficulty_inputs.py`（KEEP_FIELDS 白名单）、`tasks/agentic_bfcl_judge_eval/analyze_results.py`（glob 4 层 vs 实际 3 层——不匹配边，独立审计确认）。

## 4.4 Evidence Flow（证据流）

```
程序化 judge（EK-01 四维符号打分）= ground truth（最高权威证据）
  ↓
LLM judge 输出（judge_response / _no_gt）
  ↓
对齐证据：per-metric 1-|Δ|（EK-02）→ overall_llm_alignment_percentage(_no_gt)（每条记录的证据）
  ↓
聚合证据：analyze_results.py per-cell mean（→ 论文图/表）
证据链完整性：程序化参照始终独立于 LLM 输出（含 C3：corrupted GT 只污染 LLM 输入，true GT 供参照）
```
**关键 Edge**：`ComputeLLMVsProgrammaticAccuracy.apply`（以 `state["programmatic_judge_response"]` 为真值源）；`tasks/agentic_bfcl_judge_eval/prepare_corrupted_gt.py`（`true_expected_responses` 保留语义）。

## 4.5 Authority Flow（权威流）

```
模型凭据权威：.env SYGRA_<MODEL>_URL/TOKEN ← models.yaml 模型注册表 ← run_judge.py load_dotenv
LLM judge 输出权威：response.message.content → safe_parse_json → state（无人工复核）
程序化 judge 权威：LambdaFunction 直接写 state（无校验）
配置权威：graph YAML（节点/边/提示）→ GraphConfig（运行时单源）
约定权威：run_name 前缀（c1_/c2_/c3_/freeform_）→ analyze 条件判定（无元数据文件兜底）
```
**关键 Edge**：`run_judge.py:18-19`（load_dotenv override）、`task_executor.py`（post-processor 直接信任 LLM JSON）、`tasks/agentic_bfcl_judge_eval/analyze_results.py`（CONDITION_PATTERNS 正则）。**例外**：`os.environ["CURL_CA_BUNDLE"]=""`（禁用 CA 校验，对模型服务 TLS 信任放开——本仓库环境设定）。

## 4.6 Memory Flow（记忆流）

- **无跨运行记忆**：批处理管道，每条记录独立；无状态缓存/经验回放。
- **运行内状态**：SygraState（见 4.2）即运行内存；`resumable_execution.py` 的 `DatasetPositionTracker`（InMemoryPositionTracker）跟踪数据集位置用于 --resume 续跑（位置记忆，非知识记忆）。
- **输出即记忆**：JSONL 是唯一的持久化载体（condition 靠目录/run_name 区分）。
**关键 Edge**：`resumable_execution.py`（advance_to_position/mark_position）、`base_task_executor.py`（checkpoint_interval=500）。

## 4.7 Policy Flow（策略流）

```
评测策略定义（graph YAML prompt 内嵌）→ 策略分档（1.0-default vs 0.5-default）
  → judge 节点执行（LLM 按提示评分）
  → 对齐计算（策略效果度量）
  → 变体对比（C2/C3/RQ6 → 论文结论）
治理闭环：策略变化以 config 文件形式版本化（6 个 graph_config*.yaml）；无运行期动态策略
```
**关键 Edge**：`graph_config.yaml` prompt 全文（"Assume the programmatic judge is the ground truth" / "Scores should default to 1.0 unless..."）；`graph_config_nogt_05.yaml` 两处 diff（155/228 行，"default to 0.5 when uncertain"）。

---

## Flow 交叉校验（Flow→KO）

- KO-01 ← EK-01/02/05 ← Evidence Flow（程序化锚）✓
- KO-02 ← EK-07/08/10/11 ← Policy Flow（提示校准）✓
- KO-03 ← EK-03/04 ← Control/Data Flow（answer-only 分支）✓
- KO-04 ← EK-05 ← Evidence Flow（探针隔离）✓
- KO-05 ← EK-12/13/14/15 ← Authority Flow（分层与约定）✓

## 七类流覆盖检查

| 流 | 覆盖 | 关键 Edge 可回溯 |
|---|---|---|
| Control | ✅ 图节点链 + config 变体分支 | run_judge.py:51-54 / graph_config.yaml edges |
| State | ✅ SygraState 键演化 | output_keys / output_map / task_executor state 写入 |
| Data | ⚠️ HF→JSONL→CSV 全链（analyze glob 4 层 vs 实际 3 层不匹配，见 C-07） | prepare_*/analyze 路径协议 |
| Evidence | ✅ 程序化锚 + 对齐链 | ComputeLLMVsProgrammaticAccuracy / prepare_corrupted_gt |
| Authority | ✅ 凭据/输出/配置/约定 | load_dotenv / CONDITION_PATTERNS |
| Memory | ✅ 无跨运行记忆 + 位置续跑 | resumable_execution.py |
| Policy | ✅ 提示内嵌策略 + config 版本化 | graph_config*.yaml diff |

# 02 Engineering Knowledge — AgentJudgeBench（ARCH-2026-10-12-001）

> EK Graph（v3.1）：每条 EK 声明 links（六类边）。宽底座，L1/L2。共 15 条。
> 证据引用格式：`<file>` 相对 repo 根。全部可回溯。

---

## EK-01 程序化 judge：四维符号打分（Eqs 1-3）
**机制**：`ProgrammaticJudge`（`tasks/agentic_bfcl_judge_eval/task_executor.py`）对生成的 tool calls 计算四个 0-1 指标：
- tool_selection：`1 − (extra+missing)/max(len(exp_set),1)`（set 层面，工具名精确匹配）
- parameter_structure：期望参数 key 集合匹配；unknown tool→0；仅多参数→0.5；缺参数→0
- sequence：位置精确匹配计数/len(exp)
- query_coverage：`len(intersection)/len(exp_set)`
注释明示："strictly symbolic and deterministic: No semantic reasoning, No value correctness checks"。
**links**: mechanism→EK-02, causal→EK-02, subsystem→EK-09, contrast→EK-07

## EK-02 对齐度公式（Eq 4）：`1 − |programmatic − llm|`
**机制**：`ComputeLLMVsProgrammaticAccuracy*`（task_executor.py，两个 lambda：with-GT 与 answer-only 各一）逐指标算绝对差，match_score=1-error，整体 = 均值×100（`overall_llm_alignment_percentage` / `_no_gt`）。程序化 judge 被视作 ground truth。
**links**: causal←EK-01, dependency→EK-01, mechanism→EK-08

## EK-03 双分支同图：with-GT / without-GT 一次跑
**机制**：`graph_config.yaml` 图含两个 LLM judge 节点（`judge_tool_calls` 带 expected_responses 提示；`judge_tool_calls_answer_only` 仅 query+tools+生成），串行执行，一次 run_judge.py 产出双 alignment。**对比实验的最小运行单元设计**。
**links**: subsystem→EK-04, causal→EK-02, contrast→EK-10, contrast→EK-11

## EK-04 answer-only 分支：无 GT 提示降级策略
**机制**：`judge_tool_calls_answer_only` 提示把 AVAILABLE TOOLS 声明为唯一真源，删除所有 expected 引用；评分规则保留四维但措辞转向"plausibly helps"（宽容倾向）。测量"无 ground truth 时 judge 自我评测的可靠性"。
**links**: subsystem→EK-03, mechanism→EK-01, contrast→EK-07

## EK-05 C3 corrupted-GT：同 dag_type 内随机 derangement
**机制**：`tasks/agentic_bfcl_judge_eval/prepare_corrupted_gt.py` 按 dag_type 分组做随机排列（无自映射；singleton 组自交换并在 stdout 报告），把另一记录的 expected_responses 作为 `corrupted_expected_responses` 喂给 judge；**原 `expected_responses` 字段保留真值**（程序化 judge 读 `state["expected_responses"]`），`true_expected_responses` 为备查冗余字段。**测 judge 对 GT 的 over-reliance（anchoring）**。
**links**: causal→EK-02, mechanism→EK-01, contrast→EK-03

## EK-06 难度三档：query 列拆分
**机制**：`tasks/agentic_bfcl_judge_eval/prepare_difficulty_inputs.py` 把 HF 记录的 user_message/query_medium/query_hard 三列拆成 input_easy/medium/hard.jsonl；同 record 其余字段（available_tools/expected_responses/dag_type/id）不变。难度是**提示层**操作，非任务结构变化。
**links**: causal→EK-01, subsystem→EK-03, constraint→EK-12

## EK-07 结构化评分 prompt：1.0/0.5/0.0 + 保守默认
**机制**：graph_config.yaml 的 with-GT prompt 强制四维 JSON 输出、明确"Assume the programmatic judge is the ground truth"、"Judge structure, not semantics"、"Scores should default to 1.0 unless a clear structural violation exists"。**用提示词工程把 LLM 输出校准到程序化 judge 口径**。
**links**: contrast→EK-01, causal→EK-02, mechanism→EK-08

## EK-08 safe_parse_json：think 标签 + markdown fence 剥离
**机制**：`safe_parse_json`（task_executor.py）先 strip `</think>` 之后内容，再剥 ```json fences，最后 json.loads。**模型输出容错层**，与 C2/freeform 各变体共用。
**links**: dependency→EK-02, dependency→EK-07, subsystem→EK-09

## EK-09 SerializeFieldsPreProcessor：状态字段 JSON 序列化
**机制**：`SerializeFieldsPreProcessor` 把 state 中 generated_tool_calls/available_tools/expected_responses/corrupted_expected_responses 非 str 值 json.dumps(indent=2) 注入提示模板。**图状态与提示模板间的适配层**。
**links**: subsystem→EK-01, subsystem→EK-03, causal→EK-07

## EK-10 C2 校准变体：without-GT 0.5-default
**机制**：`graph_config_nogt_05.yaml` 仅改 prompt 两处（"default to 1.0"→"default 0.5 when uncertain; score 1.0 only when evidence supports"）。**对齐度对标度（default 值）敏感性的 ablation**。
**links**: contrast→EK-03, contrast→EK-07, causal→EK-02

## EK-11 RQ6 freeform 变体：最小自由形式 prompt
**机制**：`graph_config_freeform.yaml` 用最小 prompt（"return a JSON object with four accuracy scores"）替代结构化 rubric。**测提示结构化程度对对齐的影响**（论文 Fig 7）。
**links**: contrast→EK-03, contrast→EK-07, causal→EK-02

## EK-12 SyGra 图运行时：LangGraph StateGraph + 节点处理器
**机制**：`sygra/core/graph/langgraph/graph_builder.py` 从 YAML 动态构建 StateGraph（add_nodes/add_edges，START/END 映射）；节点类型 llm/lambda，pre/post processor 通过字符串路径从 `get_func_from_str` 解析加载（`sygra/utils/utils.py`）。`sygra/core/base_task_executor.py` 的 `DefaultTaskExecutor` 负责数据集装载/图构建/元数据收集/续跑。
**契约（盲审计补充）**：`LLMNode`（llm_node.py）`output_keys` 为 list 时 assert 强制 `post_process`——ABB 的 JudgeToolCallsPostProcessor 存在是图运行时强制契约。宿主项目另有 `sygra/core/eval/metrics` 指标注册表（unit/aggregator），**ABB 未使用**（task 内嵌 lambda 路径，见 01 层 Observation）。
**links**: subsystem→EK-13, dependency→EK-09, constraint→EK-06, mechanism→EK-14, constraint→EK-15

## EK-13 run_judge/run_generate：参数 patch 模式
**机制**：CLI 先 load YAML → patch data_config.source.file_path、node model name（judge 双节点都 patch）、output_dir → DefaultTaskExecutor 执行。**一次 CLI = 一次图实例化**；run_name 默认 `judge_<gen>_<judge>_<diff>` 或 `generate_<model>_<diff>`。
**links**: subsystem→EK-12, causal→EK-03, constraint→EK-15

## EK-14 GPT-5 温度约束 hack
**机制**：`tasks/agentic_bfcl_judge_eval/run_generate.py` 若 model key 含 `gpt_5`/`gpt-5` 则 pop 掉 temperature 参数（"GPT-5 family only supports temperature=1"）。**模型家族能力差异导致参数层特判**。
**links**: contrast→EK-13, constraint→EK-12

## EK-15 analyze 条件推断：run_name 前缀约定（含管道缺陷）
**机制**：`tasks/agentic_bfcl_judge_eval/analyze_results.py` 按 run_dir 正则推断 condition（`^(c1_|judge_)`→c1_main、`^(c2_|nogt_?05)`→c2_nogt_05、`^(c3_|corrupt)`→c3_corrupted_gt、`^freeform`→rq6_freeform）；聚合 per-cell mean（with/without GT 分开）。**实验条件隔离靠目录命名约定，无元数据文件**。
**已知缺陷（盲审计确认）**：analyze glob `*/judged_by_*/*/*.jsonl` 要求 4 层目录（run_name 作目录名），而 BaseTaskExecutor 实际输出为 3 层单文件 `judged_by_<judge>/<run_name_prefix>output_<ts>.jsonl`（run_name 在文件名，base_task_executor.py:1037-1045）→ **默认复现流程下 analyze 匹配不到输出文件**，需手动整理或修 glob（见 C-07）。
**links**: dependency→EK-13, causal→EK-02, constraint→EK-12, contrast→EK-12

---

## EK Graph 统计

- EK 总数：15（EK-01 ~ EK-15）
- 出边：全部 ≥1（均值 ~3.2）
- 游离 EK：0
- 边类型覆盖：mechanism / subsystem / causal / dependency / constraint / contrast 六类全用
- 主要簇源：程序化锚（EK-01/02/05）、提示校准族（EK-07/08/09/10/11）、运行时族（EK-12/13/14/15）

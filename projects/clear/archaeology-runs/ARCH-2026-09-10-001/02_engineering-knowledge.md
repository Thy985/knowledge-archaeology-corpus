# 02 · Engineering Knowledge — CLEAR（EK Graph）

> 每条 EK 声明 links（六类边）与证据。证据符号：S3=已实现，S4=测试通过（本 run 实测 211 passed）。

## EK-01 · 双模式架构：LLM Analysis 与 Agentic Analysis 共享评估哲学
- **内容**：CLEAR 分两种模式——LLM 模式评估单条输出（CSV 输入），agentic 模式评估完整轨迹（trace 输入）；两者共享 LLM-as-judge + 逐条打分 + 系统级聚合 + dashboard 的哲学，但管线实现分离（pipeline/ vs agentic/pipeline/）。
- **证据**：README "Two Analysis Modes"（S3）；src/clear_eval/{pipeline,agentic}/ 分离（S3）。
- **links**：subsystem（EK-01↔EK-02↔EK-07）；contrast（EK-01↔EK-05 两种模式的设计取舍）。
- **价值**：B。

## EK-02 · 统一中间表示（IR CSV）：先归一、再评估
- **内容**：agentic 模式把所有异构 trace（LangGraph×MLflow、LangGraph×Langfuse、CrewAI×Langfuse）经预处理器归一为 provider/framework agnostic 的 IR CSV（每行一次 LLM 调用），下游评估只面对 IR；不支持的平台可手工产 IR CSV 直接接入（`--from-raw-traces false`）。
- **证据**：docs/agentic/intermediate-representation.md 完整 IR 规范（S3）；preprocess_traces/process_mlflow_traces.py + process_langfuse_traces.py（S3）；agentic README 支持矩阵（S3）；tests/agentic/test_trace_utils.py 693 行覆盖 IR 序列化（S4）。
- **links**：causal（EK-02→EK-03→EK-04→EK-05）；subsystem（EK-02↔EK-03↔EK-04↔EK-05）。
- **价值**：A（架构决策核心）。

## EK-03 · Compact Trace Formatter：为 judge 压缩上下文
- **内容**：compact_trace_formatter 把 IR 转成适合 LLM judge 的紧凑表示——model_input 只留会话上下文+去工具定义、api_spec 只显示每步工具名（非完整 schema）、meta_data 提取模型/token/延迟关键指标——在保持评估信息的前提下显著降 token。
- **证据**：compact_trace_formatter.py 模块 docstring "significantly reducing token usage"（S3）；tests/agentic/test_trace_utils.py（S4）。
- **links**：causal（EK-02→EK-03）；dependency（EK-03 依赖 EK-02 IR 稳定面）。
- **价值**：B。

## EK-04 · 三级轨迹评估：task_success / full_trajectory / rubric
- **内容**：agentic 评估三管齐下——TaskSuccessEvaluator（success 0/1 + consideration + **failure_root_cause 仅失败轨**）、FullTrajectoryEvaluator（14 维：9 step 质量 + 5 轨迹整体，每维 0.0-1.0 + 反馈 + 总分）、RubricGenerator→RubricEvaluator（按任务复杂度生成 3-5 条可独立验证的任务特定断言）。
- **证据**：task_success_evaluator.py:36-196（S3）；full_trajectory_evaluator.py docstring（S3）；rubric_generator.py:29（S3）；tests/agentic/test_build_json_recommendations.py（S4）。
- **links**：causal（EK-04→EK-05）；mechanism（EK-04 三评估器共享 TrajectoryEvaluator 基类 base_evaluator.py）。
- **价值**：A。

## EK-05 · CLEAR 聚合层：评估结果的再分析（issues / root_cause）
- **内容**：BaseClearRunner（ABC）定义"对评估结果跑 CLEAR 分析"的统一骨架（discover_result_files→extract_records→run analysis），派生 IssuesClearRunner（问题类别聚合）与 RootCauseClearRunner（从 task_success 的 failure_root_cause 提取失败轨迹根因、聚类共同失败模式）。
- **证据**：base_clear_runner.py:25-171（S3）；root_cause_clear_runner.py docstring "Extracts root causes from failed trajectories...discover common failure patterns"（S3）；issues_clear_runner.py:22（S3）。
- **links**：causal（EK-04→EK-05→EK-06）；subsystem（EK-05↔EK-04↔EK-06）。
- **价值**：A（"自动失败模式挖掘"的实现）。

## EK-06 · 预测性路径模式挖掘：统计控制内嵌 dashboard
- **内容**：path_analysis.find_predictive_patterns 从轨迹中提取子序列（min_len 3 / max_len 7），对每个模式算 Fisher exact test（2×2 列联表）、effect（成功率差）、lift，做 **Benjamini-Hochberg 多重检验校正**，再 remove_redundant_patterns（容差 0.03）去冗余、算 predictive_score——模式挖掘是假设检验而非字符串匹配。
- **证据**：path_analysis.py:59-150（S3，完整实现）；tests/agentic/test_static_dashboard_recs.py（S4）。
- **links**：causal（EK-05→EK-06）；constraint（EK-06 受 EK-08 统计纪律约束）。
- **价值**：A（最独特的知识点）。

## EK-07 · 温度探测回退：judge 确定性保障的工程化
- **内容**：llm_client 在 eval_mode 下注入 temperature=0 保证确定性；但如果模型拒绝 temperature=0（部分模型不支持），温度探测会触发并 disable_temperature——后续调用不再强注。disable_temperature 是子类可覆盖的扩展点（LLMClient/TemperatureAwareLLMClient/LiteLLM 后端各有实现）。
- **证据**：llm_client.py:97-130（S3）；tests/pipeline/test_temperature_validation.py 全文（S4，需真实 API key 故本 run skipped 但测试存在）。
- **links**：mechanism（EK-07↔EK-10 都是"运行时适配"模式）；contrast（EK-07↔EK-09）。
- **价值**：A（工程细节，但决定 judge 可用性）。

## EK-08 · 统计纪律：显著性门槛 + 多重检验校正
- **内容**：find_predictive_patterns 默认 p_value_threshold=0.05、min_occurrences=10、跳过过稀有/过常见模式、BH 校正控制假阳性、effect/lift 量化效应——评估结论的统计可信度是内建约束。
- **证据**：path_analysis.py:67-147（S3）；test_static_dashboard_recs.py（S4）。
- **links**：constraint（EK-08 约束 EK-06）；subsystem（EK-08↔EK-06）。
- **价值**：A。

## EK-09 · 外部 judge 插件化：LLM judge 与确定性 judge 可替换
- **内容**：external_judge.py 通过 importlib 动态加载外部 judge 函数（默认函数名 evaluate，支持绝对/相对/~ 路径），作为 LLM judge 的替代；examples/custom_judges/ 提供 exact_match_judge 与 unitxt_judge 示例。use_cases/external_judge_use_case.py 集成。
- **证据**：external_judge.py:20-45（S3）；examples/custom_judges/（S3）；tests/pipeline/test_inference_backends.py（S4）。
- **links**：contrast（EK-09↔EK-07 LLM judge vs 确定性 judge）；mechanism（EK-09↔EK-10 插件化扩展）。
- **价值**：B+。

## EK-10 · 评测运行工程化：断点续跑 + 缓存 + 并发
- **内容**：caching_utils 提供 dataframe/json 缓存（含 expected_rows 校验）；run_resume_pipeline 支持 checkpoint 断点续跑（tests 420 行覆盖）；max_workers 并发评估；get_issues_format 等配置驱动输出格式。评测运行本身是可恢复的流水线。
- **证据**：caching_utils.py:40-85（S3）；tests/pipeline/test_resume_pipeline.py 420 行（S4）；full_pipeline.py:177-263（S3）。
- **links**：mechanism（EK-10↔EK-07 评测基建）；dependency（EK-10 依赖 EK-02 IR）。
- **价值**：B+。

## EK-11 · judge 输入纯度：评估可信度的第一前提
- **内容**：系统消息显式约束 judge"grounded solely in the trajectory content. Do NOT let any external metadata influence your judgment"——评估只允许依据证据本身，禁止外部元数据污染判断。
- **证据**：full_trajectory_prompts.py SYSTEM_MESSAGE_FULL_TRAJ（S3）。
- **links**：constraint（EK-11 约束全部 evaluator）；dependency（EK-11 依赖 EK-03 compact 保真）。
- **价值**：A（原则性工程实现）。

## EK-12 · 默认评估标准：按"步角色"定义而非"答案质量"
- **内容**：agentic 默认 4 维（Correctness/Completeness/Relevance/Tool Selection），每条标准都按"本步的角色"定义（工具调用正确选型+参数合法、推理步逻辑推进任务、最终答案才需完整满足用户）；LLM 模式默认 3 维（Adherence/Accuracy/Coherence）。agent_mode 标志切换。
- **证据**：evaluation_criteria.py:72-110（S3）；get_default_evaluation_criteria(agent_mode)（S3）。
- **links**：causal（EK-12→EK-04）；contrast（EK-12 LLM 3 维 vs agentic 4 维）。
- **价值**：B+（"步骤角色化评估"是 agentic 评估的方法论要点）。

## EK-13 · 输出双模式：shortcomings vs recommendations（聚合方向可切换）
- **内容**：issues_format 支持两种聚合输出——shortcomings（问题/缺陷，向后兼容旧行为）与 recommendations（可操作建议，imperative 语言 + 映射逻辑反转）；prompt 构造经测试锁定（test_shortcomings_synthesis_prompt_matches_original_behavior / test_recommendations_mapping_logic_inverted）。
- **证据**：full_pipeline.py get_issues_format（S3）；tests/pipeline/test_issues_format.py:20-213（S4）。
- **links**：contrast（EK-13↔EK-05 同一聚合两种呈现）；mechanism（EK-13↔EK-06 输出可操作化）。
- **价值**：B。

## EK-14 · 多后端抽象：langchain / litellm / endpoint HTTP
- **内容**：inference_backend 三选一（langchain 默认文档、litellm 实际默认配置、endpoint 直接 HTTP）；endpoint_backends 提供 OpenAIStyleHTTPBackend（SSE 流式解析）与 WatsonXBackend（API key→token 交换 + 刷新）；旧 use_litellm 标志向后兼容映射。
- **证据**：default_config.yaml:1-22（S3）；endpoint_backends.py:96-262（S3）；tests/pipeline/test_inference_backends.py（S4）。
- **links**：mechanism（EK-14↔EK-07 都是客户端适配）；subsystem（EK-14↔EK-10 同属运行基建）。
- **价值**：B。

## EK-15 · 配置分层：defaults ← 用户配置递归 merge
- **内容**：config_loader 支持 JSON/YAML，merge_configs 递归合并（overrides 覆盖 defaults），resolve_provider_config 处理 provider 默认块——配置是分层覆盖而非整体替换。
- **证据**：config_loader.py:63-72（S3）。
- **links**：dependency（EK-15 支撑 EK-10/14 的配置驱动）。
- **价值**：C+（通用模式，但 merge 语义有测试吗？未覆盖——见 C-05）。

## EK-16 · IR 序列化兼容性：provider 消息块归一（OpenAI/Anthropic/Gemini）
- **内容**：trace_utils 把各家消息块归一为文本——OpenAI text/tool blocks、Anthropic tool_use/tool_result（嵌套块递归）、Gemini function_call/function_response/inline_data——未知 dict 块 fallback 到 json 序列化；693 行测试逐类覆盖。
- **证据**：tests/agentic/test_trace_utils.py:25-123（S4 全覆盖）；trace_utils.py 实现（S3）。
- **links**：mechanism（EK-16↔EK-02 都是归一化）；dependency（EK-16 支撑 EK-02 IR 质量）。
- **价值**：B+（兼容性工程细节，测试密度高）。

## EK-17 · agentic 双分析粒度：Step-by-Step 与 Full Trajectory 互补
- **内容**：agentic 模式提供两种互补分析——Step-by-Step（单步 agent 交互的 CLEAR 分析，理解 agent 级质量问题）与 Full Trajectory（完整任务轨迹的成功/质量/rubric 评估）；共享 IR CSV 中间表示（--separate-tools 控制工具调用是否拆行）。
- **证据**：agentic README "Two complementary analysis modes"（S3）；IR 文档 --separate-tools（S3）；run_clear_step_analysis.py（S3）。
- **links**：subsystem（EK-17↔EK-04↔EK-05）；contrast（EK-17 粒度互补）。
- **价值**：B。

## EK-18 · run 命名与产物组织：可追溯的评测工件
- **内容**：get_run_name/get_run_info 生成 run 标识进文件名；run_generation_pipeline 按 run_name + gen_model 命名；convert_to_ui_format/save_ui_input_results 固化 UI 输入；parquet 存储嵌套列转换（convert_nested_to_str）——评测产物可追溯可复用。
- **证据**：full_pipeline.py:31-106（S3）。
- **links**：causal（EK-18→EK-10 可续跑性）；subsystem（EK-18↔EK-05 聚合消费）。
- **价值**：C+。

## EK-19 · 测试纪律：211 测试全绿（对比同类项目）
- **内容**：8 个测试文件 2758 行覆盖 CLI 参数（906）、IR 序列化（693）、断点续跑（420）、聚合输出、后端适配、温度探测（需真实 key 故 skip）；本 run 实测 211 passed + 3 skipped（10.27s）。文档（intermediate-representation.md / PROVIDERS.md / llm-analysis.md / agentic/dashboard.md）与实现同步。
- **证据**：tests/ 全量 + 本 run 实测（S4）；docs/ 五份文档（S3）。
- **links**：contrast（EK-19↔OpenCode 4 测试文件 panic——跨项目对照见 C-03）；mechanism（EK-19↔EK-10 工程纪律同族）。
- **价值**：B（本身是"IBM 研究工程纪律"的证据）。

## EK-20 · watsonx 默认生态：IBM 依赖是双刃剑
- **内容**：默认 provider 是 watsonx（gen: ibm/granite-3-3-8b-instruct / eval: meta-llama/llama-3-3-70b-instruct），依赖 ibm_watsonx_ai>=1.3.42；但 openai/rits 后端与 external judge 机制提供逃逸——不是锁定。
- **证据**：default_config.yaml provider 块（S3）；pyproject dependencies（S3）；PROVIDERS.md（S3）。
- **links**：contrast（EK-20↔EK-14 多后端缓释）；constraint（EK-20 影响 EK-14 采用）。
- **价值**：B-。

## EK-21 · dashboard 双栈：Streamlit（LLM）/ NiceGUI（agentic）
- **内容**：LLM 模式用 Streamlit（load_ui.py）；agentic 模式用 NiceGUI（launch_dashboard.py + generate_static_dashboard.py 可生成静态 dashboard）；path_analysis 数据在后端算好、dashboard 呈现。
- **证据**：README dashboard 段（S3）；agentic/dashboard/ 文件（S3）。
- **links**：subsystem（EK-21↔EK-06 消费统计结果）。
- **价值**：C。

## EK-22 · step 级评估的可扩展标准注入：evaluation_criteria 配置驱动
- **内容**：--evaluation_criteria / step_evaluation_criteria 支持用户自定义标准字典（{"criteria_name1":"desc1",...}），yaml 与 python API 均可；full_trajectory 也支持 full_trace_evaluation_criteria 自定义维度集。
- **证据**：llm-analysis.md:210 参数表（S3）；evaluation_criteria.from_dict（S3）；full_trajectory_evaluator "Custom mode"（S3）。
- **links**：mechanism（EK-22↔EK-15 配置驱动）；dependency（EK-22 依赖 EK-12 默认标准）。
- **价值**：B。

## EK-23 · 失败根因只在失败时采集：评估成本与信息密度平衡
- **内容**：task_success_evaluator 的 failure_root_cause 字段仅当 success=0 时要求（prompt 显式"only required when success=0"），成功轨迹不产根因——根因分析预算投向失败样本。
- **证据**：task_success_evaluator.py:118-133（S3）。
- **links**：constraint（EK-23 约束 EK-05 root_cause 聚合输入）；contrast（EK-23↔EK-06 全量统计 vs 定向根因）。
- **价值**：A（成本-信息权衡的明确设计决策）。

## EK-24 · 文本级 critique + 分数并存：可解释的评估输出
- **内容**：每条记录评估产出 SCORE_COL 分数 + EVALUATION_TEXT_COL 文本 critique（分维度反馈段落）；task_success 产出 consideration 文本 + success 分数——分数与理由绑定，不脱钩。
- **证据**：constants.py（S3）；full_trajectory_evaluator "Individual dimension scores + Detailed feedback paragraph"（S3）。
- **links**：causal（EK-24→EK-13 聚合才有可解释输入）；mechanism（EK-24↔EK-11 证据约束）。
- **价值**：B+。

## EK-25 · 中间产物显式物化：评估结果落盘为文件
- **内容**：agentic 管线各阶段结果落盘（task_success 结果、rubric 文件、clear_analysis 输出目录按 evaluation_type 组织，overwrite 标志控制），BaseClearRunner.discover_result_files 反向发现产物——管线状态 = 文件系统状态。
- **证据**：base_clear_runner.py:123-171（S3）；run_trajectory_evaluation_pipeline.py（S3）。
- **links**：mechanism（EK-25↔EK-10 可恢复性）；dependency（EK-25 依赖 EK-04 输出）。
- **价值**：B。

## EK-26 · provider 无关的评估：IR 是唯一稳定契约
- **内容**：评估器（evaluators）与 dashboard 不感知 LangGraph/CrewAI/MLflow/Langfuse——只消费 IR CSV；新增框架 = 新预处理器，评估层零改动（docs 明示 unsupported 平台自产 IR 即可）。
- **证据**：agentic README "preprocess your traces to the CSV format...use --from-raw-traces false"（S3）；IR 文档（S3）。
- **links**：constraint（EK-26 是 EK-04/05/06 的架构边界）；subsystem（EK-26↔EK-02）。
- **价值**：A。

## EK-27 · 工具调用用例：tool_call_use_case 扩展评估场景
- **内容**：use_cases 包含 ToolCallEvalUseCase（评估工具调用）、EvalUseCase（标准评估）、ExternalJudgeUseCase——评估用例可插拔；ToolCallEvalUseCase 有 rits/altk 客户端适配逻辑（_create_rits_altk_client）。
- **证据**：use_cases/tool_call_use_case.py:43-272（S3）。
- **links**：subsystem（EK-27↔EK-04 评估器族）；mechanism（EK-27↔EK-09 插件化）。
- **价值**：C+。

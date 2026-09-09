# 05 · Candidates — CLEAR（未验证假说与跨项目候选）

> 每条标注：假设状态 / 当前证据 / 缺失证据 / 验证路径。Hypothesis 不冒充 Fact。

## C-01（Hypothesis · 跨项目候选）"评估 = 证据生产管线"是评测系统通用模式
- **内容**：CLEAR 的分阶段管线（trace→IR→compact→评估→聚合→统计）猜想是所有生产级 agent 评估系统的共同形态。
- **当前证据**：CLEAR（S3/S4）+ Validation 编译器（2 例弱印证）。
- **缺失证据**：silver-shield（Benchmark Harness）的评估管线是否同构；Tafcm ADI 的评估接口。
- **验证路径**：对照 silver-shield 考古包。

## C-02（Tentative Pattern）温度探测回退（deterministic 注入失败时优雅降级）
- **内容**：eval_mode 注入 temperature=0 但探测到模型拒绝时 disable——"确定性偏好 + 运行时适配"模式；猜想在 judge 基建中常见。
- **当前证据**：CLEAR llm_client.py:97-130（S3 + 测试存在但需真实 key skip）。
- **缺失证据**：真实模型拒绝场景的端到端验证（需 API key）。
- **验证路径**：配置真实 watsonx/openai key 跑 test_temperature_validation。

## C-03（Tentative Pattern）"star 少 ≠ 质量差"的又一实例
- **内容**：CLEAR 58★ 但 211 测试全绿 + 文档完备 + 统计方法严谨；OpenCode 13.7k★ 但 4 测试文件 + panic + 归档——项目质量与生态信号系统性解耦的第三例（MemGraphRAG 亦低 star 高质量）。
- **当前证据**：CLEAR（S4 实测 211 passed）+ OpenCode（S4F）+ corpus 历史。
- **缺失证据**：量化相关性统计。
- **验证路径**：对已考古 8 项目计算 star vs 测试密度散点。

## C-04（Scope-uncertain）dashboard 统计模式的实际使用方式
- **内容**：find_predictive_patterns 产出显著路径模式——用于 dashboard 呈现（EK-06），但"这些模式是否被用来反哺评测设计（如调整标准/采样）"无证据。
- **当前证据**：path_analysis.py（S3）+ test_static_dashboard_recs（S4）。
- **缺失证据**：用户案例/文档说明模式的下游消费。
- **验证路径**：查 README/dashboard 文档或 issue 讨论。

## C-05（Unresolved Contradiction）config merge 无独立测试
- **内容**：merge_configs（递归覆盖）是配置核心语义，但仓库测试未覆盖它（tests/ 无 config_loader 测试）——文档声称"user config overrides defaults"，无测试锁定。
- **当前证据**：config_loader.py:63-72（S3）；tests 目录（S3 无对应测试）。
- **缺失证据**：merge 行为回归测试。
- **验证路径**：新增 merge 语义测试（但只读约束——记录建议即可）。

## C-06（Hypothesis）LLM 模式与 agentic 模式共享聚合层的可能性
- **内容**：两条管线（pipeline/ vs agentic/pipeline/）代码重复（评估标准、聚合逻辑各一份）——猜想后续可统一为共享核心。
- **当前证据**：目录分离（S3）。
- **缺失证据**：重构意图（无 ADR/issue 证据）。
- **验证路径**：查 PyPI 版本历史（2.0.5 演进）。

## C-07（Observation）evaluation_criteria 的 agent_mode 切换是"步角色化评估"的显式表达
- **内容**：agentic 4 维标准按"本步角色"定义（工具调用正确性 ≠ 最终答案完整性）——这是 agentic 评估与 LLM 评估的方法论分水岭；猜想"步角色化"是 agent 评估的普遍需求。
- **当前证据**：evaluation_criteria.py:88-110（S3）。
- **缺失证据**：其它框架（AgentEval 等）是否采用同构标准设计。
- **验证路径**：对照 AgentEval（ACL 2026，DAG 节点评估）的 criteria 设计。

## C-08（Hypothesis）IBM 生态依赖限制采用，但插件逃逸机制缓解
- **内容**：watsonx 默认 + ibm_watsonx_ai 依赖，但 openai/rits/endpoint 后端 + external judge 提供逃逸——猜想实际社区采用率受默认生态影响。
- **当前证据**：default_config.yaml + pyproject（S3）；58★（GitHub API）。
- **缺失证据**：PyPI 下载量。
- **验证路径**：查 pypistats clear-eval。

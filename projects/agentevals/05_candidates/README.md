# 05 — Candidates（未确认内容，全部留在候选）

> 铁律：不能确认的 → Candidate；Hypothesis 不冒充 Fact；Cross-project Candidate 不写成已验证 Principle。

## C-01 — tool trajectory 三态匹配的"语义粒度"假设
- **内容**：EXACT/IN_ORDER/ANY_ORDER 匹配 tool 名称序列（含 args 结构）是否足够表达"意图级"轨迹正确性？
- **证据**：`ToolTrajectoryCriterion.MatchType` 三态（builtin_metrics.py:188）+ cli.py:115 + config.py:14——匹配对象是 tool call 的 name/args（extract_trajectory_details：`{"name": tc.name, "args": tc.args}`）。
- **状态**：Hypothesis / Scope-uncertain——ADK 层实现的匹配语义（是否比较 args 值、部分匹配、排序容错）未在 agentevals 仓库内实现（复用 ADK），需读 ADK 源码确认。
- **缺失证据**：google.adk.evaluation.trajectory_evaluator.TrajectoryEvaluator 的匹配算法（外部依赖，本快照未含）。

## C-02 — "数据合成补全"（_enrich_app_details）是否会掩盖遥测缺陷
- **内容**：`_enrich_app_details` 从 tool_names 合成最小 FunctionDeclaration（builtin_metrics.py:120-156），使无 schema 的 multi-turn 指标可评工具质量——**合成数据能评分，但评的是"名称级别"质量**。
- **状态**：Tentative Pattern——合成补全是有意的（docstring 明示），但其对评分偏差的影响未量化。
- **验证路径**：构造"真实 schema vs 合成 schema"同 trace 对比评分差异（benchmark 候选，见 06 B-1）。

## C-03 — 跨项目假说：dsh session-telemetry-otel 对齐 GenAI semconv 后可直接接入 agentevals
- **内容**：agentevals 边界声明（README）点名 Claude Code/Codex/OpenCode 不发射 GenAI semconv；DeepSeek Harness 有 session-telemetry-otel（已考古）——若 dsh 对齐 semconv，其 harness 遥测可被 agentevals 直接评分。
- **状态**：Cross-project Hypothesis——需验证 dsh 遥测字段与 GenAI semconv（gen_ai.request.model / gen_ai.input.messages 等）的映射可行性。
- **缺失证据**：dsh 侧遥测 schema 与本仓库 docs/otel-compatibility.md 的字段对照（跨仓库，需另跑）。

## C-04 — CI 门禁的"确定性"上限 = 遥测覆盖度
- **内容**：CI/CD ready（README 声明）+ golden gating 的可靠性受遥测保真度约束；若 agent 不发 tool spans，trajectory 指标静默通过/失败？
- **状态**：Hypothesis / Scope-uncertain——未发现"遥测缺失时显式报错"的强制机制（对比 EK-07 凭据 fail-closed；轨迹类指标无等效 fail-closed）。
- **验证路径**：构造"零 tool span"trace 跑 trajectory 指标，观察行为（benchmark 候选，见 06 B-2）。

## C-05 — ADK 复用策略的长期风险
- **内容**：评估语义全部委托 Google ADK（EK-06），且 judge 凭据依赖 ADK 私有 seam（EK-07 TODO upstream）——ADK 行为变更 = agentevals 评测语义变更。
- **状态**：Tentative Pattern / Scope-uncertain——README "Expect breaking changes" 与 ADK 上游变更叠加的风险敞口未量化。
- **缺失证据**：ADK 版本兼容矩阵（agentevals 锁定的 google.adk 版本范围未在 pyproject 显式声明为紧约束）。

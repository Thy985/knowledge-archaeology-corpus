# 00 — Overview：agentevals（Trace-based Agent Evaluation）

## 一句话定位

agentevals 是框架无关的 Agent 评测方案：**从 OpenTelemetry trace 直接评分**（record once, evaluate many times）——不重放昂贵 LLM 调用，以 golden eval sets 做确定性 pass/fail 门禁，CLI/Web UI/MCP server/Helm chart 四接口，Local-first。

## 为什么选它（Job Selection 摘要）

- 雷达 #11（2026-09-12）新建 A 级候选 `[cand]trace-based-agent-eval-2026`——Trace-based Evaluation 新范式（开源 agentevals + Agent TraceBench + AWS Agent Health 三线独立确认）。
- 与已考古 Corpus 三连：OpenCode（遥测缺口被 agentevals 边界声明点名）、DeepSeek Harness（session-telemetry-otel 若对齐 GenAI semconv 可接入）、Agentic CLEAR（trace 作为证据）。
- Benchmark 学习价值：tool trajectory 匹配（EXACT/IN_ORDER/ANY_ORDER）+ golden eval sets + CI 门禁——与 Knowledge Archaeology 的"证据可重放"命题同构。

## 三个核心发现（Top Findings）

### F1 — "评测与执行解耦"是硬架构：no re-execution 由 loader 归一化层承载
trace 输入（Jaeger JSON / OTLP JSON / OTLP JSONL / Tempo batches）经 loader 自动检测 → 归一为内部 Trace/Span → converter 提取（ADK 格式优先于 GenAI semconv）→ Invocation。**一旦 trace 落盘/入库，评测可任意次执行且零额外 token**。代价：评测质量完全取决于遥测覆盖度——agentevals 明确声明 Claude Code/Codex/OpenCode 不发射 GenAI semconv，需数千行胶水适配（README + docs/otel-compatibility.md）。

### F2 — 复用 Google ADK 评估原语，而非自研评分器
`builtin_metrics.py` 把指标名映射到 ADK 的 EvalMetric/Criterion（ToolTrajectoryCriterion / LlmAsAJudgeCriterion / HallucinationsCriterion / RubricsBasedCriterion / LlmBackedUserSimulatorCriterion / BaseCriterion）；eval set 遵循 ADK EvalSet schema（可移植）。**选型决策：评估语义归 ADK，工程（trace 摄取/存储/CI/sinks/MCP）归自己。**

### F3 — Judge 凭据注入走 ADK 私有 seam + fail-closed
`_inject_judge_credential` 通过 ADK 私有属性（`_judge_model_options`/`_judge_model`）注入用 API key 构建的 judge model：credential_ref 未解析 → 评测返回 error（**fail-closed**，不静默）；`get_evaluator` 每评一次返回新实例 → 变异无共享状态、并发安全；代码内 TODO(upstream) 明示依赖私有 seam 是技术债。

## 质量指标速览

- EK：16 条（EK-01..16，全带 links）；KO：4 个；Candidates：4 个
- 测试证据：**798 passed / 24 skipped**（40.19s，补 pytest-asyncio 后；初跑 69 failed 全为该环境缺插件，非产品缺陷）
- 独立验证：ACCEPT（详见 06）

## 已知边界（诚实声明）

- 本项目**快速迭代中**（README: "Expect breaking changes"）。
- 评测上限 = 遥测保真度；非 GenAI-semconv harness（dsh/Codex/OpenCode）需胶水层。
- LLM judge 类指标（final_response_match_v2 / hallucinations_v1 等）为概率性，与确定性 trajectory 门禁属不同信任层级。

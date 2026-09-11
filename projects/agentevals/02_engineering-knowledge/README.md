# 02 — Engineering Knowledge（EK Graph）

> 规则：每条 EK 声明 `links`（mechanism/subsystem/causal/dependency/constraint/contrast）。全部证据可回溯 `src/agentevals/` 与 `docs/`。

## EK-01 — No re-execution：评测与执行解耦（trace 即证据）
**Fact**：`run_evaluation_from_traces(traces, config, eval_set)` 直接消费预加载 trace，**不调用任何 agent/LLM**；README "score agents from existing traces without replaying expensive LLM calls"。评测可任意次重复且零额外 token。
**Why**：LLM 执行昂贵且非确定；trace 是一次性采集的原始证据，评测是纯函数式消费。
**links**：`mechanism` EK-01↔EK-02（归一化是解耦前提）；`causal` EK-01→EK-03；`contrast` EK-01↔EK-14（LLM judge 仍需 API 但非重放）。

## EK-02 — 格式自动检测 + 归一化层（多遥测源统一入口）
**Fact**：`loader/auto.py` `detect_format()` 按文件形状识别（.jsonl→OTLP；resourceSpans/batches/trace.{}→OTLP；data→Jaeger）；未知/不可读→None。`_LOADERS` 注册表（jaeger-json / otlp-json）分发。
**Why**：调用方无需知道 trace 出自哪个 tracing 系统；新增格式=注册表加条目。
**links**：`mechanism` EK-02↔EK-05（双格式提取器）；`dependency` EK-02 依赖 EK-01（loader 产出统一 Trace 才能 no-re-exec）。

## EK-03 — GenAI semconv 兼容矩阵：属性版本演进显式化
**Fact**：docs/otel-compatibility.md 显式列出 v1.31.0+（agent/tool 元数据）、v1.37.0+（provider.name 取代 gen_ai.system）、v1.40.0+（temperature/max_tokens/top_p/top_k）、cache token usage（Anthropic/OpenAI prompt caching）。探测键：`gen_ai.request.model` / `gen_ai.input.messages`。
**Why**：semconv 是活的规范，版本漂移会造成静默字段缺失；显式矩阵让"支持什么版本"可审计。
**links**：`constraint` EK-03 约束 EK-04（版本差异决定提取逻辑分支）；`subsystem` EK-03↔EK-02。

## EK-04 — 消息双 schema 归一 + 三通道内容投递
**Fact**：GenAI 消息支持 content-based（OpenAI/LangChain v2）与 parts-based（v1.36.0+）双 JSON schema，按消息自动检测；tool_calls 归一为 `{name, id, arguments}`。内容可经 ①span attributes ②log records（推荐，需 OTLPLogExporter + `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true`）③span events（deprecated）投递；events 在 streaming/processor 三层被提升为 span 级属性。
**Why**：不同 instrumentor 已部署不同投递机制；向后兼容 + 归一，避免下游看到异构形状。
**links**：`mechanism` EK-04↔EK-02（都是归一）；`constraint` EK-04→EK-13（日志通道要求双 exporter）。

## EK-05 — 格式优先级：ADK 专有格式 > GenAI semconv
**Fact**：`converter.py` `get_extractor()`——trace 同时含 ADK（gcp.vertex.agent scope）与 GenAI 属性时 **ADK 优先**（"provides richer structured data"）；`extraction.py` `TraceFormatExtractor` 协议（detect/format_name/find_invocation_spans/find_llm_spans_in/find_tool_spans_in/classify_span）+ `AdkExtractor` 实现。
**Why**：专有格式结构更丰富；协议化提取器使新格式可插拔。
**links**：`mechanism` EK-05↔EK-02；`contrast` EK-05↔EK-04（专有 vs 标准优先的取舍）。

## EK-06 — 复用 Google ADK 评估原语（不自研评分器）
**Fact**：`builtin_metrics.py` `build_eval_metric()` 映射：tool_trajectory_avg_score→ToolTrajectoryCriterion（MatchType 三态）、final_response_match_v2→LlmAsAJudgeCriterion、hallucinations_v1→HallucinationsCriterion、rubric_*→RubricsBasedCriterion、per_turn_user_simulator→LlmBackedUserSimulatorCriterion、response_match/response_evaluation/safety/multi_turn_*→BaseCriterion。`get_evaluator()` 对轻量指标直接 import（"avoid pulling in heavy deps (numpy/rouge_score)"）。eval set 遵循 ADK EvalSet schema（**可移植**）。
**Why**：评估语义是成熟领域，复用 ADK 而非重造；直接 import 控制依赖体积。
**links**：`dependency` EK-06 依赖 EK-07（credential 注入依赖 ADK 实例语义）；`contrast` EK-06↔EK-12（builtin 复用 vs custom 自写）。

## EK-07 — Judge 凭据注入：私有 seam + fail-closed + 并发安全（关键安全机制）
**Fact**：`_inject_judge_credential(evaluator, api_key, base_url)`——通过 ADK 私有属性 `_judge_model_options`/`_judge_model`（`LlmAsJudge._setup_auto_rater` 设置）替换 auto-rater model；keyed on seam 而非 class，单路径覆盖 FinalResponseMatchV2/rubric_*/HallucinationsV1。`credential_ref` 未解析 → `MetricResult(error=...)`（**fail-closed，不静默**）。`get_evaluator` 每次返回新实例 → 变异无共享状态（并发安全）。TODO(upstream)：私有 seam 依赖 = 技术债，建议 JudgeModelOptions 自带 credential。
**Why**：judge API key 不能进配置文件明文；ADK 无官方 credential 通道 → 取 seam 注入；fail-closed 防止"以为有凭据实际没注入"的静默降级。**项目自带防护**：test_credential_injection.py 含 seam guard 测试（"fails loudly if the ADK judge seam moves"）——私有 seam 若在上游变更，测试会显式失败而非静默通过。
**links**：`constraint` EK-07 约束 EK-06（复用 ADK 的代价 = 依赖其私有 seam）；`mechanism` EK-07↔EK-08（都是 fail-closed 族）；`causal` EK-07→EK-11（webhook 重试携带凭据上下文）。

## EK-08 — 确定性结果身份：SHA-256 组合去重（幂等基石）
**Fact**：`compute_result_id(run_id, eval_set_item_id, evaluator_name)` = canonical SHA-256(`{run_id}|{eval_set_item_id}|{evaluator_name}`)；存储层 `INSERT ... ON CONFLICT (result_id) DO UPDATE`——重试的 webhook 与重试的执行器都干净去重。
**Why**：评测管道有异步重试（webhook 回退/worker 重启），结果必须可重放且不重复。
**links**：`mechanism` EK-08↔EK-10（幂等提交）；`dependency` EK-08 依赖 EK-01（纯函数消费才可重放）。

## EK-09 — Run 生命周期状态机 + 409 spec-mismatch + worker lease 租约
**Fact**：RunStatus 五态（QUEUED/RUNNING/SUCCEEDED/FAILED/CANCELLED）；`RunService.submit` 幂等（同 run_id 已存在）——spec 不同 → `RunSubmitConflict` → HTTP 409 + 返回已持久化 run 供客户端 reconcile。**并发领取**：`claim_next(worker_id, lease: timedelta, max_attempts)` 只取 QUEUED 且 attempt<max_attempts 的 run（created_at 排序，oldest first）；heartbeat 续租——"Returns False if the run was cancelled or lost"（repos/__init__.py:50）；cancel 为 **cooperative**：置 `cancel_requested=True`，worker 在下次 heartbeat 观察（memory.py:104 `return not run.cancel_requested`）。
**Why**：多 worker 场景防双重领取（lease 租约）；重试受 max_attempts 约束（防死循环）；取消不抢占 worker 而是协作停（避免中途破坏状态）。
**links**：`mechanism` EK-09↔EK-08（幂等族）；`causal` EK-09→EK-11（sink 消费已提交 run）；`dependency` EK-09 依赖 EK-10（worker 并发消费需限流配套）。

## EK-10 — 双 semaphore 有界并发（traces × evals 双层限流）
**Fact**：`runner.py` `asyncio.Semaphore(config.max_concurrent_traces)` + `asyncio.Semaphore(config.max_concurrent_evals)` 双层限制；`_evaluate_trace_bounded` 先取 trace 槽再取 eval 槽。
**Why**：trace 级（转换/IO）与 eval 级（子进程/judge API）资源成本不同，需独立限流防级联。
**links**：`mechanism` EK-10↔EK-07（并发安全的另一面）；`dependency` EK-10 依赖 EK-06（ADK evaluator 调用在限流内）。

## EK-11 — 可插拔结果 Sink：工厂注册表 + fanout
**Fact**：`run/sinks.py` `ResultSink` 协议 + StdoutSink/FileSink/HttpWebhookSink + `SinkFanout`（多目标扇出）+ `register_sink_factory(kind, factory)` 全局注册表 + `_extract_env_headers`（auth 从环境取，不落配置）。
**Why**：结果消费目标多样（CI 日志/文件/webhook 回调）；插件式注册使新 sink 零核心改动。
**links**：`mechanism` EK-11↔EK-05（注册表模式）；`contrast` EK-11↔EK-06（输出面可插拔 vs 评估面复用）。

## EK-12 — Custom Evaluator：多 runtime 子进程 JSON 协议
**Fact**：`custom_evaluators.py` `EvaluatorBackend`（transport 抽象：subprocess/HTTP/Docker）+ `Runtime`（PythonRuntime/NodeRuntime）+ `supported_extensions()` 解析；`_run_subprocess` 向 stdin 写 JSON、从 stdout 读 JSON、timeout 控制；venv 隔离（evaluator/venv.py）。
**Why**：评分逻辑应允许任何语言编写（Python/JS/任意语言/OpenAI Eval API）；子进程 + JSON 行协议 = 语言无关最小契约。
**links**：`mechanism` EK-12↔EK-09（子进程生命周期对应 run 状态）；`contrast` EK-12↔EK-06（自写 vs 复用）。

## EK-13 — 流式增量处理：WebSocket spans/logs 双处理器
**Fact**：`streaming/processor.py` `AgentEvalsStreamingProcessor` + `AgentEvalsLogStreamingProcessor`；SDK `AgentEvals` 在 session 期间向活动 TracerProvider 注入 processor，streaming=False 时 context manager 变 no-op（env 门控 `AGENTEVALS_STREAM`）。
**Why**：长会话需边跑边采（增量），而非等 trace 落盘再离线评；no-op 降级保持 SDK 常驻代码。
**links**：`contrast` EK-13↔EK-01（流式在线 vs 文件离线两条摄取轨）；`dependency` EK-13 依赖 EK-04（日志通道）。

## EK-14 — 评测信任谱系：确定性门禁 vs LLM judge
**Fact**：确定性指标（tool_trajectory EXACT/IN_ORDER/ANY_ORDER、response_match、threshold 门禁）与概率性指标（final_response_match_v2/hallucinations_v1/rubric_*/user_simulator → LLM judge）并存；`METRICS_NEEDING_EXPECTED` 要求 golden eval set（无 expected → MetricResult error）。
**Why**：CI 门禁需要可复现的确定性判定；语义质量需 judge——两类信任层级显式分离。
**links**：`contrast` EK-14↔EK-06（同一 ADK 框架承载两级信任）；`constraint` EK-14 约束 EK-15（无 golden 时部分指标不可用）。

## EK-15 — 边界声明：非 GenAI-semconv harness 的遥测缺口
**Fact**：README 与 docs 明确：Claude Code/Codex/OpenCode **不发射 GenAI semconv**——"对每个 harness 的专有遥测做胶水适配需数千行代码，且主导信号是'最终输出感觉对不对'而非'工具轨迹对不对'"。
**Why**：诚实界定能力边界；评测上限 = 遥测保真度。
**links**：`constraint` EK-15 约束 EK-01（no-re-exec 的前提是 trace 可采集）；`contrast` EK-15↔EK-05（ADK 富格式 vs 缺失格式的两个极端）。

## EK-16 — dev 依赖缺口：asyncio 测试配置未声明插件（测试环境发现）
**Fact**：pyproject 配置 `asyncio_mode`（pytest 配置）但 **pytest-asyncio 未在依赖/测试 optional 中声明**——本地初跑 69 failed / 29 errors 全为 "async def functions are not natively supported"；`pip install pytest-asyncio` 后 **798 passed / 24 skipped**。
**Why**：测试矩阵依赖隐式环境；README DEVELOPMENT.md 未列出该插件 → 新贡献者首跑必挂。
**links**：`constraint` EK-16 约束 EK-10（并发测试需要 async 支持）；`contrast` EK-16↔EK-08（确定性 vs 环境依赖）。

## EK Graph 质量自检
- 16 条 EK 全部有 links（平均出边 >1）；孤立 0；聚合覆盖率 100%（见 03）。

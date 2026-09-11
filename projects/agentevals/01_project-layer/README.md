# 01 — Project Layer（agentevals 项目地图）

## 1.1 它是什么 / 怎么运行

- **定位**：Agent 评测系统。输入 = OTel traces（Jaeger JSON / OTLP JSON / OTLP JSONL / Tempo v1 batches / Tempo v2 trace wrapper），输出 = 每 metric 每 eval case 的 score + pass/fail（threshold 门禁）。
- **运行入口**：CLI（`agentevals eval <traces> --config ...`）、FastAPI（`/api/runs`、`/api/evaluate`、`/api/otlp` 接收器）、`agentevals mcp`、Web UI（`ui/`）、SDK（`AgentEvals` context manager/decorator）。
- **安装**：`pip install agentevals-cli`（PyPI）；本地 `pip install -e .` 后 `agentevals --help`。

## 1.2 核心模块

```
loader/auto.py        格式自检 + 统一加载入口（detect_format / load_traces）
converter.py          格式检测（get_extractor：ADK > GenAI semconv）
genai_converter.py    GenAI semconv → 归一 Invocation（content-based / parts-based 双 schema）
extraction.py         span 分类提取（is_llm_span / is_tool_span / is_invocation_span；AdkExtractor）
builtin_metrics.py    指标 → ADK EvalMetric/Criterion；_enrich_app_details 合成；judge credential 注入
runner.py             评测编排（双 semaphore 并发控制 + 进度回调）
run/service.py        Run 生命周期（submit 幂等 / 409 spec-mismatch / list 分页 / cancel）
run/worker.py         异步 worker 执行队列
run/sinks.py          结果输出：Stdout / File / HttpWebhook / SinkFanout + sink 工厂注册表
storage/              RunRepository / ResultRepository（memory + postgres）/ migrations
api/                  FastAPI + OTLP receiver（HTTP/GRPC）+ streaming routes
streaming/            AgentEvalsStreamingProcessor（WebSocket 增量 spans / logs）
custom_evaluators.py  任意语言 evaluator（Python/Node/任意语言 + HTTP/Docker 后端）
resolvers/            kubernetes resolver + 凭据解析（get_resolved_credential）
mcp_server.py         MCP 会话式评测接口
openai_eval_backend.py OpenAI Eval API 卸载后端
sdk.py                AgentEvals SDK（TracerProvider processor 注入）
```

## 1.3 生命周期（Run 状态机）

```
submit → QUEUED → RUNNING → SUCCEEDED
                        ├──→ FAILED（error_text）
                        └──→ CANCELLED（cancel 请求）
Result 状态：PASSED / FAILED / ERRORED / SKIPPED
submit 幂等：同 run_id 不同 spec → RunSubmitConflict → HTTP 409（客户端 reconcile）
result_id = SHA-256(run_id | eval_set_item_id | evaluator_name) → 确定性去重（重试幂等）
```

## 1.4 配置

| 配置 | 说明 | 证据 |
|---|---|---|
| `EvalParams.evaluators` | builtin（threshold/judge_model/trajectory_match_type/credential_ref/judge_base_url）+ custom | config.py |
| `trajectory_match_type` | EXACT（默认）/ IN_ORDER / ANY_ORDER（大小写不敏感，非法值拒绝） | config.py:14 + cli.py:115 |
| `credential_refs` | RunSpec 逻辑名 → 解析后注入 judge（fail-closed） | builtin_metrics.py:395-404 |
| `max_concurrent_traces` / `max_concurrent_evals` | 双 semaphore 有界并发 | runner.py |
| sinks | stdout/file/http_webhook 可插拔（register_sink_factory） | run/sinks.py |

## 1.5 权限与治理

- **Judge 凭据**：逻辑名引用（credential_ref）而非明文值；解析失败 → MetricResult error（fail-closed）；通过 ADK 私有 seam 注入（技术债，TODO(upstream)）。
- **幂等与去重**：run 提交幂等（409 冲突路径）+ result_id SHA-256 确定性（webhook 重试/执行重试去重）。
- **可插拔边界**：sink 工厂注册表、evaluator 来源注册表（EvaluatorResolver）、runtime 注册表（Python/Node）。

## 1.6 外部依赖

- **opentelemetry-***（proto/sdk/exporter-otlp-http/instrumentation-openai-v2/instrumentation-openai-agents-v2）——核心
- **Google ADK（google.adk.evaluation）**——评估原语复用（trajectory_evaluator 等直接 import，避免重依赖）
- fastapi / pydantic-settings / click / rich；可选：Postgres（storage）、OpenAI Eval API、Docker（custom evaluator 后端）、Kubernetes（resolvers）、numpy/rouge（重型 metrics 按需）

# Repository Snapshot Artifact — agentevals

## Snapshot

| 字段 | 值 |
|---|---|
| repository | https://github.com/agentevals-dev/agentevals.git |
| commit_sha | `65e677b79affdda5bf056fc0e72e34f2b7cbfa8b` |
| branch | main |
| repository_version | PyPI `agentevals-cli`（pyproject `name = "agentevals-cli"`） |
| analysis_timestamp | 2026-09-12T02:00+08:00（cron 触发） |
| shallow_clone | `--depth 1` → `archaeology-jobs/ARCH-2026-09-12-001/repo` |
| 规模 | 5.2MB / 134 个 .py / tests 43 个文件 |
| GitHub API | language Python、Apache-2.0、158 stars、archived=False、created 2026-02-24、pushed 2026-09-10T16:51:55Z |
| skill_version | knowledge-archaeology v3.2（本地生产版，未修改） |

## 项目基础地图

```
agentevals/
├── src/agentevals/
│   ├── loader/           # trace 输入：jaeger.py / otlp.py / auto.py（格式自检）
│   ├── converter.py      # 格式检测 + 提取器（ADK 优先于 GenAI semconv）
│   ├── genai_converter.py# GenAI semconv → 归一 Invocation（内容/部件两种 schema）
│   ├── extraction.py     # span → LLM/tool 调用提取（TraceFormatExtractor 协议 + AdkExtractor）
│   ├── builtin_metrics.py# 指标 → Google ADK EvalMetric/Criterion 映射 + judge credential 注入
│   ├── runner.py         # 评测编排（双 semaphore 有界并发）
│   ├── run/              # Run 生命周期：service.py / worker.py / fetcher.py / sinks.py / result_builder.py
│   ├── storage/          # models（RunStatus/ResultStatus）+ repos（memory/postgres）+ migrations
│   ├── api/              # FastAPI：routes / runs_routes / otlp_app / otlp_grpc / streaming_routes
│   ├── streaming/        # 增量处理器（WebSocket spans / logs）
│   ├── evaluator/        # resolver.py（EvaluatorSource 注册表）+ venv
│   ├── custom_evaluators.py  # 任意语言子进程/HTTP/Docker 后端 + Runtime 抽象
│   ├── resolvers/        # kubernetes.py / registry.py + get_resolved_credential
│   ├── mcp_server.py     # MCP 会话式评测接口
│   ├── openai_eval_backend.py  # OpenAI Eval API 卸载
│   ├── sdk.py            # AgentEvals context manager / decorator 流式入口
│   └── cli.py            # CLI（EXACT/IN_ORDER/ANY_ORDER 参数）
├── tests/                # 43 文件；实测 798 passed / 24 skipped
├── docs/                 # otel-compatibility / eval-set-format / custom-evaluators / run-history / streaming
├── ui/                   # Web UI（golden eval set 生成）
├── charts/               # Helm chart
├── examples/ samples/
├── pyproject.toml / uv.lock / flake.nix / Makefile / Dockerfile
```

## 核心模块 / 数据结构 / 状态 / 测试 / 配置 / 权限 / 依赖

| 维度 | 事实 | 证据 |
|---|---|---|
| 主要语言 | Python 3.12+（fastapi/opentelemetry/pydantic） | pyproject.toml |
| 入口 | CLI（`agentevals eval`）+ FastAPI（/api）+ OTLP receiver + MCP server + SDK | cli.py / api/ / sdk.py |
| 核心数据结构 | Trace/Span（归一）、Invocation（conversation 单元）、EvalSet（ADK schema）、Run/RunSpec/Result、ToolTrajectoryCriterion | loader/base.py / config.py / storage/models.py |
| 核心状态 | Run：QUEUED→RUNNING→SUCCEEDED/FAILED/CANCELLED；Result：PASSED/FAILED/ERRORED/SKIPPED | storage/models.py:22-31 |
| 测试体系 | pytest 43 文件；**实测 798 passed / 24 skipped**（需 pytest-asyncio——dev 依赖缺口见 EK） | tests/ + 实测 |
| 主要配置 | EvalParams（evaluators/threshold/concurrency）、trajectory_match_type（EXACT/IN_ORDER/ANY_ORDER）、credential_refs、sinks | config.py |
| 权限/治理 | judge credential fail-closed 注入（credential_refs 逻辑名解析）；sink 工厂注册表；Run 提交幂等（409 spec-mismatch） | builtin_metrics.py:307-411 / sinks.py |
| 外部依赖 | opentelemetry-*（proto/sdk/exporter/instrumentation）、fastapi、pydantic-settings、click、**Google ADK（评估原语复用）**、OpenAI Eval API（可选）、Postgres（可选） | pyproject.toml + builtin_metrics.py |

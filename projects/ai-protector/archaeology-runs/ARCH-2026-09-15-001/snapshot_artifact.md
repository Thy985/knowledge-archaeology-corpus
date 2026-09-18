# Repository Snapshot — ai-protector

## Snapshot Metadata

```yaml
repository: https://github.com/szesnasty/ai-protector.git
commit_sha: 8f57f8533e0feb414348c16e6aa6b2e3621103ad
branch: main
tag: v0.2.8（release-please 生成，commit 标题 "chore(main): release 0.2.8"）
repository_version: 0.2.8（version.txt + pyproject ai-protector-proxy 0.2.8，x-release-please-version）
analysis_timestamp: 2026-09-15T02:10+08:00（clone）/ 2026-09-15T02:4x（evidence 采集）
clone_method: git clone --depth 1（shallow）
skill_version: knowledge-archaeology v3.2（本地生产版）
license: Apache-2.0
```

## Scale

- 350 个 .py 文件 / 34 MB（含 .git）
- apps 分布：`agent-demo` 66 py · `proxy-service` 254 py · `frontend` 0 py（Nuxt）· `reference-chat-target` 12 py · `test-agents` 16 py
- requires-python >= 3.12

## Project Map（基础地图）

```
ai-protector/                      # 仓库根
├── apps/
│   ├── agent-demo/                # Agent 运行时双门 + RBAC（66 py）
│   │   └── src/agent/
│   │       ├── graph.py           # LangGraph 11 节点 agent 编排
│   │       ├── nodes/             # pre_tool_gate(462) / post_tool_gate(435) / input / intent / llm_call / memory / policy / tools / response
│   │       ├── rbac/              # models.py（ToolDefinition/Permission/RoleConfig）+ service.py
│   │       ├── security/          # message_builder.py
│   │       ├── tools/registry.py  # 工具注册表
│   │       └── trace/             # accumulator.py 观测
│   ├── proxy-service/             # 核心防火墙（254 py）
│   │   ├── src/
│   │   │   ├── main.py            # FastAPI 入口 + lifespan（预加载/seed/stale-run 清理）
│   │   │   ├── config.py          # Settings（policy/mode/harm_ml 开关/阈值）
│   │   │   ├── pipeline/          # LangGraph 9 节点管线
│   │   │   │   ├── graph.py       # StateGraph + route_after_decision（86 行）
│   │   │   │   ├── state.py       # PipelineState（58 行 TypedDict）
│   │   │   │   ├── runner.py      # run_pipeline / pre_llm_pipeline（189 行）
│   │   │   │   ├── nodes/         # parse/rules/intent/scanners(并行)/decision/transform/llm_call/output_filter/logging
│   │   │   │   ├── utils/deobfuscate.py  # A2 反混淆（261 行）
│   │   │   │   └── rails/         # NeMo config.yml
│   │   │   ├── llm/               # providers（LiteLLM）/ streaming / mock_provider
│   │   │   ├── db/                # session / seed（policies+denylist+wizard）
│   │   │   ├── models/            # SQLAlchemy（policy/request/denylist）
│   │   │   ├── red_team/          # Benchmark Hub（scenario/pack/evaluator/run-engine/score）
│   │   │   ├── services/          # denylist / analytics / langfuse_client
│   │   │   └── wizard/            # Agent Wizard（rbac.yaml/config.yaml 生成）
│   │   ├── alembic/               # 迁移
│   │   ├── benchmarks/            # benchmark 脚本
│   │   ├── data/scenarios/        # 358 攻击场景（216 playground + 142 agent）
│   │   └── tests/                 # 91 个测试文件
│   ├── frontend/                  # Nuxt 前端（Security Scan / Wizard / Traces）
│   ├── reference-chat-target/     # 基准目标服务（12 py）
│   └── test-agents/               # LangGraph + 纯 Python 两个测试 agent（16 py）
├── docs/
│   ├── architecture/              # THREAT_MODEL(226) / PROXY_FIREWALL_PIPELINE(277) / AGENT_PIPELINE(254) / ARCHITECTURE
│   ├── BENCHMARKS.md              # 431 行基准矩阵
│   └── assets/
├── infra/                         # 部署
├── scripts/
├── issues/issues.md               # 13 条已知问题（ISS-001~013）
├── .github/workflows/             # ci / benchmark（周更+手动）/ codeql / release（release-please）
├── BENCHMARK.md / BENCHMARK_JAILBREAKBENCH.md
├── Makefile（test / test-cov / test-scenarios）
├── SECURITY.md（私有漏洞上报/0.2.x 支持/威胁模型/自审/Dependabot/CodeQL）
└── version.txt / CHANGELOG.md / mvp-diagram.md
```

## Identification

| 项 | 值 | 证据 |
|---|---|---|
| 主要语言 | Python 3.12+（后端）+ TypeScript（Nuxt 前端） | pyproject requires-python >=3.12 |
| 主要运行入口 | `apps/proxy-service/src/main.py`（FastAPI，:8000）+ `apps/agent-demo`（:8002） | main.py:150 app = FastAPI |
| 核心模块 | pipeline/graph.py（9 节点 LangGraph）+ nodes/*（7 检测层）+ agent/nodes/*（双门）+ rbac | graph.py / nodes/ |
| 核心数据结构 | PipelineState（TypedDict：scan_text/prompt_hash/risk_flags/decision）+ AgentState + RBAC dataclasses | pipeline/state.py + agent/state.py + rbac/models.py |
| 核心状态 | ALLOW/MODIFY/BLOCK 三态决策 + risk_score(0-1) + policy_name(fast/balanced/strict/paranoid) | decision.py |
| 主要测试体系 | pytest：91 测试文件；proxy-service tests/（conftest 需 Postgres+Redis）+ agent-demo tests/ + test_scenario_deterministic.py（358 场景） | Makefile:109-118 |
| 主要配置 | pydantic-settings（config.py：mode/policy/harm_ml_mode/阈值）+ DB policy JSONB + .env | config.py |
| 权限/治理机制 | RBAC（ToolDefinition sensitivity × role inheritance × requires_confirmation）+ pre/post-tool gate + SECURITY.md 治理 | rbac/models.py + SECURITY.md |
| 主要外部依赖 | fastapi/uvicorn/pydantic/sqlalchemy+asyncpg/redis/litellm/llm-guard/presidio/nemoguardrails(+fastembed+dataclasses-json)/langgraph/langfuse/structlog/weasyprint；dev: pytest/pytest-asyncio/ruff/mypy/aiosqlite | pyproject.toml |

## 运行时基础设施需求（测试证据完整性说明）

- CI（ci.yml）用 GitHub Actions services：pgvector/pg16 + Redis；本地无 Docker 时 proxy-service 全量测试不可复现（conftest autouse fixture 强制连 DB）
- 本地可复现：**668 个单元测试通过**（proxy-service 纯逻辑 203 + agent-demo 465），另有 24+23+353 errors = 环境依赖（DB/Redis/服务）非产品失败
- 358 场景 benchmark 数字（99.1%）为 CI 环境数字（S7 项目自述），本地因 DB 依赖不可完整复现

## 声明

本 Snapshot 全部事实直接来自浅克隆仓库内容（8f57f85），未修改目标项目。基准数字（99%/91%/48ms 等）来自 README/docs/BENCHMARKS.md 项目自述（S7），非本次本地实测。

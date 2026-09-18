# 01 — Project Layer：项目地图（L0 事实）

> 全部事实来自浅克隆 HEAD 8f57f85（v0.2.8）。证据格式 = 文件:行 / 文件（如 `graph.py:22`）。

## 1. 定位与形态

- AI agent 安全运行时：proxy（:8000）+ agent 运行时（:8002）+ 前端（:3000）+ benchmark 目标（reference-chat-target）+ 两个测试 agent（LangGraph / 纯 Python）— `apps/README.md`、`docker-compose` 拓扑
- Apache-2.0 许可；release-please 自动版本管理（0.2.8，`version.txt` = `0.2.8`，`CHANGELOG.md` 0.2.0→0.2.8 覆盖 2026-03-31→07-18）

## 2. 运行入口

| 入口 | 路径 | 说明 |
|---|---|---|
| Proxy API | `apps/proxy-service/src/main.py:150` | FastAPI app，router：chat/chat_direct/models/wizard/policies/requests/rules/scan/analytics |
| 生命周期 | `main.py:34 lifespan` | setup_logging → create_all（dev）→ seed_policies/seed_denylist/seed_wizard → 清理 stale benchmark runs → 清理过期 auth secrets → **后台预加载 ML 模型**（消除 ~50s 冷启动，ISS-002 修复） |
| Agent API | `apps/agent-demo/src/agent/graph.py` | LangGraph 11 节点：input→intent→policy_check→tool_router→(pre_tool_gate→tool_executor→post_tool_gate)∥llm_call→response |
| Makefile | `make demo` / `make test` / `make test-scenarios` | 全栈 / 测试 / 358 场景 |

## 3. 核心模块

### 3.1 proxy-service（254 py）—— 内容防护
- `pipeline/graph.py:22 build_pipeline`：9 节点 StateGraph，`route_after_decision`（`graph.py:18`）按 decision 路由 BLOCK→logging / MODIFY→transform→llm_call→output_filter→logging / ALLOW→llm_call→output_filter→logging
- `pipeline/nodes/*`：7 检测层每层一个 node（rules/intent/scanners/decision 主链；llm_call/output_filter/logging 输出侧）
- `pipeline/runner.py`：run_pipeline + **pre_llm_pipeline**（streaming/scan 路径先拿决策再调用，`runner.py:143`）
- `red_team/`：Benchmark Hub（scenario schema / pack loader / evaluator engine / score calculator / run engine / SSE progress / persistence）
- `wizard/`：Agent Wizard（7 步生成 rbac.yaml/config.yaml/snippet）

### 3.2 agent-demo（66 py）—— 动作防护
- `agent/nodes/pre_tool_gate.py`：5 项检查（RBAC scope=read / args Pydantic schema+注入扫描 / 上下文风险 / session budget / requires_confirmation）
- `agent/nodes/post_tool_gate.py`：PII 脱敏 + secrets 扫描 + **间接注入检测**（工具输出的注入 pattern）+ 输出大小限制
- `agent/rbac/models.py`：ToolDefinition（category×sensitivity×requires_confirmation×rate_limit）/ ToolPermission / RoleConfig（inherits 角色继承）/ PermissionResult
- `agent/graph.py`：tool_router 产出 tool_plan → pre_tool_gate → confirmation_response（需确认时暂停）→ tool_executor → post_tool_gate

## 4. 核心数据结构

| 结构 | 位置 | 关键字段 |
|---|---|---|
| PipelineState | `pipeline/state.py`（TypedDict total=False） | request_id / policy_name / policy_config / user_message / **scan_text（LLM 永不见）** / prompt_hash(SHA-256) / risk_flags / risk_score / decision[ALLOW,MODIFY,BLOCK] / modified_messages / sanitized_messages / errors[] / node_timings |
| AgentState | `agent/state.py` | role / tool_plan / pending_confirmation / limit_exceeded / session budget |
| RBAC dataclasses | `agent/rbac/models.py` | 上述 4 类 frozen dataclass |

## 5. 配置体系

- `src/config.py`（pydantic-settings，.env）：mode(demo/real) / default_policy(balanced) / **harm_ml_mode(off)** / jailbreak_ml_model+threshold(0.85) / harm_ml_model(granite-guardian-3.0-2b)+threshold(0.8) / presidio threshold(0.4)+spacy 模型 / LiteLLM providers
- DB policy JSONB：每 policy（fast/balanced/strict/paranoid）一份 config（nodes 列表决定启用哪些 scanner、thresholds 加权）
- `pipeline/rails/config.yml`：NeMo 13 rails（FastEmbed）

## 6. 权限与治理机制

- RBAC：role→tool→scope（默认 read），sensitivity（low..critical）、requires_confirmation、rate_limit；RoleConfig.inherits 角色继承 — `rbac/models.py`
- SECURITY.md：私有漏洞上报（不走 public issue）、0.2.x active support、THREAT_MODEL.md 文档化、adversarial self-review（SSRF/注入/反序列化/ReDoS/authz/供应链）、Dependabot 周扫、CodeQL 每 push+周、依赖审查、无 secrets in code
- `.github/workflows/`：ci（lint+test 需 pgvector/redis services + 模型下载）/ benchmark（周更+手动，发布 badges）/ codeql / release

## 7. 外部依赖（pyproject.toml 钉死版本）

fastapi==0.135.1 · uvicorn==0.41.0 · pydantic==2.12.5 · sqlalchemy[asyncio]==2.0.49 · alembic · asyncpg · redis · litellm==1.82.0 · **llm-guard==0.3.16** · **presidio 2.2.358** · **nemoguardrails==0.20.0** · **dataclasses-json==0.6.7 + fastembed==0.7.4（NeMo 低估依赖修复——pyproject 注释明确记录 CI ModuleNotFoundError 修复史）** · langgraph==1.0.10 · langfuse · structlog · httpx · weasyprint；dev：pytest==9.0.2 / pytest-asyncio / ruff==0.15.9 / mypy / aiosqlite

## 8. 测试体系

- `apps/proxy-service/tests/`（91 文件）+ `apps/agent-demo/tests/`（22 文件）
- **conftest autouse DB fixture**：每测试 dispose engine + create_all + seed（需 Postgres；CI 用 pgvector service）
- `test_scenario_deterministic.py`：358 场景（216 playground + 142 agent）pre-LLM 全管线，6 个 xfail（混淆/多语言）
- 本地实测：**668 passed**（proxy 纯逻辑 203 + agent-demo 465）；DB 依赖类测试本地不可复现

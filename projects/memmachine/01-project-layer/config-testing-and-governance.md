# Project Layer · Configuration, Testing & Governance

## 1. 配置体系

### 1.1 加载与校验

- 配置来源优先级：`MEMORY_CONFIG` 环境变量 → `~/.config/memmachine/cfg.yml` → `./cfg.yml`（EV-008）。
- 读取：`Path.read_text(encoding="utf-8")` → `yaml.safe_load` → 递归把所有 dict key 转小写 → `Configuration(**mapping)`（EV-070）。
- 校验：pydantic 模型 + `model_validator(mode="after")` 自动关闭不完整子系统（EV-023、EV-024）。
- 环境变量替换：只有 `PasswordMixin` / `ApiKeyMixin` / AWS 凭证 mixin 支持 `$ENV` / `${ENV}`；`user`、`db_name` 等字段不做 shell 展开（EV-071）。

### 1.2 顶层配置键（EV-072）

```yaml
logging: ...
episode_store:
  database: <resource-id>
  with_count_cache: true
episodic_memory:
  enabled: false               # 默认
  long_term_memory: {...}
  short_term_memory: {...}
retrieval_agent: {...}
semantic_memory:
  enabled: false               # 默认
resources:
  databases: {...}
  embedders: {...}
  language_models: {...}
  rerankers: {...}
  vector_stores: {...}
session_manager:
  database: <resource-id>
  instance_cache_size: 0
prompt: {...}
server: {...}
```

### 1.3 默认关闭与依赖门控

| 子系统 | 默认 | 关闭/保持条件 | 证据 |
|---|---|---|---|
| `episodic_memory` | `enabled=False` | `_initialize_default_episodic_configuration` 显式设 false；LTM/STM 均关时整体关 | EV-019 |
| LTM declarative | 不启用，除非配置 | 必须同时有 `embedder` + `reranker` + `vector_graph_store`；任一缺失 → warning 并关闭 LTM | EV-020 |
| LTM event | 不启用，除非配置 | 必须同时有 `vector_store` + `segment_store`；任一缺失 → 关闭 LTM | EV-021 |
| `short_term_memory` | 不启用，除非配置 | 默认 `llm_model="gpt-4.1"`（注意：默认值不等于已配置资源） | EV-025 |
| `semantic_memory` | `enabled=False` | `semantic_memory:` 为 null/{} 也视为关闭 | EV-023 |
| OpenAI 资源 | — | embedder/LM 的 `api_key` 为空且无 `base_url` → 自动关闭对应子系统并 warning | EV-024 |

> **部署含义**：`docker compose up` 成功、`/health` 返回 200，都不代表记忆已启用。必须检查 `/config`（若开启 config API）或启动日志中的 auto-disable warning。

### 1.4 样例拓扑（`sample_configs/configuration.event.yml`，EV-073）

| 项 | 值 |
|---|---|
| LTM backend | `event` |
| embedder | `openai_embedder`（`text-embedding-3-small`，1536 维） |
| reranker | `my_reranker_id`（`rrf-hybrid`：identity + BM25） |
| vector_store | `event_vector_store`（Qdrant） |
| segment_store | `profile_storage`（Postgres） |
| STM | `llm_model=openai_model`，`message_capacity=500` |
| semantic | `llm_model=openai_model`，`embedding_model=openai_embedder`，`database/config_database=profile_storage` |
| session cache | `instance_cache_size=0` |
| LLM | `openai-responses`，`gpt-6-luna`（样例值，不代表真实可用模型） |

### 1.5 运行时配置 API（EV-075）

- 仅 `MEMMACHINE_CONFIG_API=1` 时挂载 `/config`。
- 可读：整体配置、资源状态、episodic/semantic 配置。
- 可写：`PUT /config/memory`、`PUT /config/memory/episodic`、`PUT /config/memory/long-term`、`PUT /config/memory/short-term`、`PUT /config/memory/semantic`。
- 可增删：embedder、language model；可重试失败的 embedder/LM/reranker。
- 变更会 `save_config()` 持久化回原配置文件（若存在路径）；但**不会自动回滚**，也不会重建已缓存的 `MemMachine` 实例（部分改动需要重连/重启才生效，见候选 C-04）。

## 2. 测试体系

### 2.1 规模

| 范围 | 测试文件 | 测试函数 |
|---|---:|---:|
| `packages/server/server_tests` | 114 | 1,856 |
| `packages/client/client_tests` | 7 | 13 |
| 合计 | **121** | **2,191** |

（`find … -name 'test_*.py'` + `grep -rhE '^\s*(async )?def test_'`；静态计数，未 pytest collect；不含 `evaluation/`、`examples/`、`integrations/`）

### 2.2 覆盖域（按目录）

| 域 | 代表路径 | 说明 |
|---|---|---|
| API v2 | `server_tests/.../server/api_v2/test_router.py`、`test_service.py`、`test_mcp.py` | 路由、服务适配、MCP |
| 配置 | `common/configuration/test_configuration.py` 等 9 个文件 | 默认值、auto-disable、provider 配置 |
| episode store | `common/episode_store/*` | UUID 契约、迁移、count cache |
| session manager | `common/session_manager/test_session_manager.py` | 建/校验/状态/迁移 |
| vector store | `common/vector_store/test_{sqlite,sqlite_vec,qdrant,milvus,usearch,hnswlib}*` | 多后端一致性 |
| vector graph store | `common/vector_graph_store/test_{neo4j,nebula}*` | 图查询/批量写 |
| semantic storage | `semantic_memory/storage/test_*` | Neo4j / PGVector / VectorStore |
| episodic | `episodic_memory/{short_term,long_term,event_memory,declarative_memory}/*` | 摘要、阈值、上下文扩展、写入顺序 |
| retrieval agent | `retrieval_agent/test_*` | query 分解、工具调用 |
| client | `client_tests/test_memory.py`、`test_integration_complete.py` | SDK + 端到端 |

### 2.3 测试揭示的关键契约（只列有源码断言者）

1. **score threshold 方向**：
   - cosine：阈值高于 1.0 应丢弃全部（`test_score_threshold_drops_low_scores_under_cosine`）。
   - euclidean 且无 reranker：阈值应丢弃“距离更大”的远匹配（`test_score_threshold_not_inverted_under_euclidean_no_reranker`）。
   - `None`：euclidean 下必须保留全部，回归旧的 `-inf` sentinel（`test_score_threshold_none_keeps_all_results_under_euclidean`）。
   - 证据：EV-060、EV-061、EV-062。
2. **event 上下文扩展**：`test_expand_context_zero_returns_matches_in_score_order`、`test_unify_first_window_keeps_the_score`（EV-031）。
3. **API 身份字段**：测试请求体固定带 `org_id=test_org`、`project_id=test_proj`；并验证含 `/` 的 org_id 被拒绝（EV-083）。

> **证据上限**：以上测试文件只被阅读，**未执行**。因此本 Package 不主张 S4（Test Validated），只主张 S3（Implemented / test-specified）。

### 2.4 CI 清单（EV-078）

| Workflow | 作用 |
|---|---|
| `lint.yml` | `ruff check` + `ruff format --check` + `ty check`（server 3.12/3.14；common/client 3.10/3.14） |
| `pytest.yml` | 单元测试 |
| `pytest-integration.yml` | testcontainers 集成测试 |
| `test-server-package.yml` / `test-python-client-package.yml` / `ts-client-test.yml` | 分包测试 |
| `installation-test.yml` | 安装路径（含 Ollama 本地模型） |
| `docker-image.yml` | CPU/GPU 镜像构建与发布 |
| `security-audit.yml` / `sbom.yml` | 依赖审计 / SBOM |
| `docs-checks.yml` / `generate-openapi.yml` / `sync-platform-openapi.yml` | 文档与 OpenAPI 一致性 |
| `check-uvlockfile.yml` | `uv.lock` 与 `pyproject` 一致 |
| `build-python.yml` / `pypi-publish.yml` / `ts-client-publish.yml` | 构建与发布 |

## 3. 权限、租户与治理

### 3.1 运行时权限（代码事实）

- **REST 无认证**：`api_v2/router.py` 的端点只 `Depends(get_memmachine)`；没有 OAuth/JWT/API-key 依赖（EV-012）。
- **身份来自请求体**：`AddMemoriesSpec` / `SearchMemoriesSpec` 等继承 `_WithOrgAndProj`，`org_id` / `project_id` 是普通字符串字段（EV-011、EV-012）。
- **MCP 身份来自 header**：`ParamsContextMiddleware` 读取 `org-id` / `proj-id` / `user-id`，写入 `ContextVar`；缺省 `""`（EV-013）。
- **租户隔离机制**：`session_key = f"{org_id}/{project_id}"`；event backend 每 session 一个 collection + partition；semantic 用 set_id（EV-015、EV-044）。
- **配置 API**：只有显式开启 `MEMMACHINE_CONFIG_API` 才存在；它同样没有认证依赖（EV-075）。
- **出站凭证**：provider 的 `api_key` 由配置/env 提供，`SecretStr` 包装；日志不打印明文（EV-071）。

> **边界**：以上说明“服务自身不做认证”，不等于“部署可被任意访问”。若置于公网且无网关鉴权，任何知道 `org_id/project_id` 的调用方都可读写该命名空间——这是部署方必须确认的边界（KO-05、C-03）。

### 3.2 项目治理（文档声明）

- `GOVERNANCE.md`：User → Contributor → Maintainer（write access）→ Technical Council；**Lazy Consensus**；提案投票 72 小时；强反对 → 公开讨论；无法共识由 Project Leads 决定（EV-079）。
- `SECURITY.md`：仅 GitHub private vulnerability reporting；2 个工作日确认；只支持最新 major 版本（EV-080）。
- `CONTRIBUTING.md` / `CONTRIBUTING-CORE.md` / `CONTRIBUTING-DOCS.md`：分层贡献规则。
- `STYLE_GUIDE.md` + `AGENTS.md`：代码风格、`uv`、`ruff`、`ty`、测试命令。
- `CODE_OF_CONDUCT.md`：行为准则。

### 3.3 治理与运行时的关系

- **文档治理 ≠ 运行时强制**：`GOVERNANCE.md` 约束的是代码合并流程；它不提供运行时审批、审计或策略回滚。
- **唯一的“策略流”是配置门控**：启用/关闭记忆、切换 backend、增删资源；没有“策略提案→审批→固化→执行”的运行时闭环（详见 Flow Atlas 的 Policy Flow）。
- **删除是后台最终一致**：session 删除先改状态，再由 worker 清理；没有面向用户的“删除完成”同步确认（EV-042、EV-043）。

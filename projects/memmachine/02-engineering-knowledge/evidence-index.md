# Evidence Index — EV-001 ~ EV-092

> 证据强度：S2 = 设计文档/提案；S3 = 已实现（静态代码可核）；S4 = 自动化测试已运行（本轮未执行，不使用）；S5 = 真实运行（本轮未执行）。
> 所有 `file:line` 均相对仓库根 `archaeology-jobs/ARCH-2026-11-14-001/repo/`。

| ID | Type | Source | Claim | Strength |
|---|---|---|---|---|
| EV-001 | SNAPSHOT | `git rev-parse HEAD` | 本轮固定 commit `377510d436dc81c3651502f2d5e36ff245650ce7`，branch `main`，commit time `2026-11-02T11:35:15-08:00` | S3 |
| EV-002 | LICENSE | `LICENSE` | Apache-2.0 | S3 |
| EV-003 | SCALE | `find . -not -path './.git/*'` | 876 文件；496 个 `.py`，151,515 行；`packages/` Python 118,114 行 | S3 |
| EV-004 | LAYOUT | 仓库根 `ls` | `packages/{client,common,meta,server,skills,ts-client}`、`docs/`、`design/`、`deployments/helm/`、`integrations/`、`evaluation/`、`examples/`、`tools/` | S3 |
| EV-005 | DEP | `packages/server/pyproject.toml` | Python `>=3.12`；核心依赖 fastapi/fastmcp/sqlalchemy/asyncpg/aiosqlite/alembic/pydantic/neo4j/pgvector/sqlite-vec/usearch/rank-bm25/nltk/openai/cohere/boto3/instructor/prometheus-client/uvicorn | S3 |
| EV-006 | ENTRY | `packages/server/pyproject.toml` `[project.scripts]` | console scripts：`memmachine-server`、`memmachine-nltk-setup`、`memmachine-mcp-stdio`、`memmachine-mcp-http`、`memmachine-configure` | S3 |
| EV-007 | ENTRY | `packages/server/src/memmachine_server/server/app.py:1-276` | `MemMachineAPI(FastAPI)`；`--stdio` / `--version` / `--with-config-api`；`mount("/mcp", mcp_app)`；`start_http` 调 `uvicorn.run` | S3 |
| EV-008 | CONFIG | `packages/server/src/memmachine_server/server/app.py`（config 解析段） | 配置查找顺序 `MEMORY_CONFIG` → `~/.config/memmachine/cfg.yml` → `./cfg.yml`；找不到抛 `FileNotFoundError` | S3 |
| EV-009 | RUNTIME | `server/app.py`（`start_http`）、`main/memmachine.py` | `MEMMACHINE_WORKERS` 默认 1；多 worker 时自动建 `PROMETHEUS_MULTIPROC_DIR` 并清 stale `.db` | S3 |
| EV-010 | PROTOCOL | `server/app.py` | `/mcp` 挂载 fastmcp app；MCP 与 REST 共享同一 `app.state.mem_machine` 实例 | S3 |
| EV-011 | API | `server/api_v2/router.py:162-1140` | 业务端点均为 `POST`（health 为 GET）；路径含 `/projects`、`/memories`、`/features`、`/semantic/*` | S3 |
| EV-012 | AUTH | `server/api_v2/router.py` 全文 | 无 `Security` / OAuth / API-key 依赖；仅 `Depends(get_memmachine)`；org/project 来自请求体 | S3 |
| EV-013 | AUTH | `server/api_v2/mcp.py:257-300` | `ParamsContextMiddleware` 从 HTTP header `org-id` / `proj-id` / `user-id` 取身份写入 `ContextVar`；缺失为 `""`；无签名/令牌校验 | S3 |
| EV-014 | API | `server/api_v2/mcp.py:415,478,537` | MCP 仅注册 3 个 tool：`add_memory` / `search_memory` / `delete_memory` | S3 |
| EV-015 | DATA | `server/api_v2/service.py:39-46` | `_SessionData.session_key = f"{org_id}/{project_id}"` | S3 |
| EV-016 | WRITE | `main/memmachine.py:778-879` | `add_episodes` 并行三写：episode_storage、episodic_memory、semantic_session_manager；`asyncio.gather(return_exceptions=True)`；随后 `_raise_add_episode_errors` | S3 |
| EV-017 | FAILURE | `main/memmachine.py:867` | 源码 TODO：`Track write intent so partial failures across these parallel backends can be detected and repaired`（#1738） | S3 |
| EV-018 | WRITE | `main/memmachine.py:753-777` | `_should_dispatch_to_semantic_memory` 仅在 `MemoryType.Semantic` 且 `semantic_memory.enabled` 时派发 | S3 |
| EV-019 | WRITE | `episodic_memory/episodic_memory.py:209-244` | `add_memory_episodes` 生成 filterable_metadata 后 `asyncio.gather(short_term.add_episodes, long_term.add_episodes)`；**无 `return_exceptions`** | S3 |
| EV-020 | CONFIG | `main/memmachine.py:103-175` | `_initialize_default_episodic_configuration` 默认 `episodic_memory.enabled=False`，LTM/STM 均关 | S3 |
| EV-021 | CONFIG | `main/memmachine.py:175-238` | declarative 后端缺 embedder/reranker/vector_graph_store 任一 → `_disable_long_term_memory` 仅 warning；event 后端缺 vector_store/segment_store 任一 → 关闭 | S3 |
| EV-022 | CONFIG | `main/memmachine.py:255-266` | `_resolve_stm_defaults` 默认 `llm_model="gpt-4.1"` | S3 |
| EV-023 | CONFIG | `common/configuration/__init__.py:396-540` | `Configuration` 含 9 个顶层段；`semantic_memory` 缺省 `enabled=False` | S3 |
| EV-024 | CONFIG | `common/configuration/__init__.py:446-540` | `_maybe_disable_semantic_memory` / `_maybe_disable_episodic_memory`：OpenAI 凭据空且无 base_url 时自动关闭子系统 | S3 |
| EV-025 | CONFIG | `common/configuration/__init__.py:628-660` | `load_yml_file`：`yaml.safe_load` → 递归 key 小写 → `Configuration(**mapping)`；不做任意 env 展开 | S3 |
| EV-026 | WRITE | `episodic_memory/declarative_memory/declarative_memory.py:99-212` | 按 `(timestamp, uid)` 排序；写 `Episode_<session>` + `Derivative_<session>` 节点；`asyncio.gather` 后写 `DERIVED_FROM_<session>` 边 | S3 |
| EV-027 | WRITE | `declarative_memory.py:213-273` | `_derive_derivatives`：MESSAGE 默认整条加 `"{source}: "` 前缀；`message_sentence_chunking=True` 按句拆分；TEXT 整段一条；不支持类型 warning 返回空 | S3 |
| EV-028 | READ | `declarative_memory.py:313-500` | derivative 向量搜索 `limit=min(5*max_num_episodes,200)` → `DERIVED_FROM` 反查 → 邻域扩展 → reranker → 去重合并 | S3 |
| EV-029 | DELETE | `declarative_memory.py:595-614` | 删除时并行删节点（`asyncio.gather(*delete_nodes_tasks)`），无事务 | S3 |
| EV-030 | WRITE | `episodic_memory/event_memory/event_memory.py:223-317` | 五阶段：segment（236）→ derive（247）→ embed（271）→ add_segments（274）→ vector upsert（292）；先 segment 后 vector | S3 |
| EV-031 | READ | `event_memory.py:398-535` | `embedder.search_embed` → vector query → `get_segment_contexts`（前 1/3 后 2/3）→ reranker/embedding 分数 → 按 `higher_is_better` 排序 | S3 |
| EV-032 | DELETE | `event_memory.py:689-725` | 先查 event→segments、segments→derivatives；**先删 vector store，再删 segment store** | S3 |
| EV-033 | FAILURE | `event_memory.py:390-410` | event index 命中的 episode 在 EpisodeStorage 缺失 → warning 并丢弃；注释称 index/storage drift（likely partial failures） | S3 |
| EV-034 | STATE | `episodic_memory/short_term_memory/short_term_memory.py:105-243` | `deque` 存近期 episode；`add_episodes` 写锁内 extend；超容量 `_do_evict`：先弹已摘要旧条目，仍满则整批 summarize 并重置计数 | S3 |
| EV-035 | STATE | `short_term_memory.py:396-444` | `get_short_term_memory_context` 读锁内等待 consolidator，按 summary + 最近 episode 反向装入，受 `max_message_length`/`limit` 限制 | S3 |
| EV-036 | STATE | `short_term_memory.py:468-577` | `ShortTermMemoryConsolidator` 后台 worker + 动态批处理；`summarize()` 入 buffer 起 task；异常只 `logger.exception` | S3 |
| EV-037 | FAILURE | `short_term_memory.py:613-679` | `_create_summary` 上下文超限递归二分（单条按字符中点拆半）；其它异常记录并返回旧 summary | S3 |
| EV-038 | DATA | `short_term_memory.py:148-156` | `max_summary_length_words = ceil(max_message_len/2/8 /100)*100` | S3 |
| EV-039 | READ | `episodic_memory/long_term_memory/long_term_memory.py:109-240` | `DeclarativeBackendParams` / `EventBackendParams` 必填字段；`LongTermMemory` 按 `params.backend` 分派 | S3 |
| EV-040 | API | `episodic_memory/episodic_memory.py:308-445` | `QueryMode={BOTH,LONG_TERM_ONLY,SHORT_TERM_ONLY}`；BOTH 并发 STM/LTM，短时优先去重 | S3 |
| EV-041 | STATE | `common/session_manager/session_data_manager.py:63-77` | `SessionStatus = Active("active") | Deleted("delete")`；`SessionInfo` 含 configuration/description/user_metadata/episode_memory_conf/status | S3 |
| EV-042 | STATE | `session_data_manager.py:233-330` | `delete_session` 只改 status 为 `delete`（逻辑删除），不物理删数据 | S3 |
| EV-043 | STATE | `main/memmachine.py:350-440` | `_delete_session_worker` 消费 `_deletion_queue`；SQL lease `session-deletion:<key>`；`asyncio.TaskGroup` 并发删 episode/episodic/semantic；全部成功后 `manager.delete_session` | S3 |
| EV-044 | STATE | `episodic_memory/episodic_memory.py` 分区删除段 | `drop_session_partition` 删除 VectorStore collection + SegmentStore partition；实例随后不可复用 | S3 |
| EV-045 | WRITE | `semantic_memory/semantic_memory.py:234-306` | `add_messages`/`add_message_to_sets` 用 `gather(return_exceptions=True)` 后统一抛错 | S3 |
| EV-046 | WRITE | `semantic_memory/semantic_ingestion.py:125-298` | 每 set 拉最多 5 条未 ingest message；按 category 并发；context 超限跳过并标 ingested；其它异常在非 loud 模式 continue | S3 |
| EV-047 | WRITE | `semantic_ingestion.py:229` | 每 set/category 取已有 features `page_size=max_features_per_update=50` | S3 |
| EV-048 | WRITE | `semantic_ingestion.py:299-347` | `_apply_commands` 只实现 ADD/DELETE；未知命令只 `logger.error`；ADD 时 value embedding + `add_feature`；有 citation_id 则 `add_citations([message.uid])` | S3 |
| EV-049 | WRITE | `semantic_ingestion.py:83` | `consolidated_threshold=20`；达到阈值才触发 `_deduplicate_features` | S3 |
| EV-050 | FAILURE | `semantic_ingestion.py:466-499` | consolidation：LLM 返回 `keep_memories` + `consolidated_memories`；**先 `delete_features(memories_to_delete)`** | S3 |
| EV-051 | FAILURE | `semantic_ingestion.py:516-542` | 再并发 `_add_feature(feature)`；删除与新增非事务 | S3 |
| EV-052 | PROVENANCE | `semantic_ingestion.py:537` | 把所有被删 feature 的 citations 合并后挂到**每个**新 feature：`add_citations(f_id, citation_ids)` | S3 |
| EV-053 | READ | `semantic_memory/semantic_memory.py:198-233` | 多 set_id 分别 embed 后并行 `_set_id_search`，`merge_async_iterators` 合并 | S3 |
| EV-054 | STATE | `semantic_memory/cluster_splitter.py` | `NoOpClusterSplitter` / `RerankerClusterSplitter`；按 reranker 相邻分数 + continuity signals 切分；`split` 有 replay/idempotency | S3 |
| EV-055 | API | `packages/common/src/memmachine_common/api/spec.py:564-630` | `SearchMemoriesSpec.top_k` 默认 10 且 `ge=1`；`score_threshold` 默认 `None` | S3 |
| EV-056 | API | `server/api_v2/service.py:100-135` | `_search_target_memories` 在 `score_threshold is None` 时显式传 `-float("inf")` | S3 |
| EV-057 | CONTRACT | `long_term_memory.py:254-287` | `search_scored` docstring：`None` 保留全部；higher-is-better 低于阈值丢弃，lower-is-better 高于阈值丢弃；注释称旧 `-inf` sentinel 在 euclidean 下会静默反转 | S3 |
| EV-058 | IMPL | `long_term_memory.py` `_search_scored_declarative`（约 296-315） | 实现无条件 `score >= score_threshold`，与 docstring 的“方向自适应”不一致；declarative 强制 reranker，当前实际路径仍为 higher-is-better | S3 |
| EV-059 | IMPL | `long_term_memory.py:462-475` | `_score_passes_threshold` 按 `self._score_higher_is_better` 判断方向（event 路径） | S3 |
| EV-060 | TEST | `server_tests/.../long_term_memory/test_event_backend_wiring.py:536` | `test_score_threshold_drops_low_scores_under_cosine`：阈值 2.0 丢弃全部 | S3 |
| EV-061 | TEST | 同上 `:557` | `test_score_threshold_not_inverted_under_euclidean_no_reranker`：euclidean 无 reranker 时按方向过滤 | S3 |
| EV-062 | TEST | 同上 `:603` | `test_score_threshold_none_keeps_all_results_under_euclidean`：默认 None 不得丢弃全部 | S3 |
| EV-063 | READ | `main/memmachine.py:880-937,995-1032` | `_search_episodic_memory` 在 retrieval agent 存在且 LTM 非空时走 agent；`QueryPolicy(token_cost=10, time_cost=10, accuracy_score=10, confidence_score=10, max_attempts=3, max_return_len=10000)` 全硬编码 | S3 |
| EV-064 | READ | `main/memmachine.py:1064-1084` | retrieval agent 返回的 LTM episodes 全部赋 `score=1.0`；仅按 UID 与 short-term 去重 | S3 |
| EV-065 | READ | `main/memmachine.py:1085-1162` | `query_search` 用 `create_task` 并行 episodic/semantic；返回 `SearchResponse` | S3 |
| EV-066 | READ | `episodic_memory/episodic_memory.py:353-478` | `query_memory` 默认 `limit=20`；BOTH 并发 STM/LTM；短时优先去重；LTM 按分数排序 | S3 |
| EV-067 | READ | `episodic_memory.py:479+` | `formalize_query_with_context` 把 summary+episodes 按 `(created_at, uid)` 排序后格式化进 query | S3 |
| EV-068 | DATA | `common/api/spec.py` `SearchMemoriesSpec` | `top_k` 默认 10；`expand_context` 默认 0；`filter` 默认空串；`types` 可选 | S3 |
| EV-069 | API | `server/api_v2/router.py` 校验段 | 请求体字段校验；`org_id` 含 `/` 等非法字符被拒（`Invalid request payload: org_id`） | S3 |
| EV-070 | CONFIG | `common/configuration/__init__.py:628-660` | YAML 递归 key 小写；pydantic 校验；根必须是 mapping | S3 |
| EV-071 | SECRET | `common/configuration/mixin_confs.py:57-190` | `$ENV` / `${ENV}` 只对 password / API key / AWS 凭证解析；未解析的 `$VAR` 保留原样 | S3 |
| EV-072 | CONFIG | `common/configuration/__init__.py:396-540` | 顶层段：logging/episode_store/episodic_memory/retrieval_agent/semantic_memory/resources/session_manager/prompt/server | S3 |
| EV-073 | CONFIG | `sample_configs/configuration.event.yml` | event 拓扑：Qdrant vector_store + Postgres segment_store；STM `message_capacity=500`；session cache 0；reranker `rrf-hybrid` | S3 |
| EV-074 | DEPLOY | `docker-compose.yml` | 服务：postgres(`pgvector/pgvector:pg18`)、neo4j(`neo4j:5.23-community`, profile declarative)、qdrant(`qdrant/qdrant:v1.19.1`, profile event)、memmachine、docs | S3 |
| EV-075 | API | `server/api_v2/config_router.py:40-115,256-399` | `/config` 仅在 `MEMMACHINE_CONFIG_API` 开启时挂载；GET 配置/资源；PUT 各层配置；增删/重试资源 | S3 |
| EV-076 | PERSIST | `server/api_v2/config_service.py:387-430` | `_persist_config` 仅在 `config_file_path` 存在时 `save_config()`；失败只 warning | S3 |
| EV-077 | TEST | `packages/server/server_tests` | 114 个 `test_*.py`、1,856 个 `def test_`（静态计数）；全仓 121 个测试文件、2,191 个测试函数 | S3 |
| EV-078 | CI | `.github/workflows/*.yml`（19 个） | lint/pytest/integration/installation/docker/security/sbom/docs/openapi/build/publish 等 | S3 |
| EV-079 | GOVERNANCE | `GOVERNANCE.md` | User/Contributor/Maintainer/Technical Council；Lazy Consensus；投票 72h；强反对公开讨论；无法共识由 Project Leads 决定 | S3 |
| EV-080 | SECURITY | `SECURITY.md` | 仅 GitHub private vulnerability reporting；2 个工作日确认；只支持最新 major | S3 |
| EV-081 | DOC | `README.md:32-34,119-121` | README 称 Episodic 为 graph-based、Profile 存 SQL、Flexible Storage 为 Neo4j + SQL | S2 |
| EV-082 | DOC | `docs/core_concepts/agentic_memory.mdx:19-47` | 文档描述 Working/Persistent/Profile 三层；原始数据被“智能处理”后同时写入 Persistent 与 Profile；Context Retrieval Agent 可 RL 微调 | S2 |
| EV-083 | TEST | `server_tests/.../server/api_v2/test_router.py:50-146` | 测试请求体固定 `org_id=test_org`、`project_id=test_proj`；含 `/` 的 org_id 返回校验错误 | S3 |
| EV-084 | STORAGE | `semantic_memory/storage/*` | semantic 支持 Neo4j / PGVector / VectorStore 三类 storage 实现 | S3 |
| EV-085 | STORAGE | `common/vector_store/*` | vector store 支持 sqlite、sqlite-vec、qdrant、milvus、usearch、hnswlib、in-memory | S3 |
| EV-086 | MIGRATION | `alembic.ini` + `packages/server/src/memmachine_server/common/episode_store/migrations` | Alembic 管理 episode store / session 表迁移 | S3 |
| EV-087 | DELETE | `server/api_v2/service.py:83-96` | `_delete_memories` 对 episodic/semantic 用 `asyncio.gather`，**未 `return_exceptions`** | S3 |
| EV-088 | OBSERVE | `server/app.py` 中间件段 | 请求访问日志 + metrics 中间件；Prometheus counter/histogram | S3 |
| EV-089 | OBSERVE | `server/app.py` signal 段 | `SIGUSR1` 触发 traceback dump（用于卡死诊断） | S3 |
| EV-090 | SEMANTIC | `semantic_memory/semantic_ingestion.py` prompt/类别段 | semantic categories 驱动 Profile/Factual 等抽取；`semantic_categories` 可配置 | S3 |
| EV-092 | STATE | `episodic_memory/long_term_memory/long_term_memory.py:417-440`；`episodic_memory/event_memory/event_memory.py` | `drop_session_partition` 删除 vector collection + segment partition 后把 `EventMemory` 句柄置 None；LTM 实例不可复用，需重建 | S3 |
| EV-091 | STORAGE | `semantic_memory/storage/sqlalchemy_pgvector_semantic.py:67-79,387-400,553-564`；`neo4j_semantic_storage.py:448-458` | PG `citations.feature_id` 有 `ForeignKey("feature.id", ondelete="CASCADE")`，`delete_features` 删 Feature 时由 DB 级联删关联行；`delete_history` 按 history_id 清 citation。Neo4j `DETACH DELETE f` 删节点及属性（citations 为节点属性列表） | S3 |

## 统计

- 总数：92
- S3：90；S2：2（EV-081、EV-082）
- S4/S5：0（本轮未执行测试/服务）
- 覆盖：架构/配置/写读删/状态/测试/治理/部署/可观测性

## 证据使用纪律

- `Fact` 只用于可直接从代码/文档读出的事实（S3/S2 均可，但 S2 只能证明“文档声称”）。
- `Observation` 用于跨 2+ 证据的结构归纳。
- `Hypothesis` 只进 `05_candidates/`，不得写成 EK 或 KO 的已验证结论。

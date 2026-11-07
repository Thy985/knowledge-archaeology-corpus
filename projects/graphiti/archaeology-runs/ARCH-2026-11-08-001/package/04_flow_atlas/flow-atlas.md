# Flow Atlas · 七类流

> 每条 Flow Edge 标注来源符号/文件与条件。无法回溯到代码的边不写入。

## 1. Control Flow（控制流）

| # | From | To | 条件/触发 | 证据 |
|---|---|---|---|---|
| C-01 | REST `POST /messages` | AsyncWorker.queue | 立即返回 202，逐条入队 | `ingest.py:51-72` |
| C-02 | AsyncWorker | `Graphiti.add_episode` | 队列消费者串行执行 | `ingest.py:56-72` |
| C-03 | `add_episode` | `_resolve_request_scope` | 每次调用首步，解析 group→driver | `graphiti.py:1334-1428` |
| C-04 | `add_episode` | `retrieve_episodes` | `previous_episode_uuids is None` | `graphiti.py:1450-1458` |
| C-05 | `add_episode` | `extract_nodes` | 上下文就绪后 | `graphiti.py:1477-1484` |
| C-06 | `add_episode` | `resolve_extracted_nodes` | 抽取返回后 | `graphiti.py:1486-1493` |
| C-07 | `add_episode` | `_extract_and_resolve_edges` | 节点 uuid_map 就绪 | `graphiti.py:1495-1510` |
| C-08 | `add_episode` | `extract_attributes_from_nodes` | 新边集合就绪（与边解析并行） | `graphiti.py:1516-1530` |
| C-09 | `add_episode` | `_process_episode_data` | hydrate 完成 | `graphiti.py:1528` |
| C-10 | `add_episode` | `update_community` ×N | 仅 `update_communities=True` | `graphiti.py:1532-1545` |
| C-11 | `search` | `edge/node/episode_search` | 按 `SearchConfig` 分派 | `search.py:98-150` |
| C-12 | `search` | `node_distance` 重排 | `center_node_uuid is not None` | `graphiti.py:1955-1960` |

**串行性约束**：C-02 是单消费者队列；库文档要求调用方串行 await（`graphiti.py:1418-1425`）。

## 2. State Flow（状态流）

| # | State From | State To | 条件 | 证据 |
|---|---|---|---|---|
| S-01 | （无） | Episodic 已写入 | add_episode 成功 | `graphiti.py:1458-1475` |
| S-02 | EntityEdge `expired_at=None` | `invalid_at=候选.valid_at`; `expired_at=now` | 候选起始晚于新边起始 | `edge_operations.py:730-747` |
| S-03 | Candidate `invalid_at=None` | `invalid_at=resolved.valid_at` | 候选更早且区间不重叠 | `edge_operations.py:748-786` |
| S-04 | 边 `episodes=[e1,e2]` | `episodes=[e2]`（更新） | 删除 e1 | `graphiti.py:2180-2192` |
| S-05 | 边 `episodes=[e1]` | 删除边 | 删除 e1 后无支持者 | 同上 |
| S-06 | 节点被多 episode 提及 | 保留节点 | `MENTIONS count > 1` | `graphiti.py:2194-2205` |
| S-07 | Saga `last_summarized_*` 旧值 | 新值 | `summarize_saga` 完成 | `nodes.py:907-915`、`graphiti.py:590` |

## 3. Data Flow（数据流）

| # | From | To | 变换 | 证据 |
|---|---|---|---|---|
| D-01 | `episode_body`（原始文本） | `EpisodicNode.content` | 原样保存，不改写 | `nodes.py:321` |
| D-02 | content + previous episodes | `extract_nodes` → EntityNode | LLM 结构化抽取 | `graphiti.py:1477` |
| D-03 | EntityNode.name | embedding → `node_similarity_search` | 向量化后取 top15/≥0.6 | `node_operations.py:585-635` |
| D-04 | 节点 uuid_map + edge 候选 | EntityEdge | 端点 UUID 重映射 | `edge_operations.py:424-648` |
| D-05 | episode + 边 | `fact/valid_at/invalid_at` | LLM 抽事实与时间 | `edge_operations.py:786-833` |
| D-06 | 新边集合 | EntityNode.summary | 区域摘要（只喂新边） | `graphiti.py:1516-1524` |
| D-07 | query | embedding + BM25 | 混合召回 | `search.py:259-341` |
| D-08 | 多路得分 | RRF/MMR/rerank | 融合排序 | `search_config_recipes.py:111-217` |
| D-09 | JSON 长内容 | chunks（含 overlap） | 按元素/字段切分 | `content_chunking.py:215-310` |

## 4. Evidence Flow（证据流）

| # | Artifact | 记录位置 | 用途 | 证据 |
|---|---|---|---|---|
| E-01 | episode 原文 | `EpisodicNode.content` | provenance 根 | `nodes.py:321` |
| E-02 | 事实↔episode | `EntityEdge.episodes[]` | "谁支持这条事实" | `edges.py:271` |
| E-03 | episode↔实体 | `EpisodicEdge`（MENTIONS） | 反向溯源 | `edges.py:143` |
| E-04 | 阶段计数 | OTEL span attributes（node.count/edge.count/invalidated_count） | 可观测 | `graphiti.py:1550-1570` |
| E-05 | token 使用 | `token_tracker` | 成本核算 | `graphiti.py:413` |
| E-06 | LLM 调用链 | `tracer` span（`_trace_phase`） | 调试 | `search.py:81` |

> **证据缺口**：开源侧**没有**"谁写入/谁查询"的审计日志；E-01~E-06 都是工程证据，不是安全审计证据。

## 5. Authority Flow（权威流）

| # | Actor | Authority | 边界 | 证据 |
|---|---|---|---|---|
| A-01 | 任意调用方 | 读/写/删 **任意 group_id** | 无身份校验 | `SECURITY.md` |
| A-02 | 任意调用方 | `clear_graph` 整区清空 | 无权限检查 | `mcp_server.py:1101-1143` |
| A-03 | 应用层 | 决定"内容进哪个 group" | 唯一实际控制点 | `SECURITY.md` |
| A-04 | 同 group 内 episode | 同等信任，可失效旧事实 | 无来源分级 | `SECURITY.md` |
| A-05 | 驱动层 | 按 group 路由到独立 DB | 防止跨库串写（#1676 修复后） | `graphiti.py:1304-1330` |
| A-06 | CLA/发布 | 外部贡献准入 | 仅代码贡献，不涉及运行时 | `cla.yml`、`signatures/` |
| A-07 | SECURITY 声明 | 把缺失能力标为 out-of-scope | 明确拒绝"未鉴权"类报告 | `SECURITY.md` |

## 6. Memory Flow（记忆流）

| # | 层 | 内容 | 更新时机 | 证据 |
|---|---|---|---|---|
| M-01 | Episode（原始） | content | 每次写入 | `nodes.py:318` |
| M-02 | Entity（实体） | name/summary/attributes | 随写增量 hydrate | `graphiti.py:1516` |
| M-03 | Fact（边） | fact + 时间窗 | 随写新增/失效 | `edges.py:263` |
| M-04 | Community（区域） | 实体簇 + summary | 显式 `update_communities`/`build_communities` | `graphiti.py:1532,1867` |
| M-05 | Saga（历史段） | summary + episode 链 | 显式 `summarize_saga` | `graphiti.py:590` |
| M-06 | 检索视图 | 按时间/属性过滤后的子图 | 查询时 | `search_filters.py` |

**压缩链**：Episode(原始) → Entity/Fact(结构化) → Community/Saga(摘要)。三层都可回溯（M-04/M-05 的 summary 由 M-01 派生）。

## 7. Policy Flow（策略流）

| # | 策略 | 决策点 | 执行点 | 违反后果 | 证据 |
|---|---|---|---|---|---|
| P-01 | 边类型签名 | `edge_type_map` 定义 | `enforce_edge_type_signatures` | strict→丢弃；宽松→改名+清属性 | `edge_operations.py:671-714` |
| P-02 | 实体类型白/黑名单 | `entity_types` / `excluded_entity_types` | `extract_nodes` | 被排除类型不建节点 | `graphiti.py:1477`、`validate_excluded_entity_types` |
| P-03 | 检索时间/属性策略 | `SearchFilters` | `search` | 过滤掉不满足的事实 | `graphiti.py:1935` |
| P-04 | 并发上限 | `max_coroutines` | `semaphore_gather` | 限流 | `graphiti.py:1540` |
| P-05 | 服务安全策略 | 无（显式声明缺失） | 无 | 由部署方补 | `SECURITY.md` |
| P-06 | 贡献准入 | CLA | `cla.yml` | 未签则不能合并 | `signatures/version1/cla.json` |
| P-07 | 模型路由策略 | `LLMRuntime` routes | `resolve_model` | 未知 prompt 构造期报错 | `spec/llm-runtime.md`、`llm_runtime.py:147` |

## Flow → KO 交叉校验

| KO | 依赖的 Flow | 是否一致 |
|---|---|---|
| KO-01 | S-02/S-03 + E-02 | ✅ |
| KO-02 | C-03~C-10 | ✅ |
| KO-03 | D-03 + C-06 | ✅ |
| KO-04 | A-05 + C-03 | ✅ |
| KO-05 | A-05 + M-06 | ✅（逻辑/物理映射） |
| KO-06 | A-06/A-07 | ✅ |
| KO-07 | A-01~A-04 + P-05 | ✅ |
| KO-08 | P-01 + A-05 | ✅（能力/策略差异） |

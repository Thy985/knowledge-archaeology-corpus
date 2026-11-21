# Project Layer · Data Model & Lifecycle

> 本文只描述“有哪些数据对象、它们在什么顺序下被写/读/删/驱逐”。对失败模式的判断在 EK/KO，不在此处下结论。

## 1. 对象总表

| 对象 | 归属 | 身份 | 持久化位置 | 证据 |
|---|---|---|---|---|
| `EpisodeEntry` | 写入请求 | `uid`（默认 uuid4） | 经 `MemMachine.add_episodes` 转成 `Episode` | EV-015、EV-016 |
| `Episode` | episode store | `uid` + `session_key` | Postgres/SQLite 表 `episodestore` | EV-015、EV-086 |
| `Segment` | event backend | `uuid` + `event_uuid` + `index` | segment store（`segment_store_sg`） | EV-030、EV-075 |
| `Derivative`（event） | event backend | `uuid` + `segment_uuid` | segment store（`segment_store_dv_ln`）+ vector collection | EV-030、EV-075 |
| `Derivative`（declarative） | declarative backend | 节点 uid | `Derivative_<session>` collection | EV-026 |
| `DERIVED_FROM` edge | declarative backend | 边 uid | `DERIVED_FROM_<session>` relation | EV-026 |
| `SemanticFeature` | semantic memory | `metadata.id` + `set_id` | Neo4j / PGVector / VectorStore | EV-048、EV-084 |
| `SessionInfo` | session manager | `session_key` | `sessions` 表 | EV-041、EV-042 |
| `ShortTermMemoryData` | session manager | `session_key` | `short_term_memory_data` 表 | EV-038、EV-047 |

## 2. 写入路径

### 2.1 顶层 1~3 路条件并行写（所有 backend 共用）

`MemMachine.add_episodes`（EV-016、EV-017、EV-018）：

```
1. 生成 batch_ids = sorted(uuid4() for _ in episode_entries)
2. 补 created_at = now(UTC)（若未提供）
3. 组装任务集（最多 3 路）：
   a. episode_storage.add_episodes(batch)          # 总是
   b. episodic_memory.add_memory_episodes(batch)   # 仅 target_memories 含 Episodic
   c. semantic_session_manager.add_message(...)    # 仅含 Semantic 且 enabled；只登记 history
4. asyncio.gather(..., return_exceptions=True)
5. _raise_add_episode_errors(task_names, results, episode_ids)
```

- 多路**不是事务**；`return_exceptions=True` 只是让已组装的任务都跑完再统一抛错，没有补偿/回滚。
- 源码保留 `# TODO(#1738): Track write intent so partial failures across these parallel backends can be detected and repaired.`（EV-017）。
- `_should_dispatch_to_semantic_memory` 只在 `MemoryType.Semantic` 被请求且 `semantic_memory.enabled` 时执行第 3 路（EV-018）。

### 2.2 EpisodicMemory 内部（STM + LTM）

`EpisodicMemory.add_memory_episodes`（EV-019）：

```
1. 为 metadata 生成 filterable_metadata（仅 PropertyValue）
2. asyncio.gather(
     short_term.add_episodes(...),
     long_term.add_episodes(...)
   )                    # 注意：没有 return_exceptions
3. 任一失败立即抛出；另一侧可能已写入成功
```

- 这与顶层 `return_exceptions=True` 形成**两级并发语义不一致**：顶层等全部完成，内层先失败先抛出。

### 2.3 declarative backend

`DeclarativeMemory.add_episodes`（EV-026、EV-027）：

```
1. 按 (timestamp, uid) 排序
2. 每个 episode 建 Node：
   - 原始 content / source / content_type / user_metadata
   - filterable properties（mangle_property_key）
3. 派生 derivative：
   - MESSAGE：默认整条，加 "{source}: " 前缀
   - message_sentence_chunking=True：按句拆分
   - TEXT：整段一条
4. embedder 为 derivative 生成向量
5. 并行 add_nodes(Episode_<session>) + add_nodes(Derivative_<session>)
6. 再写 DERIVED_FROM_<session> 边
```

- 节点与边**非原子**：若边写失败，会留下无 `DERIVED_FROM` 的孤立 derivative（EV-026）。
- 原文 `Episode` 与派生 `Derivative` 分开存储，检索走 derivative，回填走 episode。

### 2.4 event backend

`EventMemory._encode_events`（EV-030、EV-031）：

```
1. segmenter.segment(event)                    # 事件 → segment
2. deriver.derive(segment)                     # segment → derivative
3. embedder.ingest_embed(derivative_texts)     # derivative → 向量
4. segment_store_partition.add_segments(...)   # 先落 segment / derivative 关系
5. vector_store_collection.upsert(...)         # 后落向量
```

- 写顺序**先 segment store，后 vector store**；vector upsert 失败会留下“无向量的孤儿 segment”（EV-030）。
- 每个阶段都有 metrics/debug timing（EV-030）。

### 2.5 semantic/profile 写入（异步）

`SemanticService.add_message_to_sets` 只把 message 登记到 set（`is_ingested=False`）；真正的抽取由后台 `IngestionService` 完成（EV-045）：

```
1. 每 set 每轮拉最多 5 条未 ingest message（get_history_messages(limit=5)）
2. 按 semantic_categories 并发
3. 每条 message：
   a. 取该 set/category 已有 features（max_features_per_update=50）
   b. LLM 输出 ADD / DELETE 命令
   c. ADD：value embedding → add_feature；有 citation_id 则 add_citations([message.uid])
   d. DELETE：按 set/category/feature/tag 过滤删除
4. 成功处理的 message 才 mark_messages_ingested
5. 达到 consolidated_threshold=20 后触发 consolidation
```

- `_apply_commands` **只实现 ADD/DELETE**；未知命令只 `logger.error`（EV-048）。
- context 超限的 message 会被跳过并标记 ingested；其它异常在 `debug_fail_loudly=False` 时 continue（EV-046）。
- consolidation：LLM 返回 `keep_memories` + `consolidated_memories`；**先 delete_features，再并发 add_feature**；被删 feature 的全部 citations 合并后挂到每个新 feature（EV-050、EV-051、EV-052）。

## 3. 读取路径

### 3.1 顶层查询分派

`MemMachine.query_search`（EV-036、EV-037、EV-044）：

```
episodic_task = create_task(_search_episodic_memory(...))
semantic_task = create_task(_search_semantic_memory(...))  # 若 Semantic 被请求
await 两者
返回 SearchResponse(episodic_memory, semantic_memory)
```

- episodic 可选 `agent_mode`：
  - `retrieval_agent is None` 或 LTM 为空 → `episodic_session.query_memory`
  - 否则 → `_query_episodic_with_retrieval_agent`（EV-063）
- semantic 查询对多个 set_id 分别 embed/检索后 `merge_async_iterators`（EV-053）。

### 3.2 EpisodicMemory 查询

`EpisodicMemory.query_memory`（EV-044、EV-045）：

- `mode=BOTH`：并发查 STM + LTM；**短时记忆优先**，其 UID 集合用于去重 LTM；LTM 结果按分数排序。
- `mode=LONG_TERM_ONLY` / `SHORT_TERM_ONLY`：只查一侧。
- 返回 `ShortTermMemoryResponse(episodes, episode_summary)` + `LongTermMemoryResponse(episodes with score)`。
- `formalize_query_with_context` 把 summary + episodes 按 `(created_at, uid)` 排序后格式化进 query（EV-045）。

### 3.3 declarative 检索（EV-028）

```
1. 对 Derivative_<session> 向量搜索
   limit = min(5 * max_num_episodes, 200)
2. 通过 DERIVED_FROM 反查 Episode_<session>
3. 以命中 episode 为 nucleus，按 timestamp/uid 前后扩展 expand_context
   （前 1/3、后 2/3）
4. 对整段 context 用 reranker 打分
5. 按分数排序 + _unify_scored_anchored_episode_contexts 去重合并
```

- **召回单位是 derivative；返回单位是 episode + 邻域上下文**。

### 3.4 event 检索（EV-031、EV-032、EV-039）

```
1. embedder.search_embed(query)
2. vector_store.query 取 seed segments
   （按 segment_uuid 去重，保留首个最高分）
3. get_segment_contexts 取前后窗口（前 1/3、后 2/3）
4. 无 reranker：用 embedding 分数；有 reranker：用 reranker 分数
5. 排序方向由 similarity_metric.higher_is_better 决定
```

- **召回/返回单位都是 segment 窗口**；需要 `episode_storage` 水合才能还原 Episode。

## 4. 删除路径

### 4.1 删除 episode

- declarative：`LongTermMemory.delete_episodes` → `DeclarativeMemory.delete_episodes` → 并行删节点/边（EV-029、EV-039）。
- event：`LongTermMemory.delete_episodes` → 把 episode uid 映射成 event uuid（`uuid5` namespace）→ `EventMemory.forget_events`（EV-039、EV-032）。
- event 删除顺序：**先查 event→segments、segments→derivatives；先删 vector，再删 segment**（EV-032）。
- 顶层 `_delete_memories` 并行删 episodic + semantic，`asyncio.gather` **没有 return_exceptions**；一侧失败会抛出，另一侧可能已删（EV-087）。

### 4.2 删除 session

`MemMachine.delete_session`（EV-041、EV-042、EV-043、EV-044）：

```
1. get_session_info；不存在 → SessionNotFoundError
2. 已是 Deleted → 直接返回（幂等）
3. update_session_status(status="delete")     # 逻辑删除，不改数据
4. if started: 入 deletion_queue
5. 后台 worker:
   - 获取 SQL lease "session-deletion:<key>"（wait_timeout=0）
   - 复核 status 仍为 delete
   - asyncio.TaskGroup:
       delete episode store
       if episodic enabled: delete episodic memory (partition drop)
       if semantic enabled: delete semantic memory
   - TaskGroup 全部成功后，manager.delete_session（物理删 session 行）
```

- `TaskGroup` 的语义是“任一子任务失败 → 其余取消并抛出”，所以部分清理可能发生，session 行可能保留待下次扫描重试（EV-043）。
- event backend 的 `drop_session_partition` 删除 VectorStore collection + SegmentStore partition；实例随后不可复用（EV-044）。

## 5. 状态机

### 5.1 Session

```
(absent) --create_or_validate_session--> Active --delete_session--> Deleted
   ▲                                         │                       │
   └───────────── create (same key) ─────────┘                       │
                                                                     ▼
                                               deletion worker (lease + TaskGroup)
                                                                     │
                                                                     ▼
                                                                  (row removed)
```

- `SessionStatus` 只有 `active` / `delete`（EV-041）。
- 删除是**两阶段**：先状态标记（快、幂等），再后台物理清理（慢、可重试）。
- 多 worker 通过 SQL lease 防止重复清理；租约拿不到就等下一次扫描（EV-043）。

### 5.2 Short-term memory

```
append episodes → len > capacity? → _do_evict
   ├─ 弹出已摘要旧条目（len > current_episode_count）
   └─ 仍满 → 整批交给 consolidator.summarize → 重置计数
consolidator:
   buffer → 后台 worker → _create_summary
       ├─ 成功 → save_short_term_memory(summary, last_seq, episode_num)
       └─ 失败 → 记录异常 / 保留旧 summary
```

- `max_summary_length_words = ceil((message_capacity / 2 / 8) / 100) * 100`（EV-038）。
- `_create_summary` 上下文超限会递归二分；单条 episode 按字符中点拆两半（EV-047）。
- consolidator 异常只 `logger.exception`，不 crash（EV-047、EV-048）。

### 5.3 EpisodicMemory 实例缓存

`MemoryInstanceCache`（EV-049、EV-050）：

```
get(key): ref_count += 1；命中则移到队首
release(key): ref_count -= 1；capacity==0 且 ref_count==0 → 入 disposal queue
janitor: 异步 close 实例
evict: 跳过 ref_count>0 的节点；超过 max_lifetime 的节点入 disposal queue
erase(key): ref_count>0 → RuntimeError（防止删正在使用的实例）
```

- `instance_cache_size=0`（样例默认）表示**不缓存空闲实例**；用完即入 disposal queue（EV-049）。
- `max_lifetime` 提供兜底回收，避免长期驻留。

## 6. 数据一致性边界（事实陈述）

| 场景 | 是否有事务 | 失败后果 | 证据 |
|---|---|---|---|
| 顶层三路写 | 否 | 一路成功一路失败；统一抛错但无补偿 | EV-016、EV-017 |
| STM + LTM 写 | 否 | 一侧成功一侧失败；内层先抛出 | EV-019 |
| declarative 节点 + 边 | 否 | 孤立 derivative（无 DERIVED_FROM） | EV-026 |
| event segment + vector | 否 | 孤儿 segment（无向量） | EV-030 |
| event 删除 | 否 | 先删索引后删 segment；中途失败留下不可检索但未删的 segment | EV-032 |
| semantic consolidation | 否 | 先删后加；新增失败 → 记忆丢失 | EV-050、EV-051 |
| session 删除 | 否（TaskGroup 非事务） | 部分存储已删；session 行可能保留待重试 | EV-043 |
| 索引/原文漂移 | 无检测修复 | 检索 warning 并丢弃缺失 episode | EV-033 |

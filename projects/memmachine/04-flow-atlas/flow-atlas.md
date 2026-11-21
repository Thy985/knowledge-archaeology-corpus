# Flow Atlas — 七类流

> 七类流：Control / State / Data / Evidence / Authority / Memory / Policy。
> 每条 Edge 尽量给出 `symbol`、`file:line`、`condition`、`state transition`、`evidence`。没有运行观测，所有流是**静态代码导出**（S3）。
> `Flow→KO 交叉校验` 见 §8。

---

## 1. Control Flow（请求控制）

| Edge | From → To | Symbol / File:Line | Condition | Evidence |
|---|---|---|---|---|
| F-C-01 | HTTP request → FastAPI route | `router.py` handler | 路径匹配 `/v2/...` | EV-011 |
| F-C-02 | route → service adapter | `api_v2/service.py:_add_messages_to` / `_search_target_memories` | 请求体通过 pydantic 校验 | EV-011、EV-055 |
| F-C-03 | service → `MemMachine.add_episodes` | `main/memmachine.py:778` | `MemoryType` 含 Episodic/Semantic | EV-016 |
| F-C-04 | `add_episodes` → 1~3 路条件并行 | `asyncio.gather(..., return_exceptions=True)` `main/memmachine.py:869` | episode_storage 总是；episodic/semantic 按 `target_memories` | EV-016 |
| F-C-05 | `add_episodes` → 错误聚合 | `_raise_add_episode_errors` `main/memmachine.py:84` | 任一 task 返回 Exception | EV-016 |
| F-C-06 | `query_search` → episodic task | `asyncio.create_task(_search_episodic_memory)` `main/memmachine.py:1085-1162` | 请求含 Episodic | EV-065 |
| F-C-07 | `query_search` → semantic task | `asyncio.create_task(_search_semantic_memory)` | 请求含 Semantic 且 enabled | EV-065 |
| F-C-08 | `_search_episodic_memory` → retrieval agent | `_query_episodic_with_retrieval_agent` `main/memmachine.py:880-937` | `agent_mode` 且 retrieval_agent 非空且 LTM 非空 | EV-063 |
| F-C-09 | `_search_episodic_memory` → direct query | `episodic_session.query_memory` | retrieval_agent 为空或 LTM 为空 | EV-063 |
| F-C-10 | `delete_session` → deletion queue | `delete_session` `main/memmachine.py:652-674` | 状态从 Active → Deleted 且 worker 已启动 | EV-042、EV-043 |
| F-C-11 | deletion worker → TaskGroup | `_delete_queued_session_data` `main/memmachine.py:350-440` | 拿到 SQL lease 且 status 仍为 delete | EV-043 |
| F-C-12 | MCP tool → service | `mcp.py:add_memory/search_memory/delete_memory` | MCP 调用 | EV-014 |
| F-C-13 | config route → ConfigService | `config_router.py` `Depends(get_config_service)` | `MEMMACHINE_CONFIG_API` 且路径 `/config` | EV-075 |

### 控制流中的异常分支

| Edge | Branch | Symbol | Condition | Effect |
|---|---|---|---|---|
| F-C-14 | `gather` 部分失败 | `_raise_add_episode_errors` | 任一 Exception | 调用方收到失败；已写数据保留（EV-016、EV-017） |
| F-C-15 | 内层 STM/LTM 失败 | `EpisodicMemory.add_memory_episodes` | 任一 await 抛错 | 先失败先抛出，另一侧可能已写（EV-019） |
| F-C-16 | LTM 缺依赖 | `_disable_long_term_memory` | 必需资源缺失 | LTM disabled，仅 warning（EV-021） |
| F-C-17 | semantic 缺凭证 | `_maybe_disable_semantic_memory` | OpenAI key 空且无 base_url | semantic disabled（EV-024） |

---

## 2. State Flow（状态迁移）

### 2.1 Session 状态

| Edge | State transition | Trigger | Guard | Evidence |
|---|---|---|---|---|
| F-S-01 | `(absent) → Active` | `create_or_validate_session` | 配置匹配；不存在则建 | EV-041 |
| F-S-02 | `Active → Deleted` | `delete_session` | 存在且当前非 Deleted | EV-042 |
| F-S-03 | `Deleted → (deleting)` | worker 取队列 | status 仍为 `delete`；lease 成功 | EV-043 |
| F-S-04 | `(deleting) → (row removed)` | `manager.delete_session` | TaskGroup 全部成功 | EV-043 |
| F-S-05 | `Deleted → Deleted`（幂等） | 再次 `delete_session` | 已是 Deleted | EV-042 |

### 2.2 Short-term memory 状态

| Edge | State transition | Trigger | Guard | Evidence |
|---|---|---|---|---|
| F-S-06 | `len ≤ capacity → len > capacity` | `add_episodes` | 写锁内 extend | EV-034 |
| F-S-07 | `len > capacity → 弹旧已摘要条目` | `_do_evict` | `len > current_episode_count` 且仍 full | EV-034 |
| F-S-08 | `仍 full → 全量 summarize` | `_consolidator.summarize` | 弹出后仍 full | EV-034 |
| F-S-09 | `summary pending → summary updated` | `_create_summary` → `save_short_term_memory` | LLM 成功 | EV-037 |
| F-S-10 | `summary pending → 旧 summary 保留` | `_create_summary` 返回旧值 | 外部 API/ValueError/RuntimeError | EV-037 |

### 2.3 实例缓存状态

| Edge | State transition | Trigger | Guard | Evidence |
|---|---|---|---|---|
| F-S-11 | `absent → cached(ref=1)` | `get` | 建实例并加引用 | EV-049 |
| F-S-12 | `cached → disposed` | `release` | `capacity==0` 且 ref==0 | EV-049 |
| F-S-13 | `cached → disposed` | janitor evict | `ref_count==0` 且 `max_lifetime` 到期 | EV-049 |
| F-S-14 | `cached → RuntimeError` | `erase` | ref_count>0 | EV-050 |
| F-S-15 | `LTM 可用 → 实例作废` | `drop_session_partition` | event partition 已删除；handles 置 None | EV-092 |

---

## 3. Data Flow（数据形态）

```
EpisodeEntry
  ├─(F-D-01) episode_storage: Episode row
  ├─(F-D-02) EpisodicMemory → STM deque
  └─(F-D-03) EpisodicMemory → LTM
                     ├─ declarative: Episode node + Derivative node + DERIVED_FROM edge
                     └─ event: Segment row + Derivative row + vector record
Semantic message history
  └─(F-D-04) IngestionService → LLM → ADD/DELETE → SemanticFeature + Citation
```

| Edge | From → To | Transform | Symbol / File:Line | Evidence |
|---|---|---|---|---|
| F-D-01 | `EpisodeEntry` → `Episode` | 补 `created_at`、batch uuid、session_key | `main/memmachine.py:778-879` | EV-016 |
| F-D-02 | `Episode` → STM deque | 直接 append（无摘要） | `short_term_memory.py:189-243` | EV-034 |
| F-D-03 | `Episode` → declarative nodes | episode 节点 + derivative 节点 | `declarative_memory.py:99-212` | EV-026 |
| F-D-04 | `Episode` → event segment/derivative | segmenter → deriver → embedder | `event_memory.py:223-317` | EV-030 |
| F-D-05 | `EpisodeEntry` → semantic history | `add_message` 登记 `is_ingested=False` | `semantic_session_manager.py` / `semantic_memory.py` | EV-045 |
| F-D-06 | semantic message → `SemanticFeature` | LLM ADD/DELETE + embedding | `semantic_ingestion.py:299-347` | EV-048 |
| F-D-07 | feature → citation | `add_citations([message.uid])` | `semantic_ingestion.py:323` | EV-048 |
| F-D-08 | consolidated feature → citation | 合并旧 citation 到每个新 feature | `semantic_ingestion.py:537` | EV-052 |
| F-D-09 | vector record → search result | vector query → segment/episode hydrate | `event_memory.py:398-535` / `declarative_memory.py:313-500` | EV-028、EV-031 |

### 数据形态的关键约束

- `filterable_metadata` 只接受 `PropertyValue` 类型；非该类型的 metadata 不进过滤索引（EV-019）。
- `Derivative` 的文本不是原文：declarative 默认加 `"{source}: "` 前缀；event 由 segmenter/deriver 决定（EV-027、EV-030）。
- `SemanticFeature` 的 value 是 LLM 抽取结果，不是原句；citation 是唯一回链（EV-048）。

---

## 4. Evidence Flow（证据/溯源）

| Edge | From → To | Symbol / File:Line | Condition | Evidence |
|---|---|---|---|---|
| F-E-01 | message → feature citation | `add_citations(f_id, [message.uid])` `semantic_ingestion.py:323` | LLM 返回 citation_id | EV-048 |
| F-E-02 | old features → consolidated feature citations | `add_citations(f_id, citation_ids)` `semantic_ingestion.py:537` | consolidation 有删除+新增 | EV-052 |
| F-E-03 | derivative → episode | `DERIVED_FROM_<session>` edge | declarative 写入成功 | EV-026 |
| F-E-04 | segment → derivative | `segment_store_dv_ln` 关系 | event 写入成功 | EV-030、EV-075 |
| F-E-05 | vector hit → episode | `episode_storage` hydrate | event 查询 | EV-031、EV-033 |
| F-E-06 | PG: feature DELETE → citation rows 级联删除；Neo4j: 节点+属性删除 | `delete_features` / `DETACH DELETE` | consolidation；PG FK `ON DELETE CASCADE` | EV-091 |
| F-E-07 | history DELETE → citation rows（按 history_id） | `delete_history` | 显式删除历史 | EV-091 |

### 证据流的断点（事实）

- **F-E-02 是“过度归属”而不是“精确映射”**：旧 citations 全量挂到每个新 feature；无法判断某条 citation 支持新 feature 的哪一部分（EV-052）。
- ~~F-E-06 未在代码中找到 citation 随 feature 删除的联动~~ **已被 Independent Validation 推翻**：PG `citations.feature_id` 有 `ON DELETE CASCADE`；Neo4j `DETACH DELETE` 删属性。C-06 降为条件性 Hypothesis（EV-091）。
- **F-E-07 是另一条 citation 生命周期**：`delete_history` 按 `history_id` 清 citation；它与 F-E-06 不是同一条路径（EV-091）。
- **F-E-03 的边可能缺失**：节点先写、边后写；边失败留下孤立 derivative，检索命中后无法回填（EV-026、EV-033）。

---

## 5. Authority Flow（权威/权限）

| Edge | Authority source → Scope | Symbol / File:Line | Condition | Evidence |
|---|---|---|---|---|
| F-A-01 | REST body `org_id/project_id` → `session_key` | `service.py:39-46` | 请求通过 schema 校验 | EV-015 |
| F-A-02 | MCP header `org-id/proj-id` → ContextVar | `mcp.py:257-300` | header 存在；缺失为空串 | EV-013 |
| F-A-03 | `session_key` → episode partition | episode store `session_key` 列 | 写入/查询 | EV-015、EV-086 |
| F-A-04 | `session_key` → event collection/partition | event backend `session_id` | LTM event | EV-044、EV-075 |
| F-A-05 | `set_id` → semantic set | semantic service | semantic 请求 | EV-053 |
| F-A-06 | config API → resource managers | `ConfigService` | `MEMMACHINE_CONFIG_API` | EV-075 |

### 权威流的边界

- **没有“认证→身份”边**：F-A-01/F-A-02 是“声明→命名空间”，不是“凭证→身份”。服务端没有校验调用方是否有权使用该 org/project（EV-012、EV-013）。
- **没有“角色→操作”边**：不存在 admin/user 角色区分；任何能调用 REST 的客户端都能调用 delete/config（若 config 开启）（EV-011、EV-075）。
- **治理文档不进入运行时**：`GOVERNANCE.md` 只约束合并流程，不产生运行时审批（EV-079）。

---

## 6. Memory Flow（记忆生命周期）

```
add_episodes
  ├─ STM: append → evict/summarize → summary persisted
  └─ LTM:
       ├─ declarative: Episode + Derivative + edge → search → rerank → context
       └─ event: Segment + Derivative + vector → search → window → rerank
delete_session
  ├─ status=delete → queue → lease → TaskGroup
  │     ├─ episode store delete
  │     ├─ episodic partition drop
  │     └─ semantic delete
  └─ row removed
semantic ingestion
  ├─ history (is_ingested=False) → 5/批 → LLM ADD/DELETE
  ├─ feature + citation
  └─ ≥20 → consolidation (delete old → add new → merge citations)
```

| Edge | From → To | Symbol / File:Line | Condition | Evidence |
|---|---|---|---|---|
| F-M-01 | episode → STM | `short_term_memory.add_episodes` | episodic enabled | EV-034 |
| F-M-02 | STM full → summary | `ShortTermMemoryConsolidator.summarize` | `len > capacity` 且弹出后仍 full | EV-034、EV-036 |
| F-M-03 | episode → declarative graph | `DeclarativeMemory.add_episodes` | LTM backend declarative | EV-026 |
| F-M-04 | episode → event segment/vector | `EventMemory._encode_events` | LTM backend event | EV-030 |
| F-M-05 | declarative derivative → episode context | `search_scored` | 查询 | EV-028 |
| F-M-06 | event segment → window context | `query` | 查询 | EV-031 |
| F-M-07 | feature set → consolidation | `_consolidate_set_memories_if_applicable` | feature count ≥ 20/tag | EV-049 |
| F-M-08 | session → deleted | `delete_session` | 调用删除 | EV-042 |
| F-M-09 | segment partition → dropped | `drop_session_partition` | session 删除 | EV-044 |
| F-M-10 | dead tenant → purge queue | `segment_store_gc` | registry 标记 dead | EV-075（设计文档） |

### 记忆流的关键事实

- **STM 摘要是“有损且可能陈旧”的**：失败时保留旧 summary，不暴露 staleness（EV-036、EV-037）。
- **LTM 的“原文”在 event 路径是 EpisodeStorage，在 declarative 路径是 Episode 节点**；两者的删除路径不同（EV-026、EV-044）。
- **semantic 的“记忆”不是原文**：feature 由 LLM 生成，citation 是唯一回链；consolidation 会重写 feature 集合（EV-048、EV-052）。

---

## 7. Policy Flow（策略/治理）

> MemMachine 没有“策略提案→审批→执行→审计”的运行时闭环；唯一的策略面是**配置门控 + 可选 config API**。

| Edge | Policy decision → Enforcement | Symbol / File:Line | Condition | Evidence |
|---|---|---|---|---|
| F-P-01 | `episodic_memory.enabled` → 是否建 STM/LTM | `_initialize_default_episodic_configuration` `main/memmachine.py:103-175` | 启动 | EV-020 |
| F-P-02 | LTM backend + 必需资源 → LTM enabled/disabled | `_resolve_ltm_defaults` `main/memmachine.py:175-238` | 启动 | EV-021 |
| F-P-03 | `semantic_memory.enabled` + 凭证 → semantic enabled/disabled | `_maybe_disable_semantic_memory` `configuration/__init__.py:446-540` | 启动 | EV-023、EV-024 |
| F-P-04 | config API PUT → 更新 `Configuration` | `config_router.py` / `ConfigService` | `MEMMACHINE_CONFIG_API` | EV-075 |
| F-P-05 | `Configuration.save_config()` → YAML 文件 | `_persist_config` `config_service.py:387-430` | `config_file_path` 存在 | EV-076 |
| F-P-06 | scope/tenant → data partition | `session_key` / set_id | 每次请求 | EV-015、EV-053 |
| F-P-07 | session status → deletion allowed | `delete_session` | 调用方请求 | EV-042 |
| F-P-08 | governance doc → merge policy | `GOVERNANCE.md` | 代码合并（仓库外） | EV-079 |

### Policy Flow 的缺口（事实，不是建议）

1. **没有策略审批**：F-P-04 的 PUT 直接改配置，没有审批、双人复核、变更日志（除应用日志）。
2. **没有策略版本化**：`save_config()` 覆盖原文件；没有旧版本保留/回滚（EV-076）。
3. **没有策略→执行的强一致**：配置更新不重建缓存实例；部分变更需重启才生效（C-04）。
4. **没有运行时审计**：配置 API 的调用不记录“谁改了什么”，只有通用访问日志（EV-088）。
5. **治理文档与运行时策略分离**：GOVERNANCE 的 Lazy Consensus 不约束运行时（EV-079）。

---

## 8. Flow → KO 交叉校验

| KO | 依赖的关键 Flow Edge | 是否一致 | 说明 |
|---|---|---|---|
| KO-01 | F-D-03、F-D-04、F-D-09、F-E-03、F-E-04 | 一致 | 原文+索引+回填链在两条 backend 都存在 |
| KO-02 | F-C-04、F-C-05、F-C-15、F-D-03、F-D-04、F-E-03、F-E-04 | 一致 | 无补偿写入链可由控制流/数据流复现 |
| KO-03 | F-C-16、F-C-17、F-P-01~03 | 一致 | 配置门控链完整；运行时恢复见 F-P-04 |
| KO-04 | F-D-09、F-M-05、F-M-06 | 一致 | 两条检索的单位不同可由 Data/Memory Flow 复现 |
| KO-05 | F-A-01~06 | 一致 | 声明→命名空间链路完整；无认证边 |
| KO-06 | F-D-05、F-D-06、F-M-07 | 一致 | 两阶段链完整；consolidation 是第三阶段 |
| KO-07 | F-M-07、F-E-02、F-E-06、F-E-07 | 一致（C-06 已撤回） | PG 级联删除 + history 清理均已确认；仍保留“过度归属”反例 |
| KO-08 | F-S-01~15、F-M-08~10 | 一致 | 状态机/延迟清理链完整；F-S-15 补“实例作废” |

## 9. Flow Atlas 质量自检

- **Control**：13 条主边 + 4 条异常边；覆盖 REST/MCP/config/delete。
- **State**：15 条边；覆盖 session、STM、缓存、partition 删除后实例作废。
- **Data**：9 条边；覆盖 Episode→三种表示→检索结果。
- **Evidence**：7 条边；F-E-06（FK 级联）与 F-E-07（history 清理）是两条删除路径。
- **Authority**：6 条边；无“认证→身份”边，是事实。
- **Memory**：10 条边；覆盖写/读/删除/consolidation/purge。
- **Policy**：8 条边；覆盖启动门控 + 运行期配置 + 治理文档。
- **未覆盖**：向量/图存储内部查询计划、LLM prompt 细节、前端 SDK 重试逻辑；这些不影响本轮 KO 的证据链，列为 coverage gap（见 06_validation）。

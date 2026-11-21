# Engineering Knowledge（EK Graph 节点）— 52 条

> 口径：每条 EK 都是**项目级**工程知识（L1/L2），不是跨项目原则。`links` 使用六类边：`mechanism` / `subsystem` / `causal` / `dependency` / `constraint` / `contrast`。
> `Strength`：S3 = 静态代码可核；S2 = 文档声称。`Epistemic`：Fact（直接读出）/ Observation（跨 2+ 证据归纳）。
> **本文件不主张 S4**：测试代码被阅读，但未执行。

---

## A. 架构与契约（EK-01 ~ EK-12）

### EK-01 顶层 `MemMachine` 是唯一的编排权威，但把具体语义下沉给两个记忆域
- **Layer**: L1 · **Type**: ARCHITECTURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `MemMachine` 持有 `episode_storage`、`SessionDataManager`、`MemoryInstanceCache<EpisodicMemory>`、`SemanticService`；对外提供 `add_episodes` / `query_search` / `delete_session` / feature API。它决定“何时调哪个子系统”，但不决定 STM/LTM 如何合并、derivative 如何生成。
- **Evidence**: EV-016、EV-065、EV-039
- **Links**: `causal:EK-14`（编排决定三路写）；`causal:EK-32`（编排决定查询分派）；`subsystem:EK-31`
- **Note**: 这使“跨后端的一致性”只能在顶层做，而顶层恰恰没有事务。

### EK-02 长期记忆是 facade：`declarative` 与 `event` 两个 backend 共享接口、不同数据模型
- **Layer**: L2 · **Type**: ARCHITECTURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `LongTermMemory` 按 `params.backend` 分派到 `DeclarativeMemory` 或 `EventMemory`；两者都实现 `add_episodes` / `search_scored` / `delete_episodes`，但 `DeclarativeBackendParams` 要 graph+embedder+reranker，`EventBackendParams` 要 vector_store+segment_store+episode_storage+embedder（reranker 可选）。
- **Evidence**: EV-039、EV-021
- **Links**: `contrast:EK-03`；`contrast:EK-04`；`constraint:EK-08`（参数必填决定能否启用）
- **Note**: “同一接口”掩盖了上下文单位、删除顺序、失败模式的差异（EK-34）。

### EK-03 declarative backend 用“原始 Episode 节点 + 派生 Derivative 节点 + DERIVED_FROM 边”表达记忆
- **Layer**: L1 · **Type**: DATA · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 每个 episode 写成一个 `Episode_<session>` 节点；每个派生片段写成 `Derivative_<session>` 节点；两者用 `DERIVED_FROM_<session>` 有向边连接。检索命中 derivative，再沿边回到 episode。
- **Evidence**: EV-026、EV-028
- **Links**: `causal:EK-18`；`causal:EK-32`；`subsystem:EK-19`
- **Note**: 这是“原文权威 + 索引视图”的显式图结构；边是检索正确性的必要条件，不是装饰。

### EK-04 event backend 用“Segment + Derivative + 向量记录”表达记忆，原文在 EpisodeStorage
- **Layer**: L1 · **Type**: DATA · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: segmenter 把 event 切成 segment，deriver 把 segment 切成 derivative，embedder 给 derivative 向量；segment/derivative 关系落 segment store，向量落 vector store；返回时用 `episode_storage` 水合。event 本身通过 `uuid5` 与 episode uid 建立稳定映射。
- **Evidence**: EV-030、EV-031、EV-032
- **Links**: `contrast:EK-03`；`causal:EK-20`；`dependency:EK-51`
- **Note**: event 后端的“记忆单位”是 segment 窗口，不是 episode；这直接改变上下文扩展语义（EK-34）。

### EK-05 多租户的运行时身份是 `org_id/project_id` 拼成的 session_key
- **Layer**: L1 · **Type**: PERMISSION · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `_SessionData.session_key = f"{org_id}/{project_id}"`；episode store、episodic partition、semantic set 都围绕这个 key 分区。`user_id` 在 MCP 中另作 producer/context 标识，但不是鉴权主体。
- **Evidence**: EV-015、EV-013、EV-083
- **Links**: `constraint:EK-10`（无认证使命名空间成为唯一边界）；`subsystem:EK-31`；`causal:EK-48`
- **Note**: 命名空间隔离依赖调用方不伪造 org/project；服务端只校验字符格式。

### EK-06 配置是一棵强类型树，加载时统一小写 key，环境变量只覆盖秘密字段
- **Layer**: L1 · **Type**: CONFIG · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `Configuration` 含 9 个顶层段；`load_yml_file` 递归把 dict key 转小写，再交给 pydantic；`$ENV` 解析只发生在 `PasswordMixin`、`ApiKeyMixin`、AWS mixin，普通字段不展开。
- **Evidence**: EV-070、EV-071、EV-072
- **Links**: `causal:EK-07`；`constraint:EK-08`；`subsystem:EK-12`
- **Note**: 配置错误通常在启动时暴露为 pydantic 错误，而不是运行期静默失败；但“缺依赖自动关闭”是例外（EK-08）。

### EK-07 记忆子系统的默认姿态是关闭，而不是“默认全开”
- **Layer**: L2 · **Type**: CONFIG · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `episodic_memory.enabled` 默认 False；LTM/STM 在未配置时都关闭；`semantic_memory.enabled` 默认 False。只有显式配置并满足依赖时，记忆才生效。
- **Evidence**: EV-020、EV-023
- **Links**: `causal:EK-08`；`constraint:EK-09`；`contrast:EK-12`
- **Note**: 这是“可用性优先”的诚实设计：不假装有记忆；代价是部署方容易把“服务起来了”误判为“记忆生效了”。

### EK-08 LTM 的启用是“全有或全无”的依赖门控
- **Layer**: L2 · **Type**: CONFIG · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: declarative 后端要求 embedder + reranker + vector_graph_store 三者齐备；event 后端要求 vector_store + segment_store 齐备。任一缺失 → `_disable_long_term_memory`，只 warning，不抛错。
- **Evidence**: EV-021
- **Links**: `causal:EK-07`；`contrast:EK-09`；`constraint:EK-02`
- **Note**: 部分配置（例如只配了 embedder）不会被“降级使用”，而是整层关闭；这避免半开状态，但也放大了配置漂移的影响。

### EK-09 缺失模型凭证会触发子系统自动关闭，且只留 warning
- **Layer**: L2 · **Type**: CONFIG · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `_maybe_disable_semantic_memory` / `_maybe_disable_episodic_memory` 在 OpenAI `api_key` 为空且无 `base_url` 时把对应子系统设为 disabled；启动继续。
- **Evidence**: EV-024
- **Links**: `constraint:EK-07`；`contrast:EK-08`；`subsystem:EK-06`
- **Note**: 与 EK-08 的区别：这是“凭证缺失”，不是“结构缺失”；两者都导致静默降级，但根因不同。

### EK-10 REST v2 自身不做认证，身份由请求体声明
- **Layer**: L1 · **Type**: PERMISSION · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `router.py` 的所有业务 handler 只依赖 `get_memmachine`；没有 `Security`、OAuth、API-key 或签名校验；`org_id` / `project_id` 是 `_WithOrgAndProj` 的普通字段。服务端只校验字符合法性（例如拒绝含 `/` 的 org_id）。
- **Evidence**: EV-012、EV-069、EV-083
- **Links**: `constraint:EK-05`；`mechanism:EK-11`；`contrast:EK-12`
- **Note**: 这不是“未实现鉴权”的口吻判断，而是架构分工：认证被假定由前置网关完成。若直接暴露公网，边界由部署方承担（C-03）。

### EK-11 MCP 用 header 声明身份，且缺省为空字符串
- **Layer**: L1 · **Type**: PERMISSION · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `ParamsContextMiddleware` 从 `org-id` / `proj-id` / `user-id` header 读取身份，写入 `ContextVar`；header 缺失时值为 `""`，不拒绝请求。MCP 只暴露 3 个 tool。
- **Evidence**: EV-013、EV-014
- **Links**: `mechanism:EK-10`；`constraint:EK-05`
- **Note**: 空字符串会成为合法命名空间；多租户隔离完全依赖调用方发送正确 header。

### EK-12 `/config` 是可选挂载的运行时控制面，能改配置但不重建实例
- **Layer**: L2 · **Type**: TOOLING · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: 仅 `MEMMACHINE_CONFIG_API` 开启时挂载；支持读配置/资源、PUT 各层配置、增删 embedder/LM、重试 reranker；`_persist_config` 只在配置文件有路径时写回，失败只 warning。配置更新不会自动替换已缓存的 `EpisodicMemory` 实例。
- **Evidence**: EV-075、EV-076
- **Links**: `contrast:EK-10`；`subsystem:EK-06`；`causal:EK-47`
- **Note**: 这构成一个“运行期策略变更”入口，但没有审批、版本化或回滚；是 Policy Flow 的唯一实现面（见 Flow Atlas）。

---

## B. 写入路径（EK-13 ~ EK-31）

### EK-13 写入批次用 `sorted(uuid4())` 生成，保证同时间戳下的稳定排序
- **Layer**: L1 · **Type**: DATA · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `add_episodes` 为每批生成 `batch_ids = sorted(uuid4() for _ in episode_entries)`；当多个 episode 共享同一 `created_at` 时，排序键退化为 uid，仍可稳定比较。
- **Evidence**: EV-016
- **Links**: `causal:EK-14`；`subsystem:EK-19`
- **Note**: 这是对“时间戳精度不足”的防御，但也意味着 uid 顺序会影响上下文窗口的先后。

### EK-14 顶层写入是“1~3 路条件并行 + 统一抛错”，不是事务
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `add_episodes` 总是写 episode_storage；仅当 `MemoryType.Episodic in target_memories` 时追加 episodic_memory（STM+LTM）；仅当 `_should_dispatch_to_semantic_memory(target_memories)` 时追加 semantic_session_manager。用 `asyncio.gather(return_exceptions=True)` 等已组装任务结束后，再由 `_raise_add_episode_errors` 统一抛错。
- **Evidence**: EV-016、EV-018
- **Links**: `causal:EK-15`；`causal:EK-16`；`contrast:EK-17`
- **Note**: “都跑完”降低了悬挂风险，但没有补偿；抛错只告诉调用方“有失败”，不告诉“哪份数据已落库”。

### EK-15 `return_exceptions=True` 把部分失败延迟到聚合点，但不保留失败明细
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `gather(return_exceptions=True)` 收集异常对象；`_raise_add_episode_errors(task_names, results, episode_ids)` 只按 task 名与 episode ids 组装错误信息并抛出，不写持久化的 write-intent/补偿记录。
- **Evidence**: EV-016、EV-017
- **Links**: `causal:EK-14`；`constraint:EK-16`
- **Note**: 这使“部分写入”在调用方看来是一次失败，但系统内部没有“待修复”状态。

### EK-16 源码明确承认“部分写失败不可检测/修复”（TODO #1738）
- **Layer**: L1 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 代码注释：`TODO(#1738): Track write intent so partial failures across these parallel backends can be detected and repaired.` 该 TODO 位于 `asyncio.gather` **之前**（约 867 行），不是 gather 之后；结论“无 write intent/无补偿”成立。
- **Evidence**: EV-017
- **Links**: `causal:EK-15`；`dependency:EK-51`（漂移告警是当前的被动兜底）
- **Note**: 这是项目自己承认的结构性缺口，不是考古推断。

### EK-17 STM 与 LTM 的内层并发语义与顶层不同：先失败先抛出
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `EpisodicMemory.add_memory_episodes` 用 `asyncio.gather(short_term.add_episodes, long_term.add_episodes)`，**没有** `return_exceptions`；任一抛错即中断，另一侧可能已完成。
- **Evidence**: EV-019
- **Links**: `contrast:EK-14`；`causal:EK-15`；`subsystem:EK-45`
- **Note**: 顶层“等全部完成”与内层“先失败先抛出”是两套并发契约；调用方无法从异常类型推断哪一侧已写。

### EK-18 declarative 写入顺序是“节点并行 → 边串行”，边失败会留下孤立 derivative
- **Layer**: L1 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 先 `gather(add Episode nodes, add Derivative nodes)`，再 `add_edges(DERIVED_FROM)`。节点与边不在同一事务；若边写失败，derivative 节点存在但无法反查 episode。
- **Evidence**: EV-026
- **Links**: `causal:EK-03`；`dependency:EK-32`；`mechanism:EK-20`
- **Note**: derivative 是检索入口；没有边的 derivative 会成为“可被向量搜到但无法回填原文”的幽灵。

### EK-19 `message_sentence_chunking` 决定 MESSAGE 型 episode 的派生粒度
- **Layer**: L1 · **Type**: DATA · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: 默认 MESSAGE 整条派生成一个 derivative，并加 `"{source}: "` 前缀；`message_sentence_chunking=True` 时按句拆分；TEXT 整段一条；不支持的类型只 warning 并返回空。
- **Evidence**: EV-027
- **Links**: `subsystem:EK-03`；`causal:EK-32`；`contrast:EK-20`（event 用 segmenter/deriver）
- **Note**: 粒度是召回质量与上下文完整性的直接权衡；默认“整条”保留了原文，但降低了细粒度命中。

### EK-20 event 写入是五阶段流水线，任何后段失败都留下前段产物
- **Layer**: L1 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `_encode_events`：segment（236）→ derive（247）→ embed（271）→ `add_segments`（274）→ vector `upsert`（292）；每阶段有 metrics/timing，但没有跨阶段事务或补偿。
- **Evidence**: EV-030
- **Links**: `causal:EK-21`；`mechanism:EK-18`；`subsystem:EK-04`
- **Note**: 这与 declarative 的“节点→边”同构：先写权威关系，后写检索索引。

### EK-21 event 写顺序是 segment store 先、vector store 后
- **Layer**: L1 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `add_segments` 先落 segment/derivative 关系，`vector_store_collection.upsert` 后落向量。若 upsert 失败，segment 已存在但不可检索（孤儿段）。
- **Evidence**: EV-030
- **Links**: `causal:EK-20`；`contrast:EK-33`（删除顺序相反）；`dependency:EK-51`
- **Note**: 写与删的顺序相反，是为了“写入时先保原文、删除时先断检索”；代价是两种中间态都不可原子恢复。

### EK-22 semantic 写入只登记 message，抽取由后台异步完成
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `add_message_to_sets` 把 message 写入 set 的 history（`is_ingested=False`）并返回；`IngestionService` 后续按 set 拉取未 ingest 消息，调用 LLM 生成 feature。调用方看到的“写入成功”不等于“semantic 已抽取”。
- **Evidence**: EV-045、EV-046
- **Links**: `causal:EK-23`；`contrast:EK-14`；`subsystem:EK-30`
- **Note**: 这是两阶段记忆：原始 episode 同步写，结构化 profile 异步派生；两者可能短暂不一致。

### EK-23 semantic ingestion 每 set 每轮只取 5 条消息
- **Layer**: L1 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `get_history_messages(limit=5)` 固定批量；按 `semantic_categories` 并发处理；成功后才 `mark_messages_ingested`。
- **Evidence**: EV-046
- **Links**: `causal:EK-22`；`constraint:EK-24`；`subsystem:EK-25`
- **Note**: 小批量降低了 LLM 上下文压力与单次失败影响面，但也意味着积压会持续多轮。

### EK-24 每条消息的 feature 更新上下文上限是 50 条已有 feature
- **Layer**: L1 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: 每 set/category 取已有 features 时 `page_size=max_features_per_update=50`；LLM 基于这 50 条决定 ADD/DELETE。
- **Evidence**: EV-047
- **Links**: `constraint:EK-23`；`causal:EK-25`；`subsystem:EK-27`
- **Note**: 超过 50 条的 set 不会被全量看到，可能出现“重复新增”或“误删未见 feature”。

### EK-25 semantic 更新只有 ADD / DELETE 两种命令，未知命令仅记日志
- **Layer**: L1 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `_apply_commands` 实现 ADD（value embedding + add_feature + citation）与 DELETE（按 set/category/feature/tag 过滤）；其它命令 `logger.error` 后忽略。
- **Evidence**: EV-048
- **Links**: `causal:EK-24`；`causal:EK-26`；`contrast:EK-28`
- **Note**: 没有 UPDATE/MERGE 原语；合并只能由 consolidation 的“删旧+加新”模拟。

### EK-26 citation 把 feature 回溯到产生它的 message uid
- **Layer**: L1 · **Type**: EVIDENCE · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: ADD 时若 LLM 返回 `citation_id`，调用 `add_citations([message.uid])`；citation 是 feature→message 的 provenance 边。
- **Evidence**: EV-048
- **Links**: `causal:EK-25`；`causal:EK-29`；`dependency:EK-27`
- **Note**: provenance 的粒度是 message，不是句子或 feature 片段；这为后续“过度归属”埋下结构条件。

### EK-27 consolidation 在 feature 数达到 20 时触发
- **Layer**: L1 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `consolidated_threshold=20`；按 tag 分组，达到阈值才 `_deduplicate_features`；`_try_consolidate` 有最大尝试次数，context-length 单独抛错，其它失败在非 loud 模式返回 None。
- **Evidence**: EV-049、EV-050
- **Links**: `causal:EK-28`；`constraint:EK-24`；`contrast:EK-30`
- **Note**: 阈值是“成本/一致性”的折中；低频 set 可能长期不合并。

### EK-28 consolidation 先删旧 feature、再并发加新 feature，且无事务
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `delete_features(memories_to_delete)` 先执行；随后 `asyncio.gather(_add_feature(feature) for feature in consolidated_memories)`。新增失败时旧记忆已被删，造成不可恢复丢失。
- **Evidence**: EV-050、EV-051
- **Links**: `causal:EK-27`；`causal:EK-29`；`contrast:EK-18`（同样是“先写一部分再补”，但这里先删）
- **Note**: 这是全项目最直接的“破坏性维护”路径；它用 LLM 判断哪些记忆该合并，却没有撤销日志。

### EK-29 citation 合并把被删 feature 的全部 provenance 挂到每个新 feature
- **Layer**: L1 · **Type**: EVIDENCE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 代码把 `memories_to_delete` 的所有 citations 合并成 `citation_ids`，然后对**每个** consolidated feature 调 `add_citations(f_id, citation_ids)`。
- **Evidence**: EV-052
- **Links**: `causal:EK-28`；`mechanism:EK-26`；`contrast:EK-25`（ADD 的 citation 是单 message）
- **Note**: 结果是 provenance 膨胀：一个由 3 条消息支持的新 feature 会声称拥有 3 个旧 feature 的全部来源，无法再区分“哪条消息支持哪部分”。

### EK-30 semantic 多 set 写入/检索按 set 并行，结果用 merge iterator 汇合
- **Layer**: L1 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `add_messages` 对多个 set 用 `gather(return_exceptions=True)`；`search` 对每个 set_id 单独 embed/检索，`merge_async_iterators` 合并结果。
- **Evidence**: EV-045、EV-053
- **Links**: `subsystem:EK-22`；`causal:EK-43`；`mechanism:EK-14`
- **Note**: 多 set 之间没有全局排序；合并顺序依赖各 iterator 的产出节奏。

### EK-31 会话删除是“状态标记 + 后台 worker”，删除队列是进程内结构
- **Layer**: L2 · **Type**: STATE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `delete_session` 只把 status 改 `delete` 并入 `_deletion_queue`；后台 `_delete_session_worker` 消费；队列是进程内 `asyncio.Queue`，多 worker 依赖 SQL lease 与状态轮询收敛。
- **Evidence**: EV-042、EV-043
- **Links**: `causal:EK-48`；`causal:EK-49`；`subsystem:EK-05`；`subsystem:EK-52`
- **Note**: 删除是最终一致；调用方无法在返回时确认物理清理完成。

---

## C. 读取与检索（EK-32 ~ EK-44）

### EK-32 declarative 检索是“derivative 召回 → episode 回填 → 邻域扩展 → reranker”
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: 对 `Derivative_<session>` 向量搜索（limit=min(5*max,200)），沿 `DERIVED_FROM` 反查 `Episode_<session>`，以命中 episode 为 nucleus 按 `(timestamp, uid)` 前后扩展（前 1/3、后 2/3），对整段 context 用 reranker 打分，最后去重合并。
- **Evidence**: EV-028
- **Links**: `dependency:EK-18`；`dependency:EK-34`；`causal:EK-36`
- **Note**: 召回单位与返回单位不同；这使“命中一段”可以返回“一整段对话邻域”。

### EK-33 event 检索是“embed → seed segment → segment 窗口 → 可选 reranker”
- **Layer**: L2 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `search_embed` 后 vector query 取 seed segments（按 segment_uuid 去重保留最高分），`get_segment_contexts` 取前后窗口（前 1/3、后 2/3）；无 reranker 用 embedding 分数，有 reranker 用 reranker 分数；排序方向由 `similarity_metric.higher_is_better` 决定。
- **Evidence**: EV-031
- **Links**: `contrast:EK-32`；`dependency:EK-34`；`causal:EK-35`
- **Note**: event 的“上下文”是 segment 邻域，不是 episode 邻域；两者不可直接比较。

### EK-34 两个 backend 的“上下文扩展单位”不等价：declarative 是 episode 邻域，event 是 segment 窗口
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: declarative 以命中 episode 为 nucleus 扩展 episode；event 以命中 segment 为 nucleus 扩展 segment。相同 `expand_context` 数值在不同 backend 返回的语义粒度不同；event 还需 `episode_storage` 水合。
- **Evidence**: EV-028、EV-031、EV-039
- **Links**: `contrast:EK-32`；`contrast:EK-33`；`constraint:EK-42`
- **Note**: 这是“接口相同、语义不同”的核心例子；跨 backend 评测必须固定单位。

### EK-35 reranker 存在时会替换 embedding 分数，而不是叠加
- **Layer**: L1 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: event 查询在 `reranker` 存在时用 reranker 分数作为最终 score；declarative 对整段 context 用 reranker 打分后排序。不是“embedding + rerank”加权。
- **Evidence**: EV-031、EV-028
- **Links**: `causal:EK-33`；`causal:EK-36`
- **Note**: 因此“分数阈值”的语义随 reranker 是否启用而变化。

### EK-36 `score_threshold` 的契约是“None 保留全部 + 按相似度方向过滤”
- **Layer**: L2 · **Type**: CONTRACT · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `search_scored` docstring：`None` 不过滤；higher-is-better（cosine/dot/reranker）低于阈值丢弃；lower-is-better（raw euclidean/manhattan 且无 reranker）高于阈值丢弃；旧 `-inf` sentinel 在 euclidean 下会静默反转成“丢弃一切”。event 路径的 `_score_passes_threshold` 按 `self._score_higher_is_better` 实现。
- **Evidence**: EV-057、EV-059、EV-060、EV-061、EV-062
- **Links**: `causal:EK-35`；`contrast:EK-37`；`constraint:EK-34`
- **Note**: 测试代码覆盖 cosine/euclidean/None 三种契约；但本轮未执行，故只标 S3。

### EK-37 declarative 的阈值实现与 docstring 不一致，只做 `score >= threshold`
- **Layer**: L1 · **Type**: IMPLEMENTATION · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `_search_scored_declarative` 只做 `score_threshold is None or score >= score_threshold`，没有按 `higher_is_better` 自适应；`search_scored` 的上层 docstring 声称按方向过滤。由于 declarative 配置强制 reranker（higher-is-better），当前实际路径仍成立，但“lower-is-better”分支在 declarative 不可达。
- **Evidence**: EV-058、EV-021
- **Links**: `contrast:EK-36`；`constraint:EK-02`
- **Note**: 这是“声明与实现边界”，不是已验证 bug；若未来允许 declarative 无 reranker，会变成真 bug。

### EK-38 REST 边界把 `None` 阈值改写成 `-inf`，重新引入 LTM 已移除的 sentinel
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `SearchMemoriesSpec.score_threshold` 默认 `None`；`_search_target_memories` 在 `None` 时显式传 `-float("inf")`。LTM 的 event 路径在 euclidean 无 reranker 时会把 `-inf` 当作“高于阈值 → 丢弃”，导致空结果；declarative 路径因强制 reranker不受影响。
- **Evidence**: EV-055、EV-056、EV-057、EV-061
- **Links**: `contrast:EK-36`；`causal:EK-33`；`constraint:EK-42`
- **Note**: 测试 `test_memmachine_mock.py:506/584` 也直接传 `-inf`，说明该 sentinel 仍在调用层流传；这是强候选缺陷（C-01）。

### EK-39 retrieval agent 的 QueryPolicy 全硬编码，没有请求级可调预算
- **Layer**: L1 · **Type**: AGENT · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `QueryPolicy(token_cost=10, time_cost=10, accuracy_score=10, confidence_score=10, max_attempts=3, max_return_len=10000)` 在 `_run_retrieval_agent_long_term_search` 内构造，没有从请求或配置读取。
- **Evidence**: EV-063
- **Links**: `causal:EK-40`；`subsystem:EK-01`
- **Note**: 所有 agent 查询共享同一预算；无法按租户/场景区分成本与召回深度。

### EK-40 retrieval agent 结果被统一赋 `score=1.0`，真实分数丢失
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `_dedupe_and_score_agent_long_term_episodes` 把 agent 返回的全部 LTM episodes 赋 `score=1.0`，仅按 UID 与 short-term 去重；随后不再能按相关性排序。
- **Evidence**: EV-064
- **Links**: `causal:EK-39`；`contrast:EK-32`；`constraint:EK-36`
- **Note**: agent 模式的“分数”退化为存在标记；与 LTM 的 score 语义断裂。

### EK-41 `BOTH` 查询以 short-term UID 集合优先去重，LTM 再按分数排序
- **Layer**: L1 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `EpisodicMemory.query_memory` 在 BOTH 下并发查 STM/LTM；STM 结果先入，LTM 中与 STM UID 相同的条目被丢弃；LTM 剩下的按 score 排序。
- **Evidence**: EV-040、EV-066
- **Links**: `causal:EK-32`；`contrast:EK-33`
- **Note**: 同一 episode 若同时在 STM/LTM，会以 STM 的版本呈现（可能没有 score）。

### EK-42 `formalize_query_with_context` 把 summary 与 episode 按 `(created_at, uid)` 重排后拼进 query
- **Layer**: L1 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: 该方法按 `(created_at, uid)` 排序 summary + episodes，格式化为单一 query 字符串；顺序由时间戳与 uid 决定，而不是原始检索分数。
- **Evidence**: EV-067
- **Links**: `causal:EK-41`；`subsystem:EK-32`
- **Note**: 这会改变 reranker/LLM 看到的上下文顺序；时间近的不一定分数高。

### EK-43 `top_k` 默认 10、`expand_context` 默认 0，API 层有 `ge=1` 约束
- **Layer**: L1 · **Type**: CONTRACT · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `SearchMemoriesSpec.top_k` 默认 10 且 `ge=1`；`expand_context` 默认 0；`filter` 默认空串；`types` 可选。源码注释解释：无下界时 0/-1 会到达 retrieval，Qdrant 路径返回 500。
- **Evidence**: EV-068、EV-055
- **Links**: `constraint:EK-34`；`subsystem:EK-10`
- **Note**: 这是“在 schema 层拒绝无意义输入”的防御，而不是在 retrieval 层做 guard。

### EK-44 semantic 检索的分数来自各 set 迭代器，合并后无统一重排
- **Layer**: L1 · **Type**: MEMORY · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `SemanticService.search` 对多个 set_id 分别 embed 后并行 `_set_id_search`，用 `merge_async_iterators` 合并；没有跨 set 的全局 score 归一化或重排。
- **Evidence**: EV-053
- **Links**: `causal:EK-30`；`contrast:EK-40`
- **Note**: 多 set 场景下“前 10 条”可能被某一个 set 占满。

---

## D. 状态与存储治理（EK-45 ~ EK-51）

### EK-45 STM 用 deque + 摘要计数做容量控制，驱逐与摘要耦合
- **Layer**: L2 · **Type**: STATE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `deque` 存近期 episode；超容量 `_do_evict` 先弹出已被摘要的旧条目（`len(self._memory) > self._current_episode_count`），仍满则把全部当前 episode 交给 `ShortTermMemoryConsolidator.summarize` 并重置计数。
- **Evidence**: EV-034、EV-035
- **Links**: `causal:EK-46`；`subsystem:EK-17`；`dependency:EK-47`
- **Note**: “已摘要”是隐式状态，不是显式字段；计数漂移会导致驱逐不正确。

### EK-46 摘要失败是静默的：保留旧 summary，后台 worker 不暴露失败状态
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `_create_summary` 对 `ExternalServiceAPIError/ValueError/RuntimeError` 记录异常并返回旧 summary；consolidator 的异常只 `logger.exception`，不 crash、不重试、不标记失败。
- **Evidence**: EV-036、EV-037
- **Links**: `causal:EK-45`；`contrast:EK-28`（另一条静默丢失路径）
- **Note**: 用户看到的是“摘要还在”，但可能已经落后于最近对话；没有 staleness 指标。

### EK-47 实例缓存用 ref_count + max_lifetime + disposal queue 管理 EpisodicMemory 生命周期
- **Layer**: L2 · **Type**: STATE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `MemoryInstanceCache` 是 LRU + ref_count：`get` 加引用，`release` 减引用；`capacity==0` 且 ref_count==0 时入 disposal queue；janitor 异步 close；`evict` 跳过 ref_count>0，超过 `max_lifetime` 的入队；`erase` 在 ref_count>0 时抛 `RuntimeError`。
- **Evidence**: EV-049、EV-050
- **Links**: `dependency:EK-45`；`causal:EK-49`；`subsystem:EK-12`
- **Note**: 这是防止“删除正在使用的实例”的保守设计；代价是实例可能比配置存活更久。

### EK-48 session 删除是逻辑删除 + 可重试物理清理
- **Layer**: L1 · **Type**: STATE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: `delete_session` 把 status 改为 `delete` 并入队；worker 用 SQL lease 防止多 worker 重复；`asyncio.TaskGroup` 删 episode store / episodic / semantic；全部成功后 `manager.delete_session` 物理删行。
- **Evidence**: EV-042、EV-043
- **Links**: `mechanism:EK-31`；`causal:EK-49`；`causal:EK-52`
- **Note**: 删除返回时数据可能还在；调用方若立即重建同名 session，会看到旧数据或状态冲突（C-05）。

### EK-49 `TaskGroup` 让部分清理失败时取消其余子任务
- **Layer**: L1 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: B
- **Statement**: `_delete_queued_session_data` 用 `asyncio.TaskGroup` 并发删 episode/episodic/semantic；任一子任务抛错，TaskGroup 取消其余并抛出，session 行保留，等待下一次扫描重试。
- **Evidence**: EV-043
- **Links**: `contrast:EK-14`；`causal:EK-48`；`mechanism:EK-52`
- **Note**: 这是“全有或全无”的编排语义，但底层存储删除并非原子；“取消”不等于“回滚”。

### EK-50 event segment store 已从 per-tenant 分区改为共享表 + incarnation 注册表
- **Layer**: L2 · **Type**: ENGINEERING · **Strength**: S2 · **Epistemic**: Fact · **Value**: A
- **Statement**: `design/segment_store_shared_tables.md`（accepted 2026-08-31）把每租户一对子表改为共享 `segment_store_pt`（tenant registry + incarnation UUID）、`segment_store_sg`、`segment_store_dv_ln`、`segment_store_gc`（purge queue）；删除为 O(1) 改 registry 行 + 入 purge queue；purge 异步分批、cursor `purged_through`。
- **Evidence**: EV-074、设计文档（`design/segment_store_shared_tables.md:1-110`）
- **Links**: `subsystem:EK-04`；`causal:EK-31`；`contrast:EK-45`
- **Note**: 设计文档记录了 #1544（`DROP CASCADE` 破坏父级 FK）与 #1546（generic plan 锁 319 分区导致 shared memory 500）两个真实缺陷及修复；这是“分区 DDL 成本”被实测推翻的证据。

### EK-51 索引/原文漂移只能被动 warning + 丢弃，没有修复任务
- **Layer**: L2 · **Type**: FAILURE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: event 查询命中 vector index 的 episode，但 `episode_storage` 缺失时，代码 warning 并丢弃该结果；注释归因于 add/delete 的部分失败。没有对账、补偿或重建索引任务。
- **Evidence**: EV-033、EV-017
- **Links**: `dependency:EK-21`；`causal:EK-16`；`contrast:EK-28`
- **Note**: 这是部分写失败在读取侧的可见症状；它只保护“不返回坏数据”，不恢复“丢失的好数据”。


---

## E. 生命周期补充（EK-52）

### EK-52 `drop_session_partition` 之后 LTM 实例不可复用，必须重建
- **Layer**: L1 · **Type**: STATE · **Strength**: S3 · **Epistemic**: Fact · **Value**: A
- **Statement**: event backend 的 `drop_session_partition` 删除 VectorStore collection 与 SegmentStore partition 后，把 `EventMemory` 的 collection/segment handles 置 None；任何后续复用会 fail loudly（而不是静默写旧分区）。调用方必须丢弃该 `LongTermMemory` 实例并重新构建。
- **Evidence**: EV-092、EV-043
- **Links**: `causal:EK-48`（session 删除触发 drop）；`subsystem:EK-31`（同属删除生命周期）；`mechanism:EK-49`（同属“失败要显式暴露”）
- **Note**: 这是对 KO-08 “延迟清理/最终一致”的重要边界：清理完成后实例不是“可继续用”，而是“已作废”。

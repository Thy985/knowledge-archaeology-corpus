# 02 Engineering Knowledge — Mem0（EK Graph）

> 每条 EK 保留证据来源；links 声明 6 类边（mechanism/subsystem/causal/dependency/constraint/contrast）。层级：L1（工程事实）/L2（工程知识——因果解释）。

## A. 写入管道（add path）

### EK-01 V3 单遍 ADD-only 抽取管道（L2）
**核心机制**：`_add_to_vector_store` 六阶段：Phase 0 会话上下文（`db.get_last_messages(limit=10)` + `parse_messages`）→ Phase 1 现有记忆检索（`vector_store.search(top_k=10)`）→ Phase 2 **单次 LLM 抽取**（system=ADDITIVE_EXTRACTION_PROMPT + response_format json_object）→ Phase 3 批量 embed（`embed_batch`）→ Phase 4/5 逐条 hash 去重（md5）+ payload 构造 → Phase 6 批量 insert + batch_add_history。
**为什么**：单次 LLM 调用同时完成"抽取+去重决策"，把写入成本压到每轮 1 次 LLM 调用；ADD-only 语义（提示词明确"Your sole operation is ADD"）避免多轮 LLM 往返。
**证据**：main.py:879-1170（"V3 PHASED BATCH PIPELINE" 注释）；prompts.py:468 ADDITIVE_EXTRACTION_PROMPT（"Your sole operation is ADD"）
**注（Reconciliation ID-1）**：AsyncMemory.add 有同构 V3 管线（main.py:2580-2620），用 `asyncio.to_thread` 包装阻塞调用；uuid_mapping/ADDITIVE_PROMPT/AGENT_CONTEXT_SUFFIX 完全同构。
**links**：mechanism→EK-02, EK-09；causal→EK-17, EK-21；subsystem→EK-14

### EK-02 检索时 hash 去重（L1）
**核心机制**：Phase 1 取回 existing_results 的 payload.hash 建 `existing_hashes` 集合；Phase 4 对每条抽取文本算 `md5(text)`，命中 existing_hashes 或 batch 内 `seen_hashes` 即跳过（`logger.debug`）。
**为什么**：LLM 抽取可能重复产出已有事实；hash 去重是零成本（无嵌入调用）的第一道防重闸。
**证据**：main.py:920-930（existing_hashes）、936-945（seen_hashes + md5）
**links**：mechanism→EK-01, EK-07；contrast→EK-04（语义去重 vs 精确去重）

### EK-03 无事实时的消息兜底落库（L1）
**核心机制**：LLM 未抽取任何记忆时，仍 `self.db.save_messages(messages, session_scope)` 后 return []——原始消息进 SQLite（供未来检索/重抽），但向量库无新增。
**为什么**：消息本身是未来抽取的原料；避免"这次没抽到=永久丢失"。
**证据**：main.py:909-912（`if not extracted_memories: ... save_messages`）；storage.py:257 `save_messages`
**links**：causal→EK-01, EK-09；subsystem→EK-14

### EK-04 LLM 抽取失败重抛而非空列表（L2）
**核心机制**：Phase 2 的 LLM 调用异常 `raise LLMError(...)`，注释明确：原 `return []` 使调用方无法区分"LLM 不可用（429/5xx/timeout）"与"LLM 无事实可抽"——两者都表面为空列表。
**为什么**：诚实失败优先——可重试错误（LLM 挂）必须与正常空结果区分，否则用户静默丢记忆。
**证据**：main.py:896-903（`raise LLMError(f"LLM extraction failed: {e}") from e` + 注释）
**links**：mechanism→EK-06（诚实失败）；contrast→EK-03（空结果兜底）；causal→EK-17（LLM 异常体系）

### EK-05 UUID→整数映射防幻觉（L2）
**核心机制**：Phase 1 把 existing_results 的 UUID 映射为 `"0","1",...`（`uuid_mapping[str(idx)] = mem.id`）再喂给 LLM；LLM 输出的 linked_memory_ids 引用整数。
**为什么**：LLM 对长 UUID 易幻觉（编造不存在的 id）；整数序列消除幻觉面。
**证据**：main.py:873-878（`uuid_mapping`）
**links**：mechanism→EK-02（都是"LLM 输出不可信"防御）；subsystem→EK-01

### EK-06 嵌入与落库双 fallback 链（L1）
**核心机制**：Phase 3 `embed_batch` 异常→逐条 `embed`；Phase 6 `vector_store.insert` 异常→逐条 insert；history `batch_add_history` 异常→逐条 add。
**为什么**：批量接口失败不整批丢弃——降级为逐条，保证单条成功不牵连。
**证据**：main.py:915-926（embed fallback）、950-960（insert fallback）、970-975（history fallback）
**links**：mechanism→EK-18（都是"批量→单条"降级）；contrast→EK-04（重抛 vs 降级）

### EK-07 消息格式三态归一化（L1）
**核心机制**：`add()` 接受 str/dict/list[dict]；str→`[{"role":"user","content":str}]`，dict→`[dict]`，否则 VALIDATION_003。
**为什么**：降低调用方摩擦——OSS SDK 需要宽容输入面（相比平台 API 严格类型）。
**证据**：main.py:796-812（VALIDATION_003）
**links**：dependency→EK-01（归一化后进入管道）；contrast→client/main.py:274-281（客户端同样归一化）

### EK-08 procedural memory 显式类型（L1）
**核心机制**：`memory_type` 仅接受 `"procedural_memory"`（PROCEDURAL）；agent_id + procedural → `_create_procedural_memory`：system prompt（PROCEDURAL_MEMORY_SYSTEM_PROMPT）+ messages + "Create procedural memory" user 尾缀 → LLM 摘要 → embed → `_create_memory`，payload 带 `memory_type: "procedural_memory"`。
**为什么**：procedural（agent 工作流程/指令类）记忆与对话事实记忆走不同提示词与语义。
**证据**：main.py:814-816（VALIDATION_002）、1993-2037
**links**：contrast→EK-01（ADD-only 抽取 vs 摘要式）；subsystem→EK-01

## B. 检索管道（search path）

### EK-09 混合检索三信号融合（L2）
**核心机制**：`_search_vector_store`：semantic over-fetch（`internal_limit = max(limit*4, 60)`）→ keyword_search（BM25）→ entity boost → 候选集（过期过滤）→ `score_and_rank`：`combined = (semantic + bm25 + entity) / max_possible`，max_possible 依活跃信号 1.0/2.0/2.5/1.5。
**为什么**：单一向量检索漏词（无嵌入命中）时，BM25 补关键词面、实体 boost 补实体关系面；三信号加性融合。
**证据**：main.py:1628-1727；scoring.py:60-130
**links**：mechanism→EK-10, EK-11；causal→EK-12；subsystem→EK-16

### EK-10 BM25 查询长度自适应 sigmoid 归一化（L2）
**核心机制**：`get_bm25_params`：query 词数 ≤3→(5.0,0.7)；≤6→(7.0,0.6)；≤9→(9.0,0.5)；≤15→(10.0,0.5)；>15→(12.0,0.5)；`normalize_bm25 = 1/(1+exp(-steepness*(raw-midpoint)))` 把无界 BM25 压到 [0,1]。
**为什么**：长查询原始 BM25 分偏高——sigmoid 中点随长度上移，使不同长度查询的 BM25 分数可比较（与语义分融合的前提）。
**证据**：scoring.py:16-55
**links**：causal→EK-09（BM25 信号来源）；constraint→EK-09（归一化是融合前置）

### EK-11 实体 boost（L2）
**核心机制**：`_compute_entity_boosts`：query 实体去重（max 8）→ `embed_batch` → entity_store.search(top_k=500, 相似度≥0.5) → boost = similarity × ENTITY_BOOST_WEIGHT(0.5) × memory_count_weight（`1/(1+0.001×(n-1)²)`，n=linked 记忆数）；ThreadPoolExecutor(4) 并发。
**为什么**：实体关系是记忆检索的高价值信号（人名/产品名精确命中）；memory_count_weight 抑制"过度链接的实体"稀释 boost。
**证据**：main.py:1733-1810；scoring.py:57（ENTITY_BOOST_WEIGHT）
**links**：mechanism→EK-13（实体提取）；causal→EK-09；subsystem→EK-16

### EK-12 threshold 前置门控（L2）
**核心机制**：`score_and_rank` 中 `if semantic_score < threshold: continue` **先于** BM25/entity 加性融合——低于语义阈值即使 BM25/实体可提升也排除。
**为什么**：语义是主信号；关键词/实体是辅助——防止"语义不相关但碰巧 BM25 命中"的噪声进入。
**证据**：scoring.py:96-99（threshold gate 注释 "before combining"）
**links**：constraint→EK-09；contrast→EK-04（门控 vs 重抛）

### EK-13 spaCy 实体提取三类 + 通用头过滤（L2）
**核心机制**：`extract_entities`：PROPER（大写专名序列）/QUOTED（引号文本）/TOPIC（名词复合）/IDENTIFIER 四类；`_GENERIC_HEADS`（thing/stuff/way/time...）过滤泛化实体头。
**为什么**：实体质量决定 boost 质量——"thing/way"这类泛词作实体头会污染实体库。
**证据**：utils/entity_extraction.py:1-80（docstring + _GENERIC_HEADS）、751-761
**links**：causal→EK-11；subsystem→EK-16

### EK-14 高级元数据过滤操作符（L1）
**核心机制**：search filters 支持 eq/ne/in/nin/gt/gte/lt/lte/contains/icontains/*/AND/OR/NOT（`_process_metadata_filters` + `_has_advanced_operators` 探测），运算符键处理后从 effective_filters 移除。
**为什么**：记忆元数据检索需要结构化查询能力，超出简单等值匹配。
**证据**：main.py:1524-1627（docstring 列出全部操作符）；search docstring 1384-1404
**links**：subsystem→EK-09；causal→EK-15

### EK-15 检索参数验证链（L1）
**核心机制**：search 前 `_validate_search_params`（threshold/top_k 合法性）+ `_validate_and_trim_search_query`（query 清理）+ `_reject_top_level_entity_params`（拒绝顶层 user_id/agent_id/run_id——必须走 filters）+ `_validate_and_trim_entity_id`。
**为什么**：API 面一致性——add 顶层参数 vs search filters 参数的分裂是 v2 历史遗留，验证链强制用户走 filters，防止误用。
**证据**：main.py:165-260、1379-1441
**links**：constraint→EK-09；causal→EK-14

### EK-16 检索结果格式化与提升键（L1）
**核心机制**：`_search_vector_store` Step 9：promoted_keys（user_id/agent_id/run_id/actor_id/role/attributed_to/expiration_date）提升到顶层；其余 metadata 进 `metadata` dict；无 `data` 的候选跳过。
**为什么**：scope 键对调用方是一等公民（过滤依据），业务元数据是二等——契约分层。
**证据**：main.py:1697-1727
**links**：dependency→EK-09；subsystem→EK-14

## C. 更新/删除路径

### EK-17 身份键不可变（L1）
**核心机制**：`update()` 的 metadata 中 user_id/agent_id/run_id/actor_id 被 `_strip_identity_keys` 忽略——更新不可改 scope；created_at 保留，updated_at 刷新，hash 重算，text_lemmatized 重算。
**为什么**：scope 是记忆的归属契约，改 scope = 换记忆；时间戳语义分离（创建/更新）。
**证据**：main.py:2050-2072（`_strip_identity_keys`）；utils:143-164
**links**：constraint→EK-14；mechanism→EK-19（删除同样走 payload 清理）

### EK-18 更新/删除时实体库清理重链接（L1）
**核心机制**：`_update_memory` 文本变化时 `_remove_memory_from_entity_store` + `_link_entities_for_memory`；`_delete_memory` 删除后同样清理；helper 吞错（非致命）。
**为什么**：实体库与向量库的一致性——记忆文本变了，旧的实体链接必须失效。
**证据**：main.py:2082-2095、2116-2126
**links**：mechanism→EK-11；dependency→EK-19

### EK-19 delete_all 分批循环 + 重复批次防护（L2）
**核心机制**：`delete_all` 循环 `vector_store.list(top_k=DELETE_ALL_BATCH_SIZE)` → 删批次 → 直到空；`seen_batches` 集合检测重复批次（多数 store list 上限 100，静默截断会让删除漏项）。
**为什么**：list 分页截断是删除类操作的经典漏删点；重复批次检测防死循环。
**证据**：main.py:1890-1944（注释 "Most vector stores cap list() at 100 results"）
**links**：mechanism→EK-06（都是批量健壮性）；causal→EK-17

### EK-20 update 的后备存储失败区分（L1）
**核心机制**：`_update_memory` 中 `vector_store.get` 异常→重抛（REST 层映射 5xx 而非 4xx）；get 返回 None → ValueError（4xx 语义）。
**为什么**：存储故障 ≠ 用户请求错误——错误面映射必须区分，否则调用方误判。
**证据**：main.py:2040-2050（注释 "Backing-store failure, not a bad memory_id"）
**links**：mechanism→EK-04（诚实失败）；causal→EK-17

## D. 平台化基础设施

### EK-21 telemetry 采样分层（L2）
**核心机制**：`MEM0_TELEMETRY_SAMPLE_RATE`（默认 0.1）；非 lifecycle 事件随机采样，lifecycle 事件 100% 必发；`$identify` 永不丢失（PostHog person merging）。
**为什么**：热路径事件（add/search）降采样控制成本，生命周期事件（init）与用户身份关联必须全量。
**证据**：telemetry.py:30-72；tests/test_telemetry_sampling.py
**links**：mechanism→EK-22（都是平台信号）；subsystem→EK-24

### EK-22 notices 远程配置提示系统（L2）
**核心机制**：`oss_notices_config.json`（GitHub raw URL，1h TTL + bundled fallback + variant_split 0.5 A/B）；StaticFlagResult 替代 PostHog evaluate_flags（无 PostHog 降级）；七类 notice（first_run/temporal_stub/temporal_usage/decay_stub/decay_usage/scale_threshold/performance_slow_query），当前全部 enabled:false。
**为什么**：SDK 可以远程开关行为引导提示（如"达到规模阈值，建议升级平台"）而不发版。
**证据**：notices.py:1-80、oss_notices_config.json（全部 disabled）
**links**：mechanism→EK-21；subsystem→EK-24；causal→EK-23

### EK-23 OSS 功能桩 = 平台导流面（L2）
**核心机制**：`timestamp`（add）/`reference_date`（search）在 OSS 直接 `raise ValueError(get_temporal_feature_error_message(...))`——时域功能是平台特性，OSS 显式拒绝而非静默忽略。
**为什么**：OSS 与平台功能差别的显式契约——失败比误导好；同时是平台价值主张的信号。
**证据**：main.py:775-776、1411-1413（temporal error）
**links**：mechanism→EK-22；causal→EK-24

### EK-24 MemoryClient 匿名身份 + 别名关联（L2）
**核心机制**：`user_id = md5(api_key)`（匿名）；`_validate_api_key()` 校验拿 user_email；`_maybe_alias_anon_to_email` 把匿名事件关联到邮箱（PostHog alias）。
**为什么**：免费层用户无显式身份，SDK 用 API key 哈希做匿名 ID 并在认证后自动合并到真实身份——遥测连续性的产品化设计。
**证据**：client/main.py:219-245
**links**：mechanism→EK-21；subsystem→EK-23

### EK-25 遥测迁移信号（L2）
**核心机制**：`Memory.__init__` 创建 `_telemetry_vector_store`：collection_name 强制 `"mem0migrations"` + faiss/qdrant 用 `migrations_*` 路径——遥测层记录"迁移"活动。
**为什么**：OSS→平台迁移是核心转化漏斗，SDK 在遥测层专门建"迁移"命名空间。
**证据**：main.py:515-545；tests/test_oss_to_platform_migrate.py
**links**：mechanism→EK-22, EK-24；subsystem→EK-21

### EK-26 server JWT 双 token + 防时序枚举（L2）
**核心机制**：JWT HS256（access 30min + refresh 30day）+ bcrypt；登录失败走 `dummy_verify_password()`（烧同样 bcrypt 周期）——**不泄露邮箱是否存在**。
**为什么**：登录时序侧信道（枚举邮箱）是认证系统的经典漏洞。
**证据**：server/auth.py:24-38（dummy_verify 注释）
**links**：mechanism→EK-27；subsystem→server

### EK-27 server API key 前缀 + 哈希存储（L1）
**核心机制**：`generate_api_key`：`secrets.token_urlsafe(32)` → `m0sk_{raw}`；返回 (full_key, prefix[:12], bcrypt hash)；校验时按 prefix 查 → bcrypt verify。
**为什么**：密钥不落明文（bcrypt）；前缀检索避免全表扫描 + 泄露面控制。
**证据**：server/auth.py:40-52
**links**：mechanism→EK-26；causal→EK-28

### EK-28 server 治理面（L1）
**核心机制**：slowapi rate limit（IP）+ 请求日志中间件（`_redact_config` 脱敏 + `_should_log_request` 选择性记录）+ ADMIN_API_KEY + AUTH_DISABLED 逃生阀 + alembic 迁移。
**为什么**：自托管多租户需要限流/审计/配置脱敏；逃生阀让部署者在认证配置错误时可恢复。
**证据**：server/main.py:66-340；server/rate_limit.py
**links**：subsystem→EK-26, EK-27

## E. 工程韧性

### EK-29 chatty LLM JSON 解析 fallback 链（L2）
**核心机制**：`remove_code_blocks` → `json.loads(strict=False)` → 失败 `extract_json`（`{`/`}` 边界搜索 + re.search code block）→ 仍失败 `[]`。
**为什么**：本地 LLM（LM Studio/Ollama）常把 JSON 包在对话文本/代码块里——解析必须容错，否则抽取整体失败。
**证据**：main.py:905-916；tests/test_chatty_llm_parsing.py（6 用例覆盖纯 JSON/代码块/对话包装）
**links**：mechanism→EK-05（LLM 输出不可信）；dependency→EK-01（Phase 2 消费）
**links**：causal→EK-01

### EK-30 keyword_search 能力检测降级（L1）
**核心机制**：`getattr(type(self.vector_store), "keyword_search", None) is VectorStoreBase.keyword_search` → 不支持则 warning：混合（BM25）评分禁用，仅语义检索。
**为什么**：28 个向量库适配器能力参差——运行期检测 + 显式降级警告，而非文档假设。
**证据**：main.py:548-556（warning 注释）
**links**：constraint→EK-09（BM25 可用性约束融合）；contrast→EK-12

### EK-31 异常层次（L1）
**核心机制**：`MemoryError` 基类 + 14 子类：Authentication/RateLimit/Validation/MemoryNotFound/Network/Configuration/MemoryQuotaExceeded/MemoryCorruption/VectorSearch/Cache/VectorStore/Embedding/LLM/Database/Dependency。
**为什么**：调用方按错误面精确处理（重试/降级/用户提示）；平台客户端错误映射到 HTTP 语义。
**证据**：mem0/exceptions.py:34-386
**links**：subsystem→EK-04, EK-20

### EK-32 _safe_deepcopy_config（L1）
**核心机制**：telemetry 配置深拷贝带异常处理（thread lock 对象 deepcopy 失败→降级手工拷贝属性清单）。
**为什么**：pydantic/线程锁对象 deepcopy 会抛错——遥测不应因配置拷贝崩溃。
**证据**：main.py:270-300（`_safe_deepcopy_config`）
**links**：mechanism→EK-21

### EK-33 集成插件的双入口（L1）
**核心机制**：16 集成统一模式：每个插件实现"extract memory from conversation"回调 + 平台 API 调用；mem0-strands（多 agent 记忆流）+ agent-plugin-core（agent 核心插件）+ 各 editor 插件。
**为什么**：记忆层要嵌入 agent 运行时（harness 回调），不是独立调用——集成是分发渠道。
**证据**：integrations/ 目录；agent-plugin-core/
**links**：mechanism→EK-01（抽取语义复用）；subsystem→EK-24

### EK-34 AGENTS.md 治理双门（L2）
**核心机制**：PR 必须（a）签 CLA（否则不审）（b）链接 `accepted` label issue（bot 1 分钟关闭违反者）；`.github/workflows/` 修改需维护者批准（publishing 凭据按 workflow 文件名 pin）；核心依赖禁随意加（optional group）。
**为什么**：开源仓库的供应侧治理——防恶意 PR（凭据窃取）与依赖投毒；bot 强制流程。
**证据**：AGENTS.md
**links**：constraint→EK-33（集成贡献也要过门）；subsystem→governance

### EK-35 双时间戳契约（L1）
**核心机制**：created_at（不可变）/updated_at（每次更新刷新）分离；`_normalize_iso_timestamp_to_utc` 保证删除历史的时间戳规范化。
**为什么**：审计/历史还原需要原始创建时间。
**证据**：main.py:301-310、2062-2066
**links**：constraint→EK-17

### EK-36 视觉消息解析（多模态记忆输入）（L1）
**核心机制**：`add()` 中 `if self.config.llm.config.get("enable_vision"): messages = parse_vision_messages(messages, self.llm, self.config.llm.config.get("vision_details"))`，否则 `parse_vision_messages(messages)`（无 LLM 轻量解析）；视觉消息 → 文本描述 → 进抽取管道。
**为什么**：多模态对话（图片/截图）是 agent 记忆的重要输入面；enable_vision 是配置门（默认关），视觉理解由 LLM 完成。
**证据**：main.py:864-867（sync）、2518-2519（async）
**links**：mechanism→EK-01（输入预处理）；dependency→EK-01（Phase 0 前处理）；subsystem→memory.main

## D 级（项目局部，保留）
- EK-D1 `_build_session_scope`（main.py:412）把 filters 编成 SQLite session_scope 字符串（隔离 messages 表）。
- EK-D2 `_entity_collection_name`（main.py:422）实体库 collection 命名按 provider+collection 派生。
- EK-D3 `generate_additive_extraction_prompt` 的 PAST_MESSAGE_TRUNCATION_LIMIT=300（prompts.py:1037）控制历史消息截断。
- EK-D4 `_escape_scope_value`（main.py:407）scope 值转义（防注入 SQLite scope 串）。

## EK 图统计
- EK 总数：39（35 活跃 + 4 D 级）
- 边数：≥85（含 mechanism 13 / subsystem 16 / causal 15 / dependency 7 / constraint 8 / contrast 5，多重计数）
- 游离 EK：0（全部 ≥1 边）

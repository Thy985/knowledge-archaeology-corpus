# 04 Flow Atlas — Mem0（七类流）

> 每条 Edge 必须可回溯到实际 symbol/file/condition/state transition。基于 `a39a802` 真实代码导出，非架构想象。

## F-1 Control Flow（add → 检索）
```
Memory.add(main.py:760)
  ├─ timestamp != None → raise ValueError (main.py:775-776) [OSS 桩]
  ├─ _normalize_expiration_date + _build_filters_and_metadata (777-782)
  ├─ memory_type 非 None 且 != PROCEDURAL → VALIDATION_002 (790-800)
  ├─ messages 归一化 str/dict/list (796-812)
  ├─ agent_id + procedural → _create_procedural_memory (815-823) [短路]
  └─ _add_to_vector_store (main.py:879)
       ├─ infer=False → 逐条 embed+_create_memory (881-907) [无 LLM 路径]
       └─ infer=True → V3 PHASED BATCH PIPELINE
            Phase 0: db.get_last_messages(limit=10) + parse_messages (886-888)
            Phase 1: vector_store.search(top_k=10) + uuid_mapping (891-899)
            Phase 2: llm.generate_response(ADDITIVE...) → LLMError 重抛 (901-917)
            Phase 3: embed_batch → fallback 逐条 (918-927)
            Phase 4-5: md5 hash 去重 + payload 构造 (928-948)
            Phase 6: vector_store.insert → fallback 逐条 + batch_add_history (950-976)
```
**关键 Edge**: `add → _add_to_vector_store`（main.py:837）；`infer=False` 分支跳过 LLM（主流程唯一非 LLM 写入路径）；procedural 短路（agent 专属）。

## F-2 State Flow（记忆生命周期）
```
[不存在] ──add→ ADD ──update(text changed)──→ UPDATE
                        └──delete──→ DELETE (is_deleted=1)
                              └──get_all(show_expired=False)──→ 过滤 _payload_is_expired (main.py:442)
状态字段: created_at(不可变) / updated_at(每次变更刷新) / hash(md5(text)) / is_deleted
状态存储: 向量库 payload（data/hash/text_lemmatized/scope/expiration）+ SQLite history（事件流水）
```
**关键 Edge**: `update 不改 created_at`（main.py:2064）；`expiration_date 可传 None 清除`（update docstring 1817-1830）；`delete 前 vector_store.get None → ValueError`（main.py:1873-1875）。

## F-3 Data Flow（对话 → 记忆）
```
messages(str/dict/list) ──parse_messages──→ parsed(role+content)
  ──embed(parsed,"search")──→ query_embedding
  ──LLM 抽取──→ extracted_memories[{text, attributed_to, linked_memory_ids}]
  ──embed_batch(texts)──→ vectors
  ──md5(text)──→ hash ──uuid4──→ memory_id
  ──payload{data, text_lemmatized, hash, created_at, updated_at, scope..., metadata}──→ vector_store.insert
  ──history_records──→ db.batch_add_history
检索侧: query ──lemmatize_for_bm25 + extract_entities──→ (lemmatized, entities)
  ──embed(query,"search")──→ embeddings ──vector_store.search(top_k=max(limit*4,60))──→ semantic
  ──keyword_search(lemmatized)──→ BM25 scores
  ──entity_store.search(top_k=500)──→ boosts
  ──score_and_rank──→ scored ──格式化(提升键)──→ results
```
**关键 Edge**: `_payload_is_expired` 在候选集构建时过滤（main.py:1648-1650）；`无 data 的候选跳过`（1713-1715）。

## F-4 Evidence Flow（抽取证据链）
```
LLM response(string) ──remove_code_blocks──→ clean
  ──json.loads(strict=False)──→ success? ──是──→ .get("memory", [])
        └──失败──→ extract_json（{} 边界 / code block re.search）──→ json.loads
  ──仍失败──→ [] （保存消息后 return）
证据落点: SQLite history（old/new/event/actor/role）+ 向量 payload hash
检索证据: explain=True → score_details{semantic,bm25,entity,raw,max_possible,final,threshold} (scoring.py:112-120)
```
**关键 Edge**: chatty LLM fallback 链（main.py:905-916 + test_chatty_llm_parsing.py 6 用例）；`explain` 透传 score_details（main.py:1722-1724）。

## F-5 Authority Flow（权限/权威边界）
```
OSS SDK: 无内置认证 —— 信任边界 = 调用方进程；scope 隔离仅靠 filters 契约（user_id/agent_id/run_id）
  └─ _reject_top_level_entity_params: search/get_all 拒绝顶层实体参数，强制 filters (main.py:165)
Server (自托管): 
  API key(m0sk_ prefix + bcrypt) ──verify_api_key_hash──→ 身份
  JWT(HS256, access 30min/refresh 30day) ──Depends(verify_auth)──→ 受保护端点
  ADMIN_API_KEY ──require_admin──→ /configure 写操作
  AUTH_DISABLED env → 绕过全部认证 [逃生阀]
MemoryClient (平台): api_key → user_id=md5(api_key) → _validate_api_key → user_email
```
**关键 Edge**: dummy_verify_password 防邮箱枚举时序（server/auth.py:28-31）；`/configure` POST 需 require_admin（server/main.py:333-338）。

## F-6 Memory Flow（记忆系统的记忆——多级存储）
```
层级 1: SQLiteManager (storage.py) —— messages 表（原始对话, session_scope 隔离）+ history 表（事件流水）
层级 2: vector_store —— 记忆本体（payload: data/hash/text_lemmatized/metadata）
层级 3: entity_store —— 实体索引（spaCy 提取 → linked_memory_ids 关联记忆）
读取路径: search → vector_store.search + keyword_search + entity_store.search → score_and_rank
写入路径: add → vector_store.insert + db.batch_add_history + (_link_entities_for_memory)
维护路径: update/delete → 实体清理重链接（EK-18）+ history 事件
```
**关键 Edge**: entity_store 延迟初始化（main.py:559-560 `_entity_store = None` 首次使用创建）；`_remove_memory_from_entity_store` 吞错非致命（main.py:652）。

## F-7 Policy Flow（治理闭环）
```
Decision → Approval → Policy → Enforcement → Future Decision
决策: 贡献规则（CLA/accepted issue 双门）+ 凭据 pin（AGENTS.md）
批准: PR 必须链接 accepted label issue；.github/workflows 修改需维护者批准
政策固化: CLA 协议 + AGENTS.md 文档化
执行: bot 1 分钟关闭违反 PR（流程自动化）
运行时策略: notices 远程配置（GitHub raw oss_notices_config.json, TTL 1h, A/B variant_split）
  └─ 当前全部 disabled（策略待激活态）
遥测策略: 采样率 0.1 默认 / MEM0_TELEMETRY_SAMPLE_RATE env 覆盖 / lifecycle 100%
未来决策: 平台功能桩（temporal）作为 OSS→平台转化的政策信号
```
**关键 Edge**: `_REQUIRED_FLOW`：notices 拉取失败→bundled fallback（notices.py:30-40）；`StaticFlagResult` 替代 PostHog（无 PostHog 环境降级）。

## 七流交叉校验
- F-1/F-2/F-3 一致：add 管道产出 ADD 事件与向量记录（无矛盾）
- F-4 与 F-3 一致：explain 的 score_details 来自 scoring.py 同一函数
- F-5 与 F-1 一致：OSS 无认证与 server 认证是两条独立 Authority 路径（无交叉）
- F-6 与 F-2 一致：delete 同时写 history(is_deleted=1) 与向量删除 + 实体清理
- F-7 与 F-5 一致：server 的 AUTH_DISABLED 是治理逃生阀（政策允许的降级路径）

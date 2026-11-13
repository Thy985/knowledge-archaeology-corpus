# Independent Validation — 盲重建审计报告

> **角色隔离声明**：本轮只有单一执行环境，没有第二个独立模型进程。本报告以“Independent Auditor”角色执行：先独立重读仓库并写下 Findings，再与 `package/` 对照。它降低了同源自我确认，但不能消除同一上下文偏置；不宣称“完全独立”。
> **禁止修改原考古产物**：本报告只记录判定，不改 `00~05` 与 `ek.md`。修正统一在阶段⑥ Reconciliation 中落到 Corpus。

审计对象：MemMachine @ `377510d436dc81c3651502f2d5e36ff245650ce7`
审计时间：2026-11-14
方法：独立重读 `main/memmachine.py`、`episodic_memory/*`、`semantic_memory/*`、`server/api_v2/*`、`common/api/spec.py`、storage 实现、设计文档；先写 Independent Findings，再对照 Package。

---

## 0. Independent Findings（先于对照）

| IF | 独立发现 | 证据 |
|---|---|---|
| IF-01 | 顶层写入是条件式多路：episode_storage 总是；episodic/semantic 由 `target_memories` 决定 | `main/memmachine.py:778-879` |
| IF-02 | 顶层 `gather(return_exceptions=True)` 后统一抛错；无 write-intent | 同上，TODO #1738 |
| IF-03 | 内层 `EpisodicMemory.add_memory_episodes` 的 gather 无 `return_exceptions` | `episodic_memory.py:209-244` |
| IF-04 | declarative 写入节点后写边；边失败留孤立 derivative | `declarative_memory.py:99-212` |
| IF-05 | event 五阶段；segment store 先于 vector | `event_memory.py:236-292` |
| IF-06 | declarative 检索 derivative→episode→邻域；event 检索 segment→窗口 | `declarative_memory.py:313-500`、`event_memory.py:398-535` |
| IF-07 | `expand_context` 在 event 是 segment 窗口；在 declarative 是 episode 邻域 | `long_term_memory.py:327-357` |
| IF-08 | `-inf` 在 lower-is-better 下会丢弃全部；默认 `SimilarityMetric=COSINE` | `long_term_memory.py:462-475`、`common/vector_store/data_types.py:32` |
| IF-09 | REST 无认证依赖，仅 access log/metrics middleware | `server/app.py:70-110`、`router.py` |
| IF-10 | MCP header 缺省空串 | `mcp.py:257-300` |
| IF-11 | semantic ingestion 5 条/批、50 feature 上下文、ADD/DELETE | `semantic_ingestion.py:83-104,229,299-347` |
| IF-12 | consolidation 先删后加；citation 全量挂到每个新 feature | `semantic_ingestion.py:499-542` |
| IF-13 | PG citation 关联表有 `ForeignKey("feature.id", ondelete="CASCADE")` | `sqlalchemy_pgvector_semantic.py:67-79` |
| IF-14 | Neo4j `delete_features` 用 `DETACH DELETE f` | `neo4j_semantic_storage.py:448-458` |
| IF-15 | session 删除：status→queue→lease→TaskGroup | `main/memmachine.py:350-440,652-674` |
| IF-16 | instance cache refcount + disposal queue + max_lifetime | `episodic_memory/instance_lru_cache.py:20-150` |
| IF-17 | STM eviction 用 `_current_episode_count`，不是显式“已摘要”字段 | `short_term_memory.py:224-243` |
| IF-18 | `drop_session_partition` 后 LTM 实例不可再用，需重建 | `long_term_memory.py:417-440` |

---

## 1. 判定统计

| 判定 | 数量 | 对象 |
|---|---:|---|
| CONFIRMED | 13 | EK-01/02/03/04/05/06/08/10/13/14*（条件修正后）/17/18/21/31/47 |
| PARTIALLY_CONFIRMED | 3 | EK-20、EK-45、KO-02（表述收窄） |
| DOWNGRADED | 1 | KO-05（scope 收窄到直连部署） |
| OVER_GENERALIZED | 2 | EK-14（“三路总是”）、EK-20（“任何后段失败”） |
| MISSING | 5 | 见 §4 |
| CONTRADICTED | 1 | C-06（citation 孤儿） |
| NEEDS_HUMAN_REVIEW | 2 | README 拓扑、multi-worker 行为 |

> 说明：`CONFIRMED` 指“独立重读后事实成立”，不等于 S4/S5；证据强度仍是 S3。

---

## 2. 三个成功判定（独立重读后成立）

### S-01 多存储写入没有补偿（EK-14/15/16）
- 独立重读 `add_episodes`：`episode_storage` 总是提交；episodic/semantic 由 `target_memories` 条件提交；`gather(return_exceptions=True)` 后统一 `_raise_add_episode_errors`；TODO #1738 明确“partial failures ... detected and repaired”。
- 判定：**CONFIRMED**。唯一修正是“不是永远三路，而是 1~3 路条件组合”；“无补偿”成立。

### S-02 两个 backend 的上下文单位不等价（EK-34）
- 独立重读 `long_term_memory._search_scored_event`：注释明确 `expand_context` 是 segment window，且 `max(0, min(expand_context, num_episodes_limit - 1))`；declarative 走 `max_num_episodes/expand_context` 的 episode 邻域。
- 判定：**CONFIRMED**。这是本轮最有价值的“接口同、语义不同”证据。

### S-03 配置缺依赖会整层关闭（EK-08/09）
- 独立重读 `_resolve_ltm_defaults` 与 `_maybe_disable_semantic_memory`：缺 embedder/reranker/vector_graph_store（declarative）或缺 vector_store/segment_store（event）→ `_disable_long_term_memory`；OpenAI key 空且无 base_url → semantic disabled。
- 判定：**CONFIRMED**（限 LTM；STM 可独立存活，已在 KO-03 scope 收窄）。

---

## 3. 三个错误判定（独立重读后推翻/降级）

### E-01 **CONTRADICTED**：C-06 “PG delete_features 留下 citation 孤儿”
- Package 曾写：PG `delete_features` 只删 Feature，不删 citation，形成孤儿证据。
- 独立重读 `citation_association_table`（`sqlalchemy_pgvector_semantic.py:67-79`）：
  ```python
  Column("feature_id", Integer,
         ForeignKey("feature.id", ondelete="CASCADE", onupdate="CASCADE"),
         primary_key=True)
  ```
  数据库层 `ON DELETE CASCADE` 会在 `DELETE FROM feature` 时删除关联行；SQLAlchemy 的 bulk `delete(Feature)` 会触发该 FK 级联（PostgreSQL 语义）。
- 判定：**CONTRADICTED**。C-06 不能作为缺陷；只能保留为“若后端不支持 FK 级联才成立”的条件性假设。
- 影响：`05_candidates` 的 C-06 必须在 Reconciliation 中降级；`run_metadata` 的“strong candidate defect”需修正。

### E-02 **OVER_GENERALIZED**：EK-14 “三路并行”是条件式，不是总是
- Package 写“同时写 episode_storage、episodic_memory、semantic_session_manager”。
- 独立重读：`tasks` 初始只有 `episode_storage.add_episodes`；只有 `MemoryType.Episodic in target_memories` 才追加 episodic；只有 `_should_dispatch_to_semantic_memory` 才追加 semantic。
- 判定：**OVER_GENERALIZED**。修正为“1~3 路条件并行”；“无补偿”仍成立。
- 影响：KO-02 的因果链仍成立，但“三路”改为“多路”。

### E-03 **OVER_GENERALIZED**：EK-20 “任何后段失败都留下前段产物”
- Package 写“任何后段失败都留下前段产物”。
- 独立重读：五阶段中 embed 失败时 `add_segments` 尚未执行，不会留下 segment；`add_segments` 失败时 vector 未写；只有 `add_segments` 成功、`upsert` 失败时才会“segment 已存在但不可检索”。因此不是“任何后段”。
- 判定：**OVER_GENERALIZED**。修正为“一旦 `add_segments` 成功而后续 upsert 失败，会留下不可检索的 segment；更早阶段失败不会留下后段产物”。
- 影响：EK-21 仍成立（写顺序 segment 先于 vector），但失败窗口描述需收窄。

### 附加降级（非“错误”，但需收窄）
- **EK-45 PARTIALLY_CONFIRMED**：代码用 `len(self._memory) > self._current_episode_count` 弹出，`_current_episode_count` 是“上次 consolidation 后新增计数”，不是显式“已摘要”标记；说“弹出已摘要旧条目”是合理解释，但应标 Observation。
- **KO-05 DOWNGRADED**：不变量成立，但只对“REST/MCP 直连且无前置网关”的部署构成风险；scope 必须写明。

---

## 4. 遗漏（MISSING）

| M-ID | 遗漏 | 为什么重要 | 应补到 |
|---|---|---|---|
| M-01 | `citation_association_table` 的 `ON DELETE CASCADE` | 直接推翻 C-06；说明 PG 删除语义 | EK/候选修正 |
| M-02 | `drop_session_partition()` 后 `EventMemory` 句柄失效、LTM 实例不可再用 | 是删除路径的重要状态机，Package 只在 EK-31 提“最终一致”，未写“实例不可复用” | EK-52 或 KO-08 补边 |
| M-03 | `delete_history` 会按 history_id 清 citation | 是 citation 生命周期的另一条路径；缺它会导致“citation 永不删”的误判 | Evidence Flow |
| M-04 | `test_memmachine_mock.py` 的实际路径 `packages/server/server_tests/...` | Package 曾写 `packages/server/tests/...`，路径错误会影响复核 | Evidence index |
| M-05 | API `top_k` 的 `ge=1` 注释解释 Qdrant 500 | 是“schema 层防御”的完整动机，Package 只写约束未写 500 证据 | EK-43 |

---

## 5. 遗漏的 Flow Edge

| FE-ID | 缺失 Edge | 应存在的位置 |
|---|---|---|
| FE-01 | `drop_session_partition → EventMemory handles=None → 后续调用 RuntimeError` | State Flow |
| FE-02 | `delete_history → citation 关联行删除` | Evidence Flow |
| FE-03 | `Feature DELETE → FK CASCADE → citation 行删除`（PG） | Evidence Flow |
| FE-04 | `delete_features → Neo4j DETACH DELETE（属性随节点删）` | Evidence Flow |

---

## 6. 对 KO 的逐条判定

| KO | 判定 | 理由 |
|---|---|---|
| KO-01 | CONFIRMED（scope 收窄） | 原文+索引+回填在两条 backend 都成立；event 的“原文”是 EpisodeStorage，不是 segment |
| KO-02 | PARTIALLY_CONFIRMED | “无补偿”成立；“三路”改为“多路条件”；重试可能重复的反例保留 |
| KO-03 | CONFIRMED | LTM 全有或全无；STM 可存活已写入 scope |
| KO-04 | CONFIRMED | 单位差异有代码注释与实现双重证据 |
| KO-05 | DOWNGRADED | 只在无网关直连部署构成风险；不是所有部署 |
| KO-06 | CONFIRMED | 两阶段结构成立；semantic 可关闭已写入 scope |
| KO-07 | CONFIRMED | 先删后加 + citation 全量挂载成立；C-06 的 PG 孤儿不成立，但“过度归属”仍成立 |
| KO-08 | CONFIRMED（补 M-02） | refcount/延迟清理/最终一致成立；需补“drop 后实例不可复用” |

---

## 7. Benchmark Case（供 skill CI 使用）

### Case ID: KA-BENCH-MEMMACHINE-01
**问题**：一个多存储记忆系统在 `delete_features` 后是否会产生“孤儿 citation”？
**输入**：只给仓库 commit + `semantic_memory/storage/sqlalchemy_pgvector_semantic.py` 与 `neo4j_semantic_storage.py`。
**期望正确输出**：
1. 发现 PG `citations.feature_id` 有 `ON DELETE CASCADE` → 删除 Feature 会级联删 citation；
2. 发现 Neo4j `DETACH DELETE` 删节点及属性；
3. 结论：**不应**报“PG 孤儿 citation”缺陷；若只读 `delete_features` 函数体而忽略表定义，会误报。
**考察点**：是否读到 schema/FK，而不是只看 delete 函数；是否区分“代码显式删除”与“数据库级联”。
**当前 Package 结果**：**FAIL**（C-06 误报），已在 Independent Validation 捕获。

### Case ID: KA-BENCH-MEMMACHINE-02
**问题**：`score_threshold=None` 在 event + euclidean 下会发生什么？
**输入**：`server/api_v2/service.py`、`long_term_memory.py`、`common/vector_store/data_types.py`。
**期望正确输出**：
1. API 把 `None` 转成 `-inf`；
2. `_score_passes_threshold` 在 lower-is-better 下用 `score <= threshold`；
3. `-inf` 使所有有限距离失败 → 空结果；
4. 但默认 `SimilarityMetric=COSINE`，所以不是默认必现，需显式 EUCLIDEAN + 无 reranker。
**考察点**：是否同时读默认值，避免把“条件缺陷”写成“默认必现”。

---

## 8. 验证结论

- **CONFIRMED 13 / PARTIALLY 3 / DOWNGRADED 1 / OVER_GENERALIZED 2 / CONTRADICTED 1 / MISSING 5 / HUMAN_REVIEW 2**。
- **必须修正**：C-06 降级、EK-14 条件化、EK-20 失败窗口收窄、EK-45 改 Observation、KO-05 scope 收窄、补 M-02/M-03。
- **不得**因本轮修正把任何 KO 升为 Validated Pattern；S4/S5 仍为 0。
- **结论**：Package **不是全绿**；经 Reconciliation 后可交付为“code-level archaeology, conditional pass”。

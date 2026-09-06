# 04 Flow Atlas — 七类流（从真实代码导出）

> 所有 Edge 可回溯 symbol/file/condition。每条流末尾标注关键边证据。

## F1 Control Flow（控制流）

```
index.py main
  → MemGraphRAG(global_config).initialize_graph()          # MemGraphRAG.py L46
  → pre_openie?  openie_mode=offline → assert False 中断    # L234-236（offline 只做 OpenIE）
  → index(): insert_strings → get_text_for_all_rows
    → load_existing_openie（恢复缓存，仅未处理 chunk）       # L1481+
    → batch_openie（NER 全并行 → triple 全并行）             # L214-276
    → merge_openie_results → reformat
    → extract_entity_nodes → flatten_facts
    → entity/fact embedding 入库
    → add_fact_edges → add_passage_edges → add_synonymy_edges  # 图边构建
    → augment_graph → save_igraph                            # 持久化
  → build_memory: extract_memory_schema → filter_memory_ontology
    → detect_memory_conflicts → resolve_memory_conflicts
    → build_memory_graph → install_memory_graph → save_memory_graph
  → qa（单步推理）: 检索 docs → 拼 prompt（Thought 前缀）→ 单次 LLM → split('Answer:')[1]  # IRCoT 死代码未接入（C-09）
```

**关键边证据**：
- index 11 步顺序：`index()` L214-276
- offline 中断：`assert False, logger.info(...)` L234-236
- 冲突/解析链：`detect_memory_conflicts` → `resolve_memory_conflicts` → `build_memory_graph`（L487-964）

## F2 State Flow（状态流）

```
语料 chunks
  → [chunk_id 哈希] → chunk_embedding_store（持久化）
  → OpenIE 结果（ner_results/triple_results dict，可落盘）
  → ThreeLayerMemory:
      schema_layer (SchemaNode: idx/content/frequency/embedding/fact_indices)
      fact_layer   (FactNode: idx/content/frequency/embedding/schema_idx/passage_indices)
      passage_layer(PassageNode: idx/chunk_id/content/embedding/fact_indices)
  → 冲突状态: conflict_items → edges(无向) → BFS components → actions{fact_id: kept/modified/discarded}
  → memory_graph（igraph）: type/entity/passage 节点 + node_name_to_vertex_idx
  → 检索状态: query embeddings → fact_scores(normalized) → top_k_facts → node_weights(phrase+passage) → PPR
  → QuerySolution{question, docs, doc_scores, answer}
```

**关键边证据**：ThreeLayerMemory 三映射去重 L70-90；冲突 actions 优先级 rank L757-771；PPR 状态 L2299-2339。

## F3 Data Flow（数据流）

```
passage text ──→ OpenIE.ner ──→ named_entities ──→ OpenIE.triple_extraction ──→ extracted_triples
    │                                                                              │
    │                                                            _triple_tuple_from_openie_item 归一化
    ▼                                                                              ▼
chunk_embedding_store ◄── embed(text)                              ThreeLayerMemory.build_from_*
    │                                                                              │
    │                                                            extract_memory_schema（LLM ontology converter）
    │                                                                              ▼
    │                                                              schema_layer（类型三元组 + frequency）
    │                                                                              │
    │                                                            filter_memory_ontology（absolute/percentile）
    ▼                                                                              ▼
entity/fact embedding store ◄── embed(entity/fact)                build_memory_graph（图节点/边）
    │                                                                              │
    └──────────────► retrieve: get_fact_scores → rerank_facts → graph_search → run_ppr → docs
                     （add_fact_edges: node_to_node_stats 共现计数 → 图边权重，供 PPR weights='weight'）
```

**关键边证据**：`get_fact_scores` L1944-2030；`dense_passage_retrieval` L2032-2088；`graph_search_with_fact_entities` L2089。

## F4 Evidence Flow（证据流）

```
passage（原文证据）
  → conflict 候选: evidence[fact.idx] = passages[:conflict_passage_evidence_per_fact]（每 fact ≤N 条，截 1200 字符）
  → detect_one: "Target evidence {i}: {text}" / "Evidence {i}: {text}" 注入 prompt
  → LLM 判定（fact checker）→ conflict_items 带 evidence_passages{target, other}
  → resolve_component: rows 带 evidence_1/evidence_2（双份证据注入 curator）
  → content-filter fallback: 去掉 evidence 的最小化 payload（保守降级）
```

**关键边证据**：`_conflict_maps` L487-495；`detect_one` L539-554；`resolve_component` rows L700-707；fallback L728-731。

## F5 Authority Flow（权威流）

```
LLM 角色（认知权威，temperature=0 约束）：
  entity classifier → 实体类型（NER）
  triple extraction → 关系三元组（OpenIE）
  ontology converter → 本体归纳（schema 抽取）
  fact checker → 冲突判定（是否 hard conflict）
  curator → 冲突解析（kept/modified/discarded 决策）
  QA / IRCoT → 答案生成（多跳推理）
代码（结构权威，确定性）：
  _conflict_candidates（候选生成）
  rank 优先级（kept<modified<discarded，冲突时低 rank 胜出）   # L757-758
  by_normalized 匹配（LLM 引用无效时用归一化回退）             # L769-775
  断言防御（空权重阻止 PPR）
```

**关键边证据**：LLM 判定权威 EK-06；代码裁决权威 EK-05/07；"LLM 提议、代码裁决"模式。

## F6 Memory Flow（记忆流）

```
短期（进程内）：ThreeLayerMemory 对象 + 冲突 actions dict + rerank_log
长期（持久化）：
  openie_results.json（OpenIE 缓存，可 load/save）
  memory.json（三层记忆序列化 from_dict/to_dict）
  memory_graph / graph.graphml（igraph 持久化）
  chunk/entity/fact embedding store（embedding_store.py）
  QA results JSON（含配置 + 逐 query 详情）
增量恢复：load_existing_openie 只处理未缓存 chunk（记忆增量扩展）
```

**关键边证据**：`save_memory_graph`/`load_memory_graph` L964-1038；`load_existing_openie` L1481；`to_dict/from_dict` Memory.py L304-390（实测往返一致）。

## F7 Policy Flow（策略流）

```
决策（配置）：BaseConfig 全量参数化（global_config）
  → 策略固化：openie_mode / force_from_scratch / conflict_* / ontology_filter_* / retrieval_* / damping
  → 强制执行点：
      * offline 模式 assert False（OpenIE 必须先完成）          # L234-236
      * 冲突判定门槛 conflict_min_confidence（低于跳过）        # L575-579
      * 分量大小上限 conflict_resolution_max_component_size     # L689-693
      * 过滤模式 ontology_filter_mode（absolute/percentile）    # L452+
      * 频率归一化 ent_node_to_num_chunk                        # L2110-2116
  → 未来决策反馈：audit 数据（conflict audit/component_resolutions/applied_actions）+ QA results JSON（实验记录）
```

**关键边证据**：config_utils BaseConfig L16；audit 返回 L783-796；QA results 落盘（README 输出结构）。

---

## 流完整性检查（Flow→KO 交叉校验）

| 流 | 对应 KO | 关键边可回溯 |
|----|---------|-------------|
| F1 Control | KO-02/KO-03 | ✅ index 11 步 + 冲突链 |
| F2 State | KO-01 | ✅ 三层记忆 + 实测 |
| F3 Data | KO-01/KO-03 | ✅ OpenIE→记忆→图→检索 |
| F4 Evidence | KO-02/KO-06 | ✅ 证据注入 + fallback |
| F5 Authority | KO-02/CM-2 | ✅ LLM 提议/代码裁决 |
| F6 Memory | KO-04 | ✅ 持久化 + 增量恢复 |
| F7 Policy | KO-05/KO-06 | ✅ 配置即策略 + 执行点 |

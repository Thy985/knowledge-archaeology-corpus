# 02 Engineering Knowledge — EK Graph（宽底座）

> 30 条 EK，每条带 `links`（6 类边）。证据来源均可回溯 cd6fabd。Fact=实现已读/实测；Obs=文档声称。

## EK-01 三层记忆架构（Schema 1:N Fact N:M Passage）
- **内容**：ThreeLayerMemory 组织知识为三层：Schema 层（本体类型三元组）、Fact 层（具体关系三元组）、Passage 层（原文 chunk）；层间双向链接（Schema→Fact→Passage）。
- **证据**：Memory.py docstring（L1-15）+ 实测（schema/fact/passage 各 1，双向链接正确）
- **epistemic**: Fact（实现 + 实测）
- **links**: [causal: →EK-02（记忆→图）], [subsystem: EK-01/EK-02 同记忆构建面]

## EK-02 去重映射与频率语义
- **内容**：三个哈希映射（_schema_to_idx/_fact_to_idx/_chunk_id_to_idx）保证唯一性；fact.frequency = 出现该 triple 的 distinct passage 数；schema.frequency = 关联 facts 数。
- **证据**：Memory.py L70-90 + 实测（同 triple 重建不增；freq=distinct chunks）
- **epistemic**: Fact（实测）
- **links**: [mechanism: EK-01], [constraint: 约束 EK-03 的 schema 过滤]

## EK-03 本体归纳（ontology induction）
- **内容**：extract_memory_schema 用 LLM（ontology converter 角色）把每条 fact + 前 3 个 passage 上下文转成 (head_type, relation, tail_type)；类型值用 < > 包裹归一化；去重到 schema 表并统计频率；可抽样（schema_extraction_sample_facts）。
- **证据**：MemGraphRAG.py L358-450；prompts/prompt.py L299（"You are an ontology converter"）
- **epistemic**: Fact（实现精读）
- **links**: [causal: EK-03→EK-04（归纳→过滤）]

## EK-04 本体频率过滤（两种模式）
- **内容**：filter_memory_ontology 两种模式：absolute（frequency < 阈值删除）或 percentile（低百分比删除）；删除后重建层间索引。
- **证据**：MemGraphRAG.py L452-486
- **epistemic**: Fact
- **links**: [constraint: EK-04 约束 EK-07（低频 schema 不进入图）]

## EK-05 冲突候选确定性生成（结构规则）
- **内容**：_conflict_candidates 用纯结构规则生成候选对：①同 subject+relation 不同 object（矛盾候选）②完全重复（duplicate 单独处理，排除出矛盾）③可选 reverse-relation 检查（head↔tail 反转 + 同 relation）；每 target 限制最大相关数。
- **证据**：MemGraphRAG.py L496-527
- **epistemic**: Fact
- **links**: [causal: EK-05→EK-06（候选→判定）], [contrast: EK-06（确定性 vs LLM）]

## EK-06 冲突判定：LLM 角色（fact checker）+ 证据交叉匹配
- **内容**：detect_memory_conflicts 对每个候选对调 LLM（temperature=0.0，fact checker 角色），输入 target triple + related triples + passage evidence；解析 conflicting_triple_ids 与 conflicts 列表；过滤规则：非 hard、置信度 < 阈值、类型属 duplicate/none/uncertain 均跳过；**证据交叉匹配**：LLM 返回的 triple 通过归一化与候选匹配回 fact_id；流式模式只查前序 fact。
- **证据**：MemGraphRAG.py L529-632；prompts/prompt.py L352（"expert knowledge graph fact checker"）
- **epistemic**: Fact（实现精读）
- **links**: [causal: EK-06→EK-07], [contrast: EK-05]

## EK-07 冲突解析：连通分量批量 + 优先级动作 + content-filter fallback
- **内容**：resolve_memory_conflicts 把冲突对构无向图 → BFS 连通分量；每个分量单次 LLM 调用（curator 角色，temperature=0）解析全部冲突对（含双份证据）；分量过大（> max_component_size）跳过为 unresolved；**content-filter fallback**：LLM 因 content filter 失败时用去掉 evidence 的最小化 payload 保守重试；动作优先级 kept(1) < modified(2) < discarded(3)；LLM 返回 triple_id 无效时用归一化匹配回退；完整 audit 数据。
- **证据**：MemGraphRAG.py L676-800
- **epistemic**: Fact
- **links**: [causal: EK-07→EK-08（解析→图）], [mechanism: EK-11（都是 fallback 模式）]

## EK-08 记忆派生图（memory-derived graph）
- **内容**：build_memory_graph + install/save_memory_graph：冲突解析后的记忆生成 type/entity/passage 三类节点的图；图节点命名含类型前缀（entity- / passage- 哈希 ID）。
- **证据**：MemGraphRAG.py L801-964
- **epistemic**: Fact
- **links**: [causal: EK-08→EK-09（图→检索）]

## EK-09 检索融合：phrase 权重 + 频率归一化 + PPR
- **内容**：graph_search_with_fact_entities：top facts 的 subject/object phrase → 图节点赋 fact_score（除以实体出现 chunk 数归一化）→ linking_score_map（phrase→平均分）→ link_top_k 截断 → dense passage 分数 min-max 归一化 × passage_node_weight=0.05 → node_weights 相加 → run_ppr（igraph personalized_pagerank，damping=0.5，directed=False，weights='weight'，prpack）。
- **证据**：MemGraphRAG.py L2089-2185、L2299-2339
- **epistemic**: Fact
- **links**: [causal: EK-09→EK-10], [mechanism: EK-16（embedding 融合）]

## EK-10 检索 fallback：无 fact → dense passage retrieval
- **内容**：retrieve 中 rerank 后无 top_k_facts 时，默认回退到 dense_passage_retrieval（"Long queries with no relevant facts ... will default to results from dense passage retrieval"）。
- **证据**：MemGraphRAG.py L1078-1090（retrieve docstring + 分支）
- **epistemic**: Fact
- **links**: [contrast: EK-09（图搜索 vs 纯 dense fallback）]

## EK-11 LLM 重排（rerank_facts）+ DSPy 优化模板
- **内容**：rerank_facts 两模式：①skip_fact_rerank + use_raw_threshold_filter（阈值过滤，跳过 top-k）②top-k + DSPyFilter.llm_call（DSPy 优化模板 filter_llama3.3-70B-Instruct.json）；rerank_log 记录 before/after 与统计。
- **证据**：MemGraphRAG.py L2186-2299；rerank.py；prompts/dspy_prompts/
- **epistemic**: Fact
- **links**: [mechanism: EK-09], [contrast: EK-10]

## EK-12 eval() 解析 fact 内容（安全边界）
- **内容**：从 fact_embedding_store 加载 fact 内容时用 eval(fact_content) 解析（内容为 OpenIE 生成的三元组字符串）；eval 未限制输入来源——若 OpenIE 结果被污染可执行任意代码。
- **证据**：MemGraphRAG.py rerank_facts 中 eval(fact_row_dict[id]['content'])（两处）
- **epistemic**: Fact（代码 + 安全边界分析）
- **links**: [constraint: EK-12 是 EK-09/11 的输入面风险], [causal: →Candidate C-04]

## EK-13 OpenIE 两阶段多线程管道（batch_openie）
- **内容**：batch_openie 两阶段 ThreadPoolExecutor：先全部 chunk 并行 NER，再全部并行 triple extraction（依赖 NER 结果）；token 使用与缓存命中统计。
- **证据**：information_extraction/openie_openai.py L129-204
- **epistemic**: Fact
- **links**: [causal: EK-13→EK-01（OpenIE→记忆）], [mechanism: EK-17（并发模式）]

## EK-14 OpenIE 结果格式归一化（_triple_tuple_from_openie_item）
- **内容**：兼容多种 OpenIE 输出（processed_triple/raw_triple/triple_str/裸 3 元组），统一为 (h,r,t) 字符串元组；含 ast.literal_eval 解析 triple_str。
- **证据**：Memory.py L117-148
- **epistemic**: Fact
- **links**: [constraint: EK-14 保证 EK-01 输入兼容性]

## EK-15 缓存策略：chunk 哈希 ID + load/save OpenIE + force-from-scratch
- **内容**：chunk 内容哈希为 ID（compute_mdhash_id）；load_existing_openie 按未处理 chunk 增量执行；save_openie_results 持久化；force_openie_from_scratch / force_index_from_scratch 强制重建。
- **证据**：MemGraphRAG.py L1481-1603；config_utils.py（force 标志）
- **epistemic**: Fact
- **links**: [causal: EK-15→EK-13（缓存→增量执行）], [mechanism: EK-18（幂等）]

## EK-16 三类 embedding 存储分离
- **内容**：chunk_embedding_store / entity_embedding_store / fact_embedding_store 分离持久化；检索时分别查询（query embeddings → fact scores → dense passage）。
- **证据**：MemGraphRAG.py（prepare_retrieval_objects/get_fact_scores/dense_passage_retrieval）；embedding_store.py
- **epistemic**: Fact
- **links**: [causal: EK-16→EK-09]

## EK-17 多线程并发（memory_max_workers / retrieval_max_workers）
- **内容**：schema 抽取/冲突检测/冲突解析/检索均支持 ThreadPoolExecutor 并发（workers 配置），单 worker 时串行 tqdm。
- **证据**：MemGraphRAG.py 多处（workers==1 分支 + ThreadPoolExecutor）
- **epistemic**: Fact
- **links**: [mechanism: EK-13], [constraint: 并发受 LLM 限流约束（未显式处理）→C-05]

## EK-18 幂等与可重建（恢复点）
- **内容**：所有中间产物可落盘（openie_results/memory.json/memory_graph/graph.graphml/embeddings），配合 force 标志可增量或全量重建；SOURCE_DATE 无（无时间戳注入，注意与 skillfortify 差异）。
- **证据**：README 输出结构 + config force 标志 + save/load 方法
- **epistemic**: Fact
- **links**: [mechanism: EK-15]

## EK-19 QA 管线：单步推理 + Answer 解析协议（IRCoT 为死代码，Reconciliation 修正）
- **内容**：qa() 是**单步推理**：检索 docs（前 qa_top_k）拼 prompt（"Thought: " 前缀）→ 单次 LLM 调用 → `response_content.split('Answer:')[1]` 提取答案（解析失败则返回原文）；数据集特定模板 rag_qa_{dataset} 缺失时 fallback musique。**IRCoT 迭代（reason_step，qa_utils.py L34-56）与 max_qa_steps 配置均未接入主流程**（全仓无调用者；MemGraphRAG.py 对 max_qa_steps 引用 0 次）——为预留扩展/死代码。
- **证据**：MemGraphRAG.py L1240-1330（qa）；qa_utils.py L34-56（reason_step 无调用者）；config_utils.py L241（max_qa_steps）；retrieval_dataset_test.py L286（传入但未消费）
- **epistemic**: Fact（单步推理）/ Observation（死代码状态）
- **links**: [causal: EK-19→EK-20（QA→评估）], [mechanism: EK-24（模板管理）], [contrast: EK-32（QA 协议）], [causal: →C-09]

## EK-20 评估即运行时（gold 传入 + EM/F1/Recall@k）
- **内容**：rag_qa/retrieve 接受 gold_answers/gold_docs 即评估：QAExactMatch/QAF1Score（aggregation_fn=np.max）、RetrievalRecall（k 列表）；结果写入 QA results JSON。
- **证据**：evaluation/qa_eval.py / retrieval_eval.py；MemGraphRAG.py rag_qa
- **epistemic**: Fact
- **links**: [causal: EK-20→Candidate C-06（无自动化测试）]

## EK-21 断言即控制流（assert 用法）
- **内容**：pre_openie 末尾 `assert False, logger.info('Done with OpenIE, run online indexing for future retrieval.')`——离线模式用 assert False 有意中断流程（先做 OpenIE，之后才跑 online indexing）；graph_search assert sum(node_weights) > 0 防空图搜索。
- **证据**：MemGraphRAG.py L234-236、L2150
- **epistemic**: Fact
- **links**: [contrast: EK-22（防御 vs 流程）], [causal: →Candidate C-03]

## EK-22 防御性断言与异常处理
- **内容**：graph_search 空权重断言；resolve 中 LLM 调用 try/except（content_filter 特定降级，其他异常重抛）；Memory 构建容错（无效 triple 跳过）。
- **证据**：MemGraphRAG.py L734-748、L2150；Memory.py 多处
- **epistemic**: Fact
- **links**: [mechanism: EK-21]

## EK-23 "Multi-Agent"命名 vs 多角色 prompt 管道（设计-实现差距）
- **内容**：README 标题 "Multi-Agent System"、候选卡称"多 Agent 组协同构建"；实现为 7 个 LLM 角色 prompt（entity classifier / triple extraction / ontology converter / fact checker / curator / QA / IRCoT）+ 多线程并发——**无 agentic 循环、无工具调用、无状态 agent**。
- **证据**：README 标题 vs prompts/prompt.py 4 角色 + templates 3 角色 + ThreadPoolExecutor
- **epistemic**: Fact（命名 vs 实现并列）
- **links**: [contrast: EK-19（IRCoT 是唯一接近 agentic 的部分）], [causal: →Candidate C-01]

## EK-24 prompt_template_manager（模板注册/渲染/回退）
- **内容**：模板管理器：is_template_name_valid 检查数据集模板存在，缺失 fallback 到 musique；render 渲染 user prompt。
- **证据**：prompts/prompt_template_manager.py L200 行；MemGraphRAG.py L1245-1258
- **epistemic**: Fact
- **links**: [mechanism: EK-19]

## EK-25 LLM 缓存（CacheOpenAI）
- **内容**：openai_gpt.py 的 CacheOpenAI 对相同输入缓存响应（cache_hit 统计在 batch_openie 中）；token 使用全程统计（prompt/completion）。
- **证据**：llm/openai_gpt.py；openie_openai.py（num_cache_hit）
- **epistemic**: Fact
- **links**: [causal: EK-25→EK-13（缓存命中降成本）], [mechanism: EK-15]

## EK-26 response_format=json_object + 解析容错
- **内容**：默认 response_format={"type":"json_object"} 强制 LLM JSON；_memory_json/_memory_infer_result 解析 LLM 输出，异常/空结果返回默认结构（不崩溃）。
- **证据**：config_utils.py（response_format）；MemGraphRAG.py L313-353
- **epistemic**: Fact
- **links**: [constraint: EK-26 约束 EK-03/06/07 的 LLM 输出解析]

## EK-27 图节点命名与查询（entity-/passage- 前缀哈希）
- **内容**：图节点 key 用 compute_mdhash_id(prefix="entity-"/"passage-")；node_name_to_vertex_idx 映射；get_triples_by_entity / build_entity_to_triples_index 支持实体查询。
- **证据**：MemGraphRAG.py L1746-1875；misc_utils（compute_mdhash_id）
- **epistemic**: Fact
- **links**: [mechanism: EK-09（phrase→节点）], [subsystem: EK-08]

## EK-28 频率归一化（ent_node_to_num_chunk）
- **内容**：实体节点权重按出现 chunk 数归一化（phrase_weights /= ent_node_to_num_chunk）——高频实体不被过度放大。
- **证据**：MemGraphRAG.py L2110-2116
- **epistemic**: Fact
- **links**: [constraint: EK-28 约束 EK-09 的权重公平性]

## EK-29 数据与实验组织（dataset/ + outputs/ + results/）
- **内容**：dataset/ 提供 4 个基准（hotpotqa/2wikimultihopqa/musique/medical）；outputs/ 存中间产物；results/ 存 QA 结果 JSON（含 summary/config/solutions/raw/metadata）。
- **证据**：README 输出结构 + dataset/ 目录
- **epistemic**: Fact
- **links**: [constraint: EK-29 支撑 EK-20 评估]

## EK-30 无测试目录（研究仓库特性）
- **内容**：仓库无任何测试文件；质量保障依赖：①KDD'26 论文中的基准结果 ②运行时评估（gold 传入）③缓存/幂等设计。
- **证据**：find（无 tests/）+ requirements 无 pytest
- **epistemic**: Fact
- **links**: [causal: →Candidate C-06]

---

## EK-31 图边权重：实体共现统计（Reconciliation 补充）
- **内容**：add_fact_edges 中 node_to_node_stats 记录实体对（subject↔object）共现次数（双向 +1）作为图边权重统计来源；ent_node_to_num_chunk 累计每个实体出现的 chunk 数（检索时做频率归一化，见 EK-28）。图边携带共现信号，非无权图。
- **证据**：MemGraphRAG.py add_fact_edges（L1330-1420）；add_passage_edges
- **epistemic**: Fact
- **links**: [causal: EK-31→EK-09（共现→PPR 权重面）], [mechanism: EK-28]

## EK-32 QA 单步推理 + Answer 解析协议（Reconciliation 补充）
- **内容**：qa() 用 `split('Answer:')[1]` 解析 LLM 输出——**prompt 格式约定（Thought/Answer）与解析代码强耦合**；解析失败 fallback 返回原文（不崩溃）；QA prompt 保存前 5 条到 sample_qa_prompts.json（可审计）。
- **证据**：MemGraphRAG.py L1300-1325
- **epistemic**: Fact
- **links**: [constraint: EK-32 约束 EK-19（解析依赖格式）], [mechanism: EK-26]

## EK-33 检索/QA CLI 默认走阈值过滤（Reconciliation 补充）
- **内容**：run_retrieval_test.sh 默认 SKIP_LLM_RERANK=true + FACT_SIMILARITY_THRESHOLD=0.6 + USE_RAW_THRESHOLD_FILTER=true——主路径为**相似度阈值过滤（免 LLM rerank 成本）**；LLM rerank（DSPyFilter）为可选路径（--skip-fact-rerank false）。示例脚本硬编码作者第三方端点 https://apis.aaife.cn/v1 与本地 bge-large-en-v1.5 路径。
- **证据**：run_retrieval_test.sh（全文）；retrieval_dataset_test.py L247-249 默认值
- **epistemic**: Fact
- **links**: [contrast: EK-11（阈值 vs LLM rerank）], [constraint: EK-33 表明 EK-11 双模式中阈值为默认]

## EK 边统计（防退化检查）

- 总 EK：33；含 links：33/33（100%）；游离：0；平均出边 ~1.9

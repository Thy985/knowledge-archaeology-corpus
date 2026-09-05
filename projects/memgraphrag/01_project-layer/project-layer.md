# 01 Project Layer — MemGraphRAG 项目地图

> 全部事实来自 cd6fabd 快照实际内容（`repo/`）。

## 1. 项目身份

| 项 | 值 | 证据 |
|----|-----|------|
| name | MemGraphRAG | README.md / pyproject 无（无 pyproject，纯脚本仓库） |
| 描述 | "Memory-based Multi-Agent System for Graph Retrieval-Augmented Generation" | README.md 标题 |
| license | MIT（Copyright 2026 DeepLIT Group in Xiamen University） | LICENSE |
| 论文 | arXiv 2606.00610（KDD'26 接收，2026-05-17 news） | README News |
| 依赖 | openai / litellm==1.73.1 / vllm / gritlm / torch==2.5.1 / transformers / networkx / python_igraph / tenacity / tiktoken / nest_asyncio / numpy / scipy / tqdm / einops / boto3 | requirements.txt |
| Python | 3.10+ 推荐 | README Quickstart |
| 入口 | `code/index.py`（索引 CLI）+ `code/retrieval_dataset_test.py`（检索+QA CLI） | README / code/ |
| 上游借鉴 | HippoRAG（OSU-NLP-Group）——Acknowledgements 声明 | README |
| 测试 | **无测试目录** | find（tests/ 不存在） |

## 2. 顶层结构

```
repo/
├── README.md / LICENSE / requirements.txt / framework.png
├── code/
│   ├── index.py / retrieval_dataset_test.py / run_index.sh / run_retrieval_test.sh
│   └── src/
│       ├── MemGraphRAG.py      # 管线编排（2,339 行，核心）
│       ├── Memory.py           # 三层记忆数据结构（503 行）
│       ├── embedding_store.py  # 持久化 embedding 存储（156 行）
│       ├── rerank.py           # DSPy fact 重排（LLM）
│       ├── embedding_model/    # BGE / Contriever / GritLM / NVEmbedV2 后端
│       ├── information_extraction/  # openie_openai / openie_vllm_offline
│       ├── llm/                # openai_gpt（带缓存）/ vllm_offline
│       ├── prompts/            # prompt.py（4 角色）+ prompt_template_manager + templates/（ner/triple_extraction/rag_qa/ircot）+ dspy_prompts/
│       ├── evaluation/         # qa_eval（EM/F1）/ retrieval_eval（Recall@k）
│       └── utils/              # config_utils / llm_utils / embed_utils / misc_utils / qa_utils / eval_utils / logging_utils / typing
└── dataset/                    # hotpotqa / 2wikimultihopqa / musique / medical 示例
```

## 3. 核心模块

| 模块 | 职责 | 关键符号 |
|------|------|---------|
| MemGraphRAG.py | 管线编排：索引 11 步 + 检索 9 步 + 图操作 7 项 | class MemGraphRAG（L46） |
| Memory.py | 三层记忆（Schema/Fact/Passage）+ 双向链接 + 去重 + 序列化 | ThreeLayerMemory / SchemaNode / FactNode / PassageNode |
| rerank.py | DSPy 优化模板的 LLM fact 重排 | class DSPyFilter |
| embedding_store.py | 持久化 embedding 存储（chunk/entity/fact 三类） | class（store per type） |
| openie_openai.py | OpenIE：NER + triple extraction 两阶段多线程 | class OpenIE（batch_openie） |
| llm/openai_gpt.py | OpenAI 兼容调用 + 缓存 | class CacheOpenAI |
| prompts/prompt.py | 4 个 LLM 角色 prompt（entity classifier / ontology converter / fact checker / curator） | MEMORY_PROMPTS |
| prompts/templates/ | NER / triple_extraction / rag_qa_{dataset} / ircot_{dataset} | 数据集特定模板 |
| evaluation/qa_eval.py | QA 评估：ExactMatch / F1 | QAExactMatch / QAF1Score |
| utils/config_utils.py | 单一配置类 BaseConfig（dataclass + HfArgumentParser） | class BaseConfig |

## 4. 核心数据结构

| 结构 | 定义 | 语义 |
|------|------|------|
| `SchemaNode` | Memory.py L37 | 本体层节点：(head_type, relation, tail_type) + frequency + fact_indices |
| `FactNode` | Memory.py L47 | 事实层节点：(head, relation, tail) + frequency（出现 chunk 数）+ schema_idx + passage_indices |
| `PassageNode` | Memory.py L58 | 原文层节点：chunk_id + content + fact_indices |
| `ThreeLayerMemory` | Memory.py L67 | 三层列表 + 3 个去重映射（_schema_to_idx/_fact_to_idx/_chunk_id_to_idx） |
| `QuerySolution` | typing.py | 检索结果：question/docs/doc_scores/answer/metadata |
| `BaseConfig` | config_utils.py L16 | 全量配置（LLM/存储/预处理/OpenIE/embedding/检索/QA/冲突/实验） |

## 5. 核心状态

- **三层记忆状态**：schema_layer/fact_layer/passage_layer 列表 + 去重映射；层间双向索引（schema→facts、fact→passages、passage→facts）
- **图状态**：igraph Graph（type/entity/passage 三类节点）+ node_name_to_vertex_idx + ent_node_to_num_chunk（实体出现 chunk 数）
- **缓存状态**：chunk_embedding_store / entity_embedding_store / fact_embedding_store（持久化）+ openie_results 缓存（load/save）
- **冲突状态**：conflict_items → 连通分量 → actions（kept/modified/discarded 优先级）→ audit 数据
- **配置状态**：BaseConfig（global_config，贯穿所有阶段）

## 6. 测试体系

- **无测试目录、无 pytest 配置**（find 确认）——研究仓库，以"实验脚本 + 结果 JSON"代替自动化测试
- 评估体系：evaluation/qa_eval.py（EM/F1）+ retrieval_eval.py（Recall@k，k=[1,2,5,10,20,30,50,100,150,200]）——评估在运行时执行（gold answers/docs 传入）
- 实验追踪：batch QA 记录 answers/retrieved docs/scores/latency/token usage

## 7. 主要配置（BaseConfig 关键项）

| 配置 | 默认 | 用途 |
|------|------|------|
| llm_name / llm_base_url | gpt-4o-mini / None | LLM 后端 |
| temperature | 0 | 采样温度（冲突判定/解析用 0.0） |
| response_format | {"type": "json_object"} | 强制 LLM JSON 输出 |
| max_retry_attempts | 5 | LLM 重试 |
| openie_mode | online / offline | 在线 vs 离线（vllm）OpenIE |
| passage_node_weight | 0.05 | PPR 中 passage 权重缩放 |
| conflict_passage_evidence_per_fact | — | 冲突证据 passage 数上限 |
| conflict_min_confidence | — | 冲突判定置信度门槛 |
| conflict_resolution_max_component_size | — | 分量解析上限（过大跳过） |
| conflict_enable_reverse_relation_check | — | 反向关系冲突检查开关 |
| conflict_streaming_only_previous | — | 流式只查前序 fact |
| schema_extraction_sample_facts | 0 | schema 抽取抽样（0=全部） |
| ontology_filter_mode | absolute/percentile | 本体过滤模式 |
| linking_top_k / retrieval_top_k / qa_top_k | — | 检索各阶段 top-k |
| damping | 0.5 | PPR 阻尼 |
| retrieval_max_workers / memory_max_workers | — | 并发度 |

## 8. 权限 / Policy / Governance 机制

| 机制 | 证据 |
|------|------|
| 无显式权限模型 | 纯研究工具，无沙箱/权限控制 |
| 配置即治理 | 所有关键行为由 BaseConfig 参数化（可重现实验） |
| 缓存即幂等 | chunk 哈希 ID + load/save openie + force-from-scratch 标志（可增量/可重建） |
| 保守解析 fallback | 冲突解析遇 content-filter 时用最小化 payload 重试（L734-748） |
| 断言防御 | graph_search assert sum(node_weights) > 0；pre_openie assert False 作流程中断 |

## 9. 主要外部依赖

- 运行时：openai / litellm / vllm（可选离线）/ gritlm（可选 embedding）/ torch / transformers / networkx / python_igraph / tenacity / tiktoken / nest_asyncio / numpy / scipy / boto3
- 模型：bge-large-en-v1.5（默认 embedding）/ NV-Embed-v2 / gpt-4o-mini 等 OpenAI 兼容端点

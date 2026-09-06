# Independent Validation Report — ARCH-2026-09-06-001 (memgraphrag)

> Auditor 盲重建：不把考古结果当事实来源，独立重读 repo（重点重读 qa()/qa_utils.py/retrieval_dataset_test.py/run_retrieval_test.sh/embedding_store.py/add_fact_edges/index.py），建立 Independent Findings 后与考古包比对。

## 判定统计

| 判定 | 数量 | 对象 |
|------|------|------|
| CONFIRMED | 24 | EK-01~18（除 15 外）、EK-20~30（除 19 外）、KO-01/02/04/05/06/08、CM-1/2/4、M-1/2/3/4 |
| PARTIALLY_CONFIRMED | 2 | EK-15（缓存机制对，增量路径细节待核）、KO-03（IRCoT 表述需收紧） |
| DOWNGRADED | 1 | EK-19（"IRCoT 迭代推理用于 QA"→ 实为单步推理 + IRCoT 死代码） |
| OVER_GENERALIZED | 1 | KO-03/CM-3 中"IRCoT agentic"表述 |
| MISSING | 4 | 图边权重共现机制 / QA 单步+Answer 解析协议 / CLI 阈值过滤主路径 / IRCoT 死代码+max_qa_steps 未消费 |
| CONTRADICTED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

## 原考古最重要的 3 个成功

1. **三层记忆核心数据结构 + 双向链接/去重/频率/序列化往返经定向实测验证**（S4）——考古包中最强证据点，独立重建亦通过，且与 Memory.py docstring 一致。
2. **冲突处理三阶段分离（确定性候选 → LLM 判定 → 连通分量解析）**机制识别准确，含证据交叉匹配、置信度门槛、分量上限、content-filter fallback 等细节——独立重建确认全部属实。
3. **eval() 安全边界发现（C-04）**——rerank_facts 中 eval(fact_content) 两处确凿，对"LLM 输出 → eval"反模式有普遍警示价值，未因是研究工具而淡化。

## 原考古最重要的 3 个错误

1. **EK-19 错误（DOWNGRADED）**：声称"QA 管线：IRCoT 迭代推理（一步一个 thought）"并把它当主流程。独立核查：`qa()`（MemGraphRAG.py L1240-1330）是**单步推理**——检索 docs 拼 prompt → 单次 LLM 调用 → `response_content.split('Answer:')[1]` 提取答案；`ircot_{dataset}` 模板只在 `qa_utils.reason_step`（L34-56）被引用，而 reason_step **无任何调用者**（grep 全仓确认）；`max_qa_steps` 在 MemGraphRAG.py **0 次引用**。→ IRCoT 迭代路径是**死代码/预留扩展**，不是 QA 主流程。
2. **遗漏图边权重机制（MISSING）**：`add_fact_edges`（L1330-1420）中 `node_to_node_stats` 记录实体对共现次数作为边权重统计、`ent_node_to_num_chunk` 累计实体出现 chunk 数——图不是无权图，边有权重（共现），原考古的 KO-03/EK-08 未覆盖。
3. **遗漏 CLI 主路径（MISSING）**：`run_retrieval_test.sh` 默认 `SKIP_LLM_RERANK=true` + `FACT_SIMILARITY_THRESHOLD=0.6` + `USE_RAW_THRESHOLD_FILTER=true`——**实际主路径是阈值过滤而非 LLM rerank**（LLM rerank 是可选路径）；原考古把 LLM rerank 描述为默认主路径之一（EK-11 表述偏重）。

## 关键遗漏清单（已 Reconciliation 补入）

- **EK-31 图边权重：实体共现统计**（add_fact_edges 的 node_to_node_stats / ent_node_to_num_chunk）
- **EK-32 QA 单步推理 + Answer 解析协议**（qa() 单次调用 + split('Answer:')，prompt 格式强耦合）
- **EK-33 检索/QA CLI 默认走阈值过滤**（skip_fact_rerank 默认 True；LLM rerank 可选）
- **C-09 IRCoT 死代码 + max_qa_steps 未消费**（预留扩展点，未接入主流程）

## 是否存在错误升维？

- 是（轻度）：KO-03/CM-3 中"IRCoT agentic 迭代"表述 → 已收紧为"单步推理 + 预留 IRCoT 死代码（未接入）"。
- KO-07（名实之辨）经审计反而**强化**：连 IRCoT 都未接入主流程，"Multi-Agent"更多是命名/论文叙事，agentic 元素更少。

## 是否存在事实错误？

- EK-19 部分（见错误 1）。其余 24 条 EK 独立重读未发现事实错误。
- 06_validation 中"QA 依赖 LLM 未实跑"标注诚实。

## 是否存在 Flow 错误？

- F1 控制流：`qa` 段"IRCoT 迭代推理"有误 → 已修正为"单步推理"。
- F3 数据流：补充 fact 边权重（node_to_node_stats）数据流。
- 其余六类流 Edge 逐一回溯通过。

## 新 Benchmark / Regression Case？

- **C-08（原）**：ThreeLayerMemory 纯 Python 实测 → 可固化为 skill 的 Regression Case（数据结构声明 vs 实现一致性）。
- **新 R-01**：`max_qa_steps` dead-config 检查——"配置项是否被消费"可做成通用 Regression Case（grep config 字段在核心类的引用次数），对考古"设计 vs 实现差距"检查有用。
- **新 R-02**：eval/动态执行点扫描——考古任何 LLM 管道时扫描 `eval(`/`exec(` 与输入来源（C-04 模式）。

## 结论

**PASS（含 Reconciliation）**：核心认知（三层记忆/冲突三阶段/检索融合/名实之辨）全部成立；3 处错误与 4 处遗漏已在 Reconciliation 补正，不改变认知核心，但显著提升 QA/IRCoT 相关表述的准确性。

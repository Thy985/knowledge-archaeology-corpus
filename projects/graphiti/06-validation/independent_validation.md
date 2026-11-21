# Independent Validation Report

> **方法**：Auditor 未把 Package 结论作为输入，直接从 `repo/` 重新读取源码与文档，先形成 Independent Findings，再与 Package 比对。
> **约束**：仅只读仓库；未修改 `package/` 任何文件。
> **环境**：无 Neo4j / FalkorDB / Docker / LLM key，**无任何运行验证**。

## 一、Independent Findings（重新建立，摘要）

| # | 独立读到的事实 | 来源 |
|---|---|---|
| IF-01 | `add_episode` 顺序为：`retrieve_episodes` → `extract_nodes` → `resolve_extracted_nodes` → `_extract_and_resolve_edges` → `extract_attributes_from_nodes` → `_process_episode_data` → 可选 `update_communities` | `graphiti.py:1450-1545` |
| IF-02 | `_resolve_request_scope` 对非默认 group 返回 `driver.clone(database=group_id)` + `clients.model_copy(update={'driver': driver})`；默认库路径原样返回 | `graphiti.py:1304-1330` |
| IF-03 | 去重常量 `15` / `0.6`；顺序 identity → similarity → `_resolve_with_llm` | `node_operations.py:67-68, 816-890` |
| IF-04 | 候选三路：identity / exact name / semantic | `node_operations.py:456-469` |
| IF-05 | 边签名：strict 丢弃；非 strict 改名 `DEFAULT_EDGE_NAME` + `attributes={}` | `edge_operations.py:671-714` |
| IF-06 | 失效：候选起始更晚 → 本边失效；矛盾消解仅在"候选在区间外/更早"时失效候选，否则 `continue` | `edge_operations.py:715-786` |
| IF-07 | `SearchConfig` 常量共 **16** 个：`COMBINED_*` 3 + `EDGE_*` 5 + `NODE_*` 5 + `COMMUNITY_*` 3 | `search_config_recipes.py:34-217` |
| IF-08 | `@mcp.tool()` 共 **13** 个（含 `get_status`） | `mcp_server.py:403-1146` |
| IF-09 | REST 端点 **10** 个；`/messages` 返回 202 并入队 | `routers/ingest.py:51-105`、`retrieve.py:17-44` |
| IF-10 | 实测规模（排除 `.git`）：**386 文件 / 300 .py / 76,764 行 / test_*.py 81 / workflows 15 / spec 6 md + 1 yaml** | `os.walk` 复算 |
| IF-11 | `SECURITY.md` 原文含 "These limits are intentional. Do not expose these servers to an untrusted network." 与 3 类 out-of-scope 报告 | `SECURITY.md:1-35` |
| IF-12 | `clear_graph` 遍历 group 调 `clear_data`，无任何鉴权 | `mcp_server.py:1102-1143` |
| IF-13 | `GraphDriver` 的 `clone` / `build_fulltext_query` 有默认实现（非 abstract） | `driver.py:131-150` |
| IF-14 | AGENTS 明文：`make test` 需 Neo4j；`test_add_triplet.py` 11 个既有失败且 CI 不跑；FalkorDB 并发断连 | `AGENTS.md:34-45` |
| IF-15 | `_search` 标注 DEPRECATED，`search_` 为正式入口 | `graphiti.py:1974-1990` |

## 二、与 Package 的比对判定

### 2.1 CONTRADICTED（事实错误，必须修正）

| ID | Package 说法 | 独立事实 | 影响 |
|---|---|---|---|
| V-01 | EK-14 / E-31：检索配方 **13** 个 | **16** 个（漏 `COMBINED_HYBRID_SEARCH_RRF/MMR/CROSS_ENCODER`） | 数量与覆盖描述失真 |
| V-02 | E-48：MCP **12** 个工具 | **13** 个（`get_status` 也是 `@mcp.tool()`） | 接口面少计 |
| V-03 | 规模：424 文件 / 302 py / 77,773 行 / 83 测试文件 | 386 / 300 / 76,764 / 81（排除 `.git`） | 初版口径不可复现，所有规模表述需重写 |

### 2.2 OVER_GENERALIZED（抽象过头，降级）

| ID | 原层级 | 判定理由 | 修正 |
|---|---|---|---|
| KO-04 请求级状态隔离 | Principle | 3 个回归（#1676/#1659/#1758）**同仓库、同维护者、同语言栈**，不构成跨项目证据；"principle" 应为 "strong pattern" | **降为 L3 Pattern**，标 `single-project, 3 independent regression sites` |
| KO-06 能力分层记账 | Principle | 仅 README + SECURITY 两份文档，无跨项目对照；"framework/reference/managed" 三层是本文档自述，不是独立验证的结构 | **降为 L3 Pattern** |
| KO-07 信任边界=写入边界 | Principle | 仅 SECURITY 文档 + 代码无鉴权事实；"写入边界"是我的推论，官方只说"应用决定 ingest 什么"；未实测投毒 | **降为 L3 Pattern**，并把"必然结果"改为"在该信任模型下成立" |

### 2.3 PARTIALLY_CONFIRMED（成立但表述需收紧）

| ID | 问题 | 应收紧为 |
|---|---|---|
| EK-03 | "双时间"是分析术语，仓库未使用该词 | "四个时间字段（业务 valid/invalid + 系统 created/expired），可视为双时间结构" |
| EK-10 | "重叠即共存"过度简化 | 实现是"**无法证明先后则不裁决**"；区间重叠只是其中一种情形 |
| EK-12 | Saga 增量的"只处理新增"未验证 | 保留字段事实，结论标 `Hypothesis`（移入 CAND-06） |
| EK-19 | Namespace "已部分落地"未量化 | 只陈述"目录存在（733 行）+ spec 标 Draft + 旧方法仍在"，不推断完成度 |
| EK-33 | "同一根因"是三 issue 的推断 | 改述为"三者共享同一类前提（请求状态放入实例/可变参数），是否同一根因需维护者确认" |
| KO-08 | "能力矩阵 + 显式 unsupported"是建议，非项目事实 | 标为 `Recommendation derived from Pattern`，不作为已验证模式 |

### 2.4 CONFIRMED（抽样复核通过）

EK-01, EK-02, EK-04, EK-05, EK-06, EK-07, EK-08, EK-09, EK-11, EK-13, EK-17, EK-22, EK-26, EK-27, EK-28, EK-29, EK-30, EK-31；KO-01, KO-02, KO-03, KO-05。

> 抽样方式：对每条抽取 1 个断言行，回到源码定位原文；命中即 CONFIRMED。未做逐条全量比对。

### 2.5 MISSING（遗漏）

| 遗漏 | 性质 |
|---|---|
| 反例预算 8/24 | 生产方已自报，Auditor 确认未达标；不是"未发现"，是"未完成" |
| `llm_client/` 其余 provider、`prompts/` 3,038 行、`search_utils.py` 2,075 行、`examples/` | Coverage 外围缺口；不影响已述结论，但限制"提示词/排序质量"类结论 |
| Flow Atlas 54 条仅抽样 12 条复核 | 其余 42 条未独立逐条回溯 |
| `build_communities` 的聚类算法未读 | KO-05/08 不依赖，但社区质量结论缺失 |

### 2.6 NEEDS_HUMAN_REVIEW

| 项 | 原因 |
|---|---|
| CAND-06 Saga 增量边界 | 需要构造乱序 episode 并实跑，环境不支持 |
| Kuzu 移除时间表 | 浅克隆无 issue/roadmap 历史，无法确认 |

## 三、判定统计

| 判定 | 数量 |
|---|---:|
| CONFIRMED | 22 |
| PARTIALLY_CONFIRMED | 6 |
| OVER_GENERALIZED（含降级） | 3 |
| CONTRADICTED | 3 |
| MISSING | 4 |
| NEEDS_HUMAN_REVIEW | 2 |
| **合计判定条目** | **40** |

## 四、三项最有价值的确认

1. **请求级作用域修复是真实的、结构性的**：`_resolve_request_scope` 的 docstring 与实现一致，且默认库路径刻意"不 clone 不 copy"，证明作者理解代价（IF-02）。
2. **SECURITY 的负向清单比 Package 转述更强**：原文明确"These limits are intentional. Do not expose these servers to an untrusted network."，说明这不是疏漏而是产品分层（IF-11）。
3. **去重的成本分层是真实的实现顺序**：identity → similarity → LLM，且 LLM 只接收 unresolved 集合，与"LLM 兜底"一致（IF-03/04）。

## 五、三项审计发现的错误（生产方需修正）

1. **配方数量 13 → 16**：漏计 `COMBINED_*` 三条组合检索配置，说明生产方只按 EDGE/NODE/COMMUNITY 分组清点，未枚举全文件。
2. **MCP 工具 12 → 13**：`get_status` 被当作辅助函数，实际是 `@mcp.tool()`。
3. **规模统计不可复现**：424/302/77,773/83 与 386/300/76,764/81 不一致，且未声明是否含 `.git`。**任何引用规模数字的地方都必须改用可复现口径。**

## 六、Benchmark Case（供 skill 回归使用）

**案例名**：`graphiti-single-source-principle`
**场景**：一个项目有 3 个同源回归、2 份强文档，但无跨项目证据。生产方把 3 条候选写成 `Principle`。
**期望行为**：Abstraction Auditor 必须将其降为 `Pattern`，并要求 `cross-project validation pending`。
**本案例结果**：Auditor 判定 3 条 OVER_GENERALIZED 并降级 → 与期望一致。
**用途**：作为"文档强度 ≠ 抽象层级"的反例样本，防止后续项目把"官方写得很明确"误当作"跨项目成立"。

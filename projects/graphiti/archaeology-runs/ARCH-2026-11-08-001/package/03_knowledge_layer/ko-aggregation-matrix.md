# KO Aggregation Matrix

| KO | 聚合规则 | 簇内 EK | 边类型依据 | 跨实例性（≥2 独立子系统/文件） | 解释范围扩大 | Epistemic |
|---|---|---|---|---|
| KO-01 | R1 机制簇 | 02,03,04,10,11 | mechanism ×2, causal ×2, contrast ×1 | nodes.py / edges.py / search_filters.py（3 子系统） | 从"边字段"→"记忆记录单元" | Pattern (cross-project pending) |
| KO-02 | R2 因果链簇 | 01,11,12,13,04 | causal ×4, dependency ×1 | graphiti.py / node_operations.py / edge_operations.py | 从"函数顺序"→"降不确定管线" | Pattern (cross-project pending) |
| KO-03 | R1 机制簇 | 07,08,09 | constraint ×2, mechanism ×1 | node_operations.py / search_utils.py（候选采集 vs 语义检索） | 从"去重阈值"→"分层裁决" | Pattern (cross-project pending) |
| KO-04 | R3 不变量簇 | 05,33,21 | constraint ×2, causal ×2 | graphiti.py / decorators.py / llm_runtime.py（写/读/模型三面） | 从"一个 bug"→"并发正确性不变量" | **Pattern**（审计由 Principle 降级：单项目/单团队） |
| KO-05 | R4 主题簇 | 06,18,31,05 | subsystem ×2, contrast ×2, constraint ×1 | decorators.py / pyproject / AGENTS.md（3 处不同材料） | 从"group_id 字段"→"逻辑×物理×失败模式" | Pattern (cross-project pending) |
| KO-06 | R3 不变量簇 | 26,27,22 | causal ×3, contrast ×1 | README / SECURITY.md / mcp_server（3 份独立文档） | 从"产品对照表"→"能力分层记账" | **Pattern**（审计由 Principle 降级：仅 2 份文档） |
| KO-07 | R4 主题簇 | 28,29,24,06 | causal ×2, dependency ×1, contrast ×1 | SECURITY.md / mcp_server.py / graphiti.py（3 处） | 从"没有鉴权"→"该信任模型下的写入边界" | **Pattern**（审计由 Principle 降级：单一文档 + 未实测） |
| KO-08 | R1 机制簇 | 17,19,32,31 | mechanism ×2, contrast ×2, constraint ×1 | driver.py / namespaces / pyproject / AGENTS.md | 从"多后端"→"能力矩阵可观测" | Pattern (cross-project pending) |

## 簇成立五条件核对

| KO | 内聚性 | 跨实例性 | 解释范围扩大 | 可命名 | 可回溯 |
|---|---|---|---|---|---|
| 01 | ✅ 同一机制（时间+溯源绑定） | ✅ 3 子系统 | ✅ | ✅ | ✅ E-05/10/13/26 |
| 02 | ✅ 完整因果链 | ✅ 3 文件 | ✅ | ✅ | ✅ E-11~17 |
| 03 | ✅ 同一机制（分层裁决） | ✅ 2 实现点 | ✅ | ✅ | ✅ E-20~24 |
| 04 | ✅ 同一不变量 | ✅ 写/读/模型 3 面 | ✅ | ✅ | ✅ E-18/19/46 |
| 05 | ✅ 同一主题（分区映射） | ✅ 3 材料 | ✅ | ✅ | ✅ E-04/06/53 |
| 06 | ✅ 同一约束（能力边界） | ✅ 3 文档 | ✅ | ✅ | ✅ E-51/52 |
| 07 | ✅ 同一主题（信任边界） | ✅ 3 处 | ✅ | ✅ | ✅ E-49/51 |
| 08 | ✅ 同一机制（能力降级） | ✅ 4 实现点 | ✅ | ✅ | ✅ E-33/38~43 |

## 反退化检查

- ❌ 无"因为都属于 graphiti_core"式聚合；
- ❌ 无 KO 缺少 `aggregation_rule`；
- ✅ 每条 EK 均被至少 1 个 KO 引用（见下）；
- ✅ 无悬空升维：8 个 KO 全部可回溯到 EK → Evidence。

## EK → KO 覆盖

| EK | 被引用于 |
|---|---|
| 01,11,12,13 | KO-02 |
| 02,03,04,10 | KO-01 |
| 05,33,21 | KO-04 |
| 06 | KO-05, KO-07 |
| 07,08,09 | KO-03 |
| 14,15,16 | 未升维（保留 L2；检索配方/分块属实现细节，不构成跨项目模式） |
| 17,19,32 | KO-08 |
| 18 | KO-05, KO-08 |
| 20 | 未升维（LLMRuntime 设计属项目局部实现） |
| 22,26,27 | KO-06 |
| 23,24,25 | KO-07（24）；23/25 未升维 |
| 28,29 | KO-07 |
| 30,31 | KO-05（31）/ 30 见 Candidates |
| 33 | KO-04 |

> 说明：未升维的 EK（14/15/16/20/23/25/30）**不是被丢弃**，而是保留在 Engineering Knowledge 层作为推理原材料。这符合"宽底座 + 窄尖顶"原则。

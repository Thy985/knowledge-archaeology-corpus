# Validation（考古方自评）

> 本文件是**生产方的自我检查**，不等于独立验证。独立 Auditor 的盲重建结果见 `independent_validation.md`（阶段⑤）。两者冲突时以独立验证为准，并在 Reconciliation 中显式记录。

## 1. Truth（真实性）

| 检查项 | 方法 | 结果 |
|---|---|---|
| 每条 EK 至少有 1 条 E 编号 | 脚本/人工核对 | 33/33 ✅ |
| 每条 E 有 file:line | 人工核对 | 53/53 ✅（行号为区间，精确点在行号处，未逐行 diff） |
| 关键数值（15 / 0.6 / 4） | 直接读常量定义 | ✅ `node_operations.py:67-68`、`content_chunking.py:36` |
| 版本号 0.31.2 | `pyproject.toml` + GitHub release | ✅ 两处一致 |
| 未把 spec 当实现 | 检查 E-43/E-44 标注 | ✅ 已标 S2/设计 |

**已知弱点**：部分 EK 的结论依赖"注释即事实"（如 `graphiti.py:1311` 对 #1676 的描述）。代码注释是**作者陈述**，不是运行证据——已在 Truth 维度降级为"注释级证据"。

## 2. Coverage（覆盖面）

| 覆盖 | 状态 |
|---|---|
| `graphiti_core/graphiti.py` | 完整读关键方法 |
| `nodes.py` / `edges.py` | 数据结构与方法签名 |
| `utils/maintenance/node_operations.py` | 去重主路径 |
| `utils/maintenance/edge_operations.py` | 签名/失效/时间 |
| `search/search.py` / `search_config_recipes.py` | 检索分派与配方 |
| `driver/driver.py` | ABC |
| `decorators.py` | 多 group 路由 |
| `llm_client/llm_runtime.py` + spec | 已覆盖 |
| `mcp_server.py` / `server/routers` | 端点与工具清单 |
| `SECURITY.md` / `README.md` / `AGENTS.md` | 已覆盖 |

| 未覆盖 | 影响 |
|---|---|
| `llm_client/` 其余 ~4,300 行（各 provider client） | Provider 差异未评估 |
| `prompts/` 3,038 行（extract_nodes 710 / extract_edges 等） | 抽取质量取决于 prompt，但未逐条评估 |
| `search/search_utils.py` 2,075 行 | 排序细节未验证 |
| `utils/bulk_utils.py` | bulk 路径未深读 |
| `examples/` | 未评估 |
| `server/tests`、`mcp_server/tests` | 未读 |

→ **Coverage 判定：核心机制覆盖，外围实现未覆盖。** 任何"Graphiti 的 prompt 设计如何"的结论都不应基于本轮。

## 3. Flow（流真实性）

| 检查 | 结果 |
|---|---|
| 54 条 Flow Edge 均有 file 证据 | ✅ |
| 七类流齐全 | ✅（Control 12 / State 7 / Data 9 / Evidence 6 / Authority 7 / Memory 6 / Policy 7） |
| 关键 Edge 可回溯 symbol/condition | ✅ |
| **是否运行验证** | ❌ 未运行（无 Neo4j/FalkorDB、无 Docker、无 API key） |

→ Flow 是**代码阅读级**证据（S3），不是运行级（S5）。所有"实际会怎样"的表述必须标注为推断。

## 4. Abstraction（抽象晋升）

| 检查 | 结果 |
|---|---|
| 是否每条 EK 都被迫升维 | ❌ 否，7 条 EK 保留 L2（见矩阵） |
| KO 是否都有 aggregation_rule | ✅ 8/8 |
| 是否有 L5 方法论 | ❌ 未产出（8 个 KO 最高到 L4/Principle，且标 cross-project pending） |
| 是否把单项目证据写成普遍定律 | ✅ 全部标注 pending |

**自评**：KO-04/KO-06/KO-07 标为 `Principle` 存在**过度晋升风险**——它们只有 Graphiti 单项目证据，虽有多处回归/文档互证，但未跨项目。独立审计应重点攻击这三条。

## 5. Counterexample（反例预算）

| KO | 反例数 | 是否满足 v3 要求（L3+ ≥3） |
|---|---:|---|
| KO-01 | 1（Mem0 ADD-only） | ❌ 差 2 |
| KO-02 | 1（bulk 合并阶段） | ❌ 差 2 |
| KO-03 | 1（别名多→全落 LLM） | ❌ 差 2 |
| KO-04 | 1（单租户 CLI） | ❌ 差 2 |
| KO-05 | 1（单后端部署） | ❌ 差 2 |
| KO-06 | 1（README 与代码一致的项目） | ❌ 差 2 |
| KO-07 | 1（单用户助理） | ❌ 差 2 |
| KO-08 | 1（单后端库） | ❌ 差 2 |

**这是本轮最明确的质量缺口**：反例预算未达标（8/24）。处理方式：**不伪造反例凑数**；在独立验证阶段要求 Counterexample Hunter 对 KO-04/KO-06/KO-07 重点攻击，未补足者降级为 `Pattern`。

## 6. Epistemic（认知状态诚实性）

| 检查 | 结果 |
|---|---|
| Fact / Observation / Hypothesis / Pattern / Principle 是否混用 | ✅ 分层：EK=L1/L2 事实与知识；KO=L3/L4；Candidates=Hypothesis |
| Candidates 是否被写成 KO | ✅ 未混入 |
| 未运行的内容是否标推断 | ✅ 已标 |

## 质量指标汇总

| 指标 | 值 |
|---|---|
| Evidence / Facts | **53** |
| Engineering Knowledge | **33** |
| EK 边 / 平均出度 / 孤立率 / 连通分量 | 95 / 2.88 / 0% / 1 |
| Generalized KO | **8** |
| Candidates | **7** |
| Flow Edges（七类） | **54** |
| 代码行覆盖 | 76,764 行中精读约 9,000 行（核心路径），约 11.7% |
| 运行验证 | **0**（环境无 DB/容器/key） |

## 结论

- ✅ 可交付：Project Layer、Engineering Knowledge、Flow Atlas、Candidates。
- ⚠️ 需独立验证：8 个 KO 的抽象层级（尤其 KO-04/06/07）、反例预算、注释级证据。
- ❌ 不可声称：任何"运行行为"结论、任何跨项目 Principle。

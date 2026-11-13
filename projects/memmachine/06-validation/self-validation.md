# Self-Validation — 交付前自检（阶段④ 内部）

> 这不是 Independent Validation。按 v3.2 纪律，本文件只做**作者侧自检**；阶段⑤ 的独立 Auditor 必须在不读本文件结论的前提下盲重建。本文件不得被当作“验证通过”的证据。
> 验证时段：2026-11-14，静态代码 + 文档，未执行测试/服务。

---

## 0. 本轮修正记录（作者自查发现并已修正）

| 编号 | 问题 | 原值 | 修正值 | 影响 |
|---|---|---|---|---|
| SV-01 | 测试函数计数用了错误 grep 模式（缺少 `\s*`，漏掉缩进函数） | 1,347 | **2,191**（全仓静态 `def test_`）；server_tests 1,856 | 只影响规模描述，不影响 KO |
| SV-02 | `_search_scored_declarative` 被描述为“无条件 `score >=`” | 无条件 | **`None or score >=`**；无方向自适应 | EK-37 措辞修正；结论（声明 vs 实现边界）不变 |
| SV-03 | C-06 原标 Hypothesis；作者侧曾据 delete 函数体升级为强候选 | Hypothesis | 审计前误升级；Independent Validation 以 FK `ON DELETE CASCADE` 推翻，最终降为条件性 Hypothesis | 见 reconciliation |

> 这三条是“自检发现并已改产物”的记录，保留在 Corpus，不隐藏。

---

## 1. Truth（真实性）

### 抽查方法

从 52 条 EK 中抽 12 条，逐条回读源码；抽 8 条 KO 的每条“关键句”，回读对应符号。

| 抽查对象 | 结论 | 证据 |
|---|---|---|
| EK-01 顶层编排 | 确认 | `main/memmachine.py` 字段与调用 |
| EK-02 facade 分派 | 确认 | `long_term_memory.py:240-286` |
| EK-08 全有或全无 | 确认 | `_resolve_ltm_defaults` 175-238 |
| EK-10 REST 无认证 | 确认 | `router.py` 无 Security 依赖 |
| EK-13 batch sorted uuid | 确认 | `main/memmachine.py:778-879` |
| EK-14 三路 gather | 确认 | 同上；`return_exceptions=True` |
| EK-17 内层无 return_exceptions | 确认 | `episodic_memory.py:209-244` |
| EK-20 五阶段 | 确认 | `event_memory.py:236-292` |
| EK-24 50 feature | 确认 | `semantic_ingestion.py:86,229` |
| EK-27 阈值 20 | 确认 | `semantic_ingestion.py:83` |
| EK-37 declarative 阈值 | **修正后确认** | `long_term_memory.py:306-313` |
| EK-45 STM eviction | 确认 | `short_term_memory.py:224-243` |
| EK-47 refcount | 确认 | `episodic_memory/instance_lru_cache.py` |
| EK-50 shared tables | 文档确认（S2，非实现） | `design/segment_store_shared_tables.md` |

**Truth 结论**：抽查 12/12 在修正后成立；0 条无法回溯；1 条（EK-50）只有 S2，不能当实现事实。

---

## 2. Coverage（覆盖）

### 已覆盖

| 面 | 覆盖 | 证据 |
|---|---|---|
| 顶层编排 | `main/memmachine.py` 全文关键段 | EV-016/065 |
| 两个 LTM backend | declarative / event 写入+检索+删除 | EV-026~032 |
| STM | 写入/驱逐/摘要/读取 | EV-034~037 |
| semantic | 写入/ingestion/consolidation/citation | EV-045~052 |
| API | REST / MCP / config | EV-011~014、075 |
| 状态 | session / cache / delete | EV-041~043、049 |
| 存储设计 | segment shared tables | EV-074 |
| 测试 | 121 文件静态阅读（未执行） | EV-077 |
| 治理 | GOVERNANCE/SECURITY | EV-079/080 |

### 明确缺口（coverage gap）

1. **向量/图存储内部**：Qdrant / Milvus / Neo4j / pgvector 的查询计划、过滤下推、距离归一化未逐行读；只确认接口与配置。
2. **client SDK**：`packages/client` 的重试/错误处理未读；影响 C-02 的“调用方是否可修复”判断。
3. **测试未执行**：S4=0；所有“测试揭示了什么”只能算静态阅读，不是执行证据。
4. **文档未全读**：`docs/` 只读了 `agentic_memory.mdx` 与 README；未读 API reference 全部。
5. **部署安全**：helm ingress/network policy 未读；C-03 的部署判断不完整。
6. **多 worker 运行时**：SQL lease / 状态轮询只在代码层确认，未跑多进程。

**Coverage 结论**：核心链路覆盖充分；存储内部与部署面覆盖不足，已在 `06_validation/coverage-gaps.md` 列出。

---

## 3. Flow（流）

- 七类流全部存在，共 **72 条 Edge**：Control 17（含 4 异常）、State 15、Data 9、Evidence 7、Authority 6、Memory 10、Policy 8。
- 每条 Edge 都有 symbol/file:line 或明确的“断点”标记。
- **Flow→KO 交叉校验**：8 个 KO 全部能回溯到至少 2 条 Flow Edge；KO-07 的 F-E-06 是显式断点（citation 删除联动）。
- **Flow 结论**：无“KO 与 Flow 矛盾”的硬冲突；有 1 个部分一致（KO-07）。

---

## 4. Abstraction（抽象）

| KO | 层级 | 是否过度升维 | 理由 |
|---|---|---|---|
| KO-01 | L3 candidate | 否 | 跨 declarative/event/retrieval/delete 四条路径；有 3 反例 |
| KO-02 | L3 candidate | 否 | 完整因果链；明确“重试可能重复”等反例 |
| KO-03 | L3 candidate | 否 | 跨配置/资源/凭证；明确 STM 可存活等反例 |
| KO-04 | L3 candidate | 否 | 跨两个 backend + API；明确单位差异 |
| KO-05 | L3 candidate | 否 | 跨 REST/MCP/session/semantic；明确网关可补足 |
| KO-06 | L3 candidate | 否 | 跨 semantic 写入/ingestion；明确 semantic 可关 |
| KO-07 | L3 candidate | 否 | 完整 consolidation 链；明确“可能不删”等反例 |
| KO-08 | L3 candidate | 否 | 跨 STM/cache/session/segment；明确 capacity=0 等反例 |

- **L4/L5 数量：0**。没有把项目经验写成“普遍定律”。
- **KO 平均簇规模：6.5**（52 条 EK 引用，去重前）；在 3~12 范围内。
- **每个 KO 3 条反例，共 24 条**；无反例预算缺口。

**Abstraction 结论**：8 个 KO 均为 L3 candidate，Epistemic=Observation；没有 L4/L5 悬空升维。

---

## 5. Counterexample（反例）

| KO | 反例数 | 最强反例 | 是否影响 KO 成立 |
|---|---:|---|---|
| KO-01 | 3 | event 返回 segment 窗口，不是完整 episode | 收窄 scope，不推翻 |
| KO-02 | 3 | 重试可能重复而非丢失 | 收窄“变少”表述 |
| KO-03 | 3 | STM 可在 LTM 关闭时存活 | 收窄“整层”到 LTM |
| KO-04 | 3 | `expand_context=0` 单位仍不同 | 强化而非削弱 |
| KO-05 | 3 | 网关可补足认证 | 收窄到“直连部署” |
| KO-06 | 3 | semantic 可关闭 | 收窄到启用 semantic |
| KO-07 | 3 | 可能不删 / context 超限中止 | 收窄到“确有删除” |
| KO-08 | 3 | capacity=0 不缓存 | 收窄到长期缓存 |

**Counterexample 结论**：无反例导致 KO 被推翻；8 个 KO 均需在 scope 内成立。

---

## 6. Epistemic（认知状态）

| 类型 | 数量 | 说明 |
|---|---:|---|
| EV S3 | 90 | 静态代码/测试/配置 |
| EV S2 | 2 | README/文档声称（EV-081/082） |
| EV S4 | 0 | 未执行测试 |
| EV S5 | 0 | 未启动服务 |
| EK Fact | 47 | 直接读出 |
| EK Observation | 4 | 跨 2+ 证据归纳（EK-34/50 等） |
| KO Observation | 8 | 全部 single-project, code-level |
| KO Validated Pattern | 0 | 无跨项目/运行验证 |
| Candidate Hypothesis | 12 | 全部在 05_candidates |
| Principle / Law | 0 | 未写 |

**Epistemic 结论**：没有把 Hypothesis 写成 Fact；没有把 single-project Observation 写成 Principle；EK-50 的 S2 已显式标注。

---

## 7. 质量指标汇总

| 指标 | 值 | 门槛 | 结果 |
|---|---:|---|---|
| EK 数 | 52 | 40~60 | PASS |
| EK 平均出边 | 2.79 | ≥1 | PASS |
| 游离 EK | 0% | <20% | PASS |
| EK target 缺失 | 0 | 0 | PASS |
| KO 数 | 8 | 7~12 | PASS |
| KO 聚合规则覆盖 | 100% | 100% | PASS |
| KO 反例 | 3/KO | ≥3 | PASS |
| Flow 类别 | 7/7 | 7 | PASS |
| Flow Edge | 72 | 覆盖核心 | PASS |
| Evidence S3 占比 | 90/92 | 无硬门槛 | PASS（诚实） |
| S4/S5 | 0 | 无硬门槛 | 标注为缺口 |
| 测试执行 | 未执行 | 无硬门槛 | **GAP** |

---

## 8. 未解决的矛盾（交给 Independent Auditor）

1. **README 拓扑 vs 代码**：README 称 Episodic→Neo4j、Profile→SQL；代码存在 Nebula/SQLite/Qdrant 多后端。哪一个是“权威描述”未裁决（EV-081 vs EV-030/074）。
2. **`-inf` 是否真能触发**：`SimilarityMetric` 默认 COSINE（`data_types.py:32`），C-01 需要显式配置 EUCLIDEAN；默认部署不会触发。不能写成“默认必现”。
3. **declarative 阈值方向**：当前 declarative 强制 reranker，所以 `score >=` 成立；若未来允许无 reranker，docstring 的方向承诺会失效。
4. **citation 删除**：PG 后端 `delete_features` 不删关联表；Neo4j `DETACH DELETE` 删属性。是否构成“孤儿证据”取决于后续 `delete_history` 是否覆盖；未运行验证。
5. **测试数口径**：静态 `def test_` 计数 2,191；pytest collect 可能不同（参数化会展开）。本文件只声明静态计数，不声称“2,191 个用例”。

---

## 9. Reconciliation 后记（非作者自检）

- C-06 在 Independent Validation 中被 `citation_association_table.feature_id ON DELETE CASCADE` 推翻，已降为条件性 Hypothesis；本文件 §0 的 SV-03 记录保留为“审计前判断”，不作为最终结论。
- EK 由 51 增至 52（EK-52），Evidence 由 91 增至 92（EV-092），平均簇规模由 6.8 修正为 6.5。

## 10. 自检结论

**CONDITIONAL PASS（待独立验证）**。

- 通过项：EK 图结构、KO 聚合规则、反例预算、Epistemic 标注、七类流覆盖。
- 未通过/待验证项：S4/S5=0（未跑测试/服务）、存储内部覆盖不足、部署安全未读、C-01/C-06 未运行验证。
- **不宣称“全绿”**：本轮没有执行测试，也没有启动服务；任何 `Validated` 表述都不成立。

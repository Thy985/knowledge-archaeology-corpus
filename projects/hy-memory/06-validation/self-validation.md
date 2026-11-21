# 06.1 · Self Validation（六类自检）

> 执行者：Orchestrator 自检（非独立 Auditor）。所有"通过"仅代表"在当前证据范围内未发现反证"，不代表已运行验证。

## 1. Truth（真实性）

**方法**：对 28 条 EK 随机抽取 12 条，回读其引用的 `file:line`，确认符号存在、分支条件与陈述一致。

| 抽查 | EK | 结论 |
|---|---|---|
| 1 | EK-01 `_normalize_scene` | PASS（`client.py:861-890` 存在，非法抛 ValueError） |
| 2 | EK-02 无 tool 短路 | PASS（分支注释与实现一致） |
| 3 | EK-03 judge 三分类 | PASS |
| 4 | EK-04 fail-soft | PASS（`defaulting to chat` 文本存在） |
| 5 | EK-05 强制 coding 回退 | PASS |
| 6 | EK-07 L1 non-fatal | PASS（`_l1_error` 命名一致） |
| 7 | EK-08 reconcile fail-closed | PASS（无 ADD-all 兜底） |
| 8 | EK-09 SHADOW | PASS |
| 9 | EK-12 召回层白名单 | PASS |
| 10 | EK-17 链零 LLM | PASS |
| 11 | EK-24 root_identity | PASS |
| 12 | EK-27 loopback 解析 | PASS（注释含 `127.attacker.example` 反例） |

**发现的问题**：
- **KV-01**：`README.md` 把 L4 标为 "Reserved"，但 `reader_base.py:146-155` 明确把 `L4_IDENTITY` 纳入召回。已按**实现为准**记录，并在 01.2 标注为 KNOWLEDGE_CONFLICT。→ 已修正文档层陈述（EK-12 未采用 README 说法）。
- **KV-02**：`00_overview.md` 初稿的质量自评数字（Facts 32 / EK 25 / 边 34 / Flow 24）与逐条建表后的实际值（36 / 28 / 57 / 46）不一致。→ 已在 Overview 与各文件同步为实际值。
- **KV-03**：`README.md` 的 `fact_chain_summary` 未在代码中定位（C-02）。→ 已降级为 Candidate，未写入 EK。

## 2. Coverage（覆盖）

**已覆盖的子系统**：scene 路由、Chat 写入、reconcile/版本化、Chat 读取主干、productivity 双层、版本链、MemHub 授权/隔离/本地 API、配置、CI。

**未覆盖（显式）**：
| 区域 | 规模 | 影响 |
|---|---|---|
| `memhub/apps/desktop` + `web` + `packages/*` | ~182k 行中的大部分 | 产品层 UI 行为未知 |
| `pipelines/system2_agent.py` | 2,476 行 | L6 schema 归纳算法未知 |
| `reader_base.py` 的 rerank/图扩展分支 | 3,693 行中的多数 | 检索排序细节未知 |
| `prompts/` 全量 | — | 抽取质量的决定因素未评估 |
| 40+ `chore/*` 分支 | — | 版本差异未知 |

**覆盖率自评**：精读 ~1,150 行 / 322,396 行 ≈ **0.36%**。这不是"全仓库考古"，而是"主干抽样"。Coverage Auditor 若要求"重要子系统全覆盖"，本轮**不通过**；因此 Overview 已显式声明覆盖边界，不宣称完整。

## 3. Flow（流真实性）

- 46 条 Edge 均给出 symbol/file/condition。
- **未验证**：
  - `DF-03`（抽取失败后是否仍进入 reconcile）— 需读 `_run_agent` 的异常处理。
  - `SF-06`（`fact_chain_summary`）— 未定位实现。
  - `MF-03`（System 2 失败后的状态）— 只读 docstring。
- 其余 43 条可在代码中定位到对应分支。

**结论**：Flow 层 `PARTIALLY_VERIFIED`。

## 4. Abstraction（抽象层级）

| KO | 判定 | 理由 |
|---|---|---|
| KO-01 | L3 成立 | 5 条 EK 跨 client/judge；模式可命名、可迁移 |
| KO-02 | L3 成立 | 3 条 EK 形成完整因果链；有明确取舍 |
| KO-03 | L3 成立 | 3 条 EK 维护同一不变量 |
| KO-04 | **保持 L2** | 簇规模 2，仅 coding 子系统；升 L3 会过度概括 |
| KO-05 | L3 成立 | 5 条 EK 形成"决策→解析→过滤→消费"完整链 |
| KO-06 | L3 成立 | 3 条 EK 汇聚到"权威单一" |
| KO-07 | L4，标 pending | 4 条 EK 跨四个子系统；但"用户维度是声明式的"削弱普适性 |

**未升维的内容**：所有 L1/L2 工程知识（28 条 EK）全部保留，未因形成 KO 而删除；KO-04 明确停在 L2。

**L5 = 0**：没有足够证据提出"如何行动"的准则；不强行凑数。

## 5. Counterexample（反例）

- L3+ KO 共 6 个，每个 3 条定向反例，合计 18 条。
- 反例类型分布：成本/复杂度（4）、语义歧义（3）、无界增长/保留（2）、权限模型边界（3）、迁移/兼容（1）、审计需求（1）、可观测性（2）、其他（2）。
- **无 KO 因反例被降级**，但 KO-05 与 KO-07 的适用边界被显著收窄（分别加"必须有保留策略""键前缀化不防故意构造"）。

## 6. Epistemic（认知状态）

| 检查 | 结果 |
|---|---|
| 厂商指标是否进入 Fact 层 | ❌ 未进入；仅 Overview §5 标"文档声明" |
| Hypothesis 是否被写成知识 | ✅ 6 条全部在 05_candidates，未进入 EK/KO |
| L4 是否标 cross-project pending | ✅ KO-07 已标 |
| 文档声明是否被当作实现事实 | ⚠️ 3 处：`fact_chain_summary`（降为 C-02）、`L4 reserved`（按实现纠正）、`417/536` 统计（标为"代码注释自述，未独立复核"） |
| 单一项目证据是否写成普遍定律 | ✅ 所有 L3 标 `Validated Pattern (single-project)` |

## 7. 自检结论

| 维度 | 状态 |
|---|---|
| Truth | PASS（抽查 12/12 一致；3 处文档冲突已处理） |
| Coverage | **PARTIAL**（主干覆盖，0.36% 抽样；产品层未覆盖） |
| Flow | PARTIAL（43/46 可定位，3 条未验证） |
| Abstraction | PASS（1 条保持 L2，1 条 L4 pending，无 L5） |
| Counterexample | PASS（6×3=18 条） |
| Epistemic | PASS（7 条 candidates 隔离，指标未入事实层） |

**不通过项**：Coverage 未达"全子系统"标准。本轮定位为 **Memory 引擎主干考古**，不宣称覆盖 MemHub 产品层与 System 2 算法。

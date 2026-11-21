# Independent Validation（Blind Reconstruction）

> 独立 Auditor：不把 Package 当作事实来源。先独立重读仓库建立 findings，再与 Package 对比。
> 只读；未修改任何考古产物。范围：`main@9737431c`，聚焦 Package 的 28 条 EK / 7 个 KO / 46 条 Flow Edge。
> 证据上限：S3（未运行）。

## 1. 独立重建方法

1. 先读 `client.py` 的 add/search 入口、`coding/store.py`、`coding/chain_store.py`、`coding/writer.py`、`reader_base.py` 召回层、MemHub 的 `auth/user_scope/grants/api`。
2. 对每个 Package 结论，先写"我从代码里看到什么"，再打开 Package 对照。
3. 重点攻击：单案例→Pattern、Pattern→L4、ADR/README→实现事实、Flow Edge 真实性、bypass/fallback 路径、Epistemic 混用。

## 2. 判定统计

| 判定 | 数量 |
|---|---:|
| CONFIRMED | 6 |
| PARTIALLY_CONFIRMED | 2 |
| DOWNGRADED | 0 |
| OVER_GENERALIZED | 0 |
| MISSING（包内未覆盖的新事实） | 2 |
| CONTRADICTED | 0 |
| NEEDS_HUMAN_REVIEW | 0 |
| **合计** | **10** |

---

## 3. 三个被确认的成功判断

### V-01 · CONFIRMED：`coding_available` 是四重 AND 门，不是"scene 一判就进 coding"
- **独立观察**：`client.py:2954-2960` 的 `coding_available` 同时要求 `mode != "lite"`、`content_type == "messages"`、`config.coding is not None`、`config.coding.enable`。只有四个条件全真，才进入 `if scene_forced == "coding" or has_any_tool_message(...)` 分支。
- **Package 对照**：EK-05 与 CF-07 覆盖了"强制 coding 无 tool 回退"，但**没有覆盖"coding 功能被禁用"这一前置门**。
- **结论**：Package 的"回退 chat"结论方向正确，但漏了一个前置条件。判 CONFIRMED（方向）+ MISSING（细节），见 V-07。

### V-02 · CONFIRMED：链不进入 Chat 检索，`reader_base.py` 无任何 chain 引用
- **独立观察**：`grep "chain" reader_base.py` 只命中 `expand_evolution_chains`（记忆演化链，不是代码版本链）。`HyCodeMem.recall()` 只在 `coding/chain_store.py:1058` 定义，由 productivity 侧调用。
- **Package 对照**：EK-22 断言"链只装饰检索结果、不是第二检索源"。
- **结论**：CONFIRMED。且这是**结构性证据**（不是文档声明）：Chat 召回层根本没有导入 chain 模块。

### V-03 · CONFIRMED：`user_id` 由请求体声明，默认 `local`，不从认证态推导
- **独立观察**：`api.py` 多处 `user_id: UserId = DEFAULT_USER_ID`（`:500/508/515/861/1105`）；隔离靠 `owns()` 与字符集，不靠 token 身份映射。
- **Package 对照**：KO-07 的反例 1 已写"用户维度是声明式的，不是从认证态推导的"。
- **结论**：CONFIRMED，且该反例是 KO-07 最重要的适用边界。**不应**把它升格为"隔离完备"。

---

## 4. 三个被纠正的错误 / 过度判断

### V-04 · PARTIALLY_CONFIRMED：EK-16 "阈值无硬编码常量"是范围性结论
- **独立观察**：`coding/writer.py:188` 只有 `no drafts produced (value bar)` 日志；`grep threshold|min_confidence|MIN_VALUE` 在 `coding/*.py` 未命中。但抽取阈值可能在 prompt 文本或 `agent/prompts_*.py` 中，本轮未读全量 prompt。
- **Package 对照**：EK-16 已标 "Fact（行为）/ Observation（无数值常量基于本轮 grep 范围）"。
- **判定**：PARTIALLY_CONFIRMED。**不可**写成"阈值一定由模型判断"；正确表述是"代码路径中未见数值门槛，阈值出现在抽取判断环节"。

### V-05 · PARTIALLY_CONFIRMED：KO-02 的"可重跑"缺少 L1 幂等证据
- **独立观察**：`coding/store.py:1016-1027` 的 `insert_fact` 明确"先向量化再落 SQLite，不落没有向量的事实"，支持"派生写入 all-or-nothing"；但 **L1 raw 的 vector upsert id 规则未读**，无法确认同 request 重跑不产生重复 raw。
- **Package 对照**：KO-02 的 L3 结论依赖 EK-28（request_id 幂等）；EK-28 自己已标 Observation。
- **判定**：PARTIALLY_CONFIRMED。KO-02 的**模式**成立（两类失败策略并存是事实），但其**前提**（可安全重跑）未闭环。

### V-06 · 纠正：`fact_chain_summary` 存在，C-02 的"未定位"不成立
- **独立观察**：`coding/store.py:112` 的 `CREATE TABLE` 含 `fact_chain_summary TEXT NOT NULL DEFAULT ''`；`:364` 读取、`:1425/1441` 在 upsert 中写入、`:1454` 落值。
- **Package 对照**：C-02 写"`coding/store.py` 中未见同名常量（grep 未命中）"，并作为"文档过度承诺"的风险。
- **判定**：C-02 的**风险假设被推翻**（字段存在且参与写入），但"其语义/是否完整保留历代说法"仍未被本轮读完。应改为 **CONFIRMED（字段存在）+ 语义待读**，而不是"未定位"。
- **影响**：Package 的 01.2 §4.3 与 05_candidates C-02 需要修正表述；EK 层不受影响（未把它写成知识）。

---

## 5. 遗漏（Package 未覆盖的重要事实）

### V-07 · MISSING：`coding.enable=False` 时，强制 `scene="coding"` 被静默忽略，无任何 warning
- **独立观察**：`client.py:2976-2978`：`elif coding_available and (scene_forced == "coding" or has_any_tool_message(...))`。当 `coding_available=False`（例如 `config.coding.enable=False`）时，整个分支不进入，直接落到 chat 路径；`scene_forced=="coding"` **不会触发任何日志**。
- **与 Package 的差异**：EK-05 / CF-07 只描述"有 coding_available 但无 tool → warning + 回退"。遗漏了"功能被禁用 → 静默降级"这条更隐蔽的路径。
- **影响**：这是 KO-01 "fail-soft" 模式的一个**未被记录的实例**；也解释了为什么"用户以为写了 coding memory"可能完全无感知。应补入 EK-05 的 failure_path 与 Flow CF-07。
- **严重度**：中（不丢数据，但静默改变行为）。

### V-08 · MISSING：coding 场景 `memory_id` 取首个 `experience_id`，无经验时为 `None`
- **独立观察**：`client.py:3055-3060`：`resp["memory_id"] = _coding_eids[0] if _coding_eids else None`。而 WorkingFact 已写入（每 session 1 条），但**不体现在 memory_id**。
- **与 Package 的差异**：01.2 描述了双层结构，但未指出这个 API 契约的后果：调用方按 `memory_id` 判断"是否写入成功"时会误判（fact 成功、experience 为空 → `memory_id=None`）。
- **影响**：对下游集成是真实的契约陷阱；建议在 EK-15 增加一条"返回值只暴露 experience，不暴露 fact"的边界说明，或单列 EK。
- **严重度**：中（会误导调用方，但不丢数据）。

---

## 6. Flow Edge 真实性复核

抽查 10 条（CF-03/04/05/07/08/09, DF-01/07, AF-01/05, PF-05）：

| Edge | 结果 |
|---|---|
| CF-03 无 tool 短路 | ✅ 分支存在，位置在 judge 之前 |
| CF-04 judge 调用 | ✅ |
| CF-05 coding writer | ✅ |
| CF-07 强制 coding 无 tool | ✅ 有 warning |
| CF-08 reconcile fail-closed | ✅ 无 ADD-all |
| CF-09 chain 挂载 | ✅ 条件 `scene_variant=="coding" and resp.success` |
| DF-01 L1 raw | ✅ |
| DF-07 patch 三来源 | ✅ 方法名与分支存在 |
| AF-01 authorized_root | ✅ |
| AF-05 local_api_guard | ✅ |
| PF-05 ultra gate | ✅ |

**漏项**：Flow Atlas 未记录 `coding.enable` 门（V-07），应补 CF-07b。

---

## 7. Epistemic 复核

| 检查 | 结果 |
|---|---|
| 是否有 Hypothesis 被写成 Fact | 未发现；C-01~C-07 均在 Candidates |
| L4 是否过度概括 | KO-07 标 pending 且列 3 条反例；但"隔离=键前缀化"仍偏强，建议在 corpus 中保留 pending 标记 |
| 文档声明是否当成实现 | C-02 曾误判为"未定位"（V-06）；其余 README 声明（厂商指标、417/536）已隔离 |
| 单项目 Pattern 是否写成通用 | 未发现绝对化措辞（均带 single-project） |

## 8. Benchmark Case（供 skill CI 使用）

**Case ID**: `hy-memory-scene-routing-failsoft`
**类型**: 反例敏感型（Fallback / Precondition）
**输入**: `client.py:2930-3065` 的路由段 + `coding/judge.py:154-230`
**期望**：
1. 识别三条 fail-soft 路径（judge 异常 / 无 tool / 强制无 tool）；
2. 识别**第四条**：`coding.enable=False` 时强制 coding 静默落到 chat；
3. 不把"回退 chat"写成"报错"；
4. 不把 README 的 L4 reserved 当成召回事实。
**评分**：命中 4/4 = PASS；漏第 2 条 = FAIL（本轮 Package 即漏此项，可作为回归样本）。

## 9. 结论

- Package 的主干判断**方向正确**：三场景路由、L1 non-fatal vs reconcile fail-closed、SHADOW 语义、链的附件定位、授权身份绑定，均能在代码中复现。
- **2 条遗漏**（V-07/V-08）需要写入 Corpus 的 reconciliation 与 EK/Flow 修正。
- **1 条错误表述**（V-06: `fact_chain_summary` 未定位）需要修正 Candidates。
- 不建议做任何"升维"调整：KO-07 维持 L4 pending；KO-04 维持 L2。

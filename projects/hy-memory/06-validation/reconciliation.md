# 06.2 · Reconciliation

> 输入：Package（阶段④）+ Independent Validation（阶段⑤）。
> 原则：先处理"证据与结论不一致"，再统一数字口径；不修改生产 Skill，只修 Corpus Artifact。

## 1. 判定汇总

| 判定 | 数量 | 处理 |
|---|---:|---|
| CONFIRMED | 6 | 保留原结论 |
| PARTIALLY_CONFIRMED | 2 | 收窄表述，补"未闭环前提" |
| MISSING | 2 | 新增 EK-29 / 补 EK-05 边界与 CF-07b |
| CONTRADICTED | 0 | — |
| OVER_GENERALIZED | 0 | — |
| DOWNGRADED | 0 | — |
| NEEDS_HUMAN_REVIEW | 0 | — |

**结论**：Package 主干成立；需 3 处修订（1 处新增 EK、1 处补边界、1 处改 Candidate 表述）。

## 2. 逐项修订

### R-01 · 补 `coding.enable=False` 的静默降级（来自 V-07）
- **原状**：EK-05 / CF-07 只写"强制 coding 无 tool → warning + 回退"。
- **修订**：在 EK-05 的 failure_path 增加第二条：
  - `coding_available=False`（`mode=lite` / 非 messages / `coding` 配置缺失 / `coding.enable=False`）时，整个 productivity 分支不进入，`scene="coding"` 被**静默忽略**，无 warning。
  - Flow Atlas 增加 **CF-07b**。
- **影响面**：KO-01 的"保守降级"模式多一个实例；C-01（可观测性）证据增强。
- **证据**：`client.py:2954-2960`（四重 AND）、`:2976-2978`（elif 条件）。
- **状态**：ACCEPTED。

### R-02 · 新增 EK-29：coding 的 `memory_id` 只暴露 experience，不暴露 working fact（来自 V-08）
- **原状**：EK-15 描述双层结构，但未写返回值契约。
- **修订**：新增 EK-29，并在 EK-15 加一条 `constraint` 边指向 EK-29。
- **证据**：`client.py:3055-3060`。
- **影响面**：下游集成若以 `memory_id is not None` 判断成功会误判。
- **状态**：ACCEPTED。

### R-03 · 修正 C-02：`fact_chain_summary` 存在（来自 V-06）
- **原状**：C-02 写"未在代码中定位该字段"。
- **修订**：改为"字段存在并参与 upsert（`coding/store.py:112/364/1425/1441/1454`）；未验证的是'是否保留历代说法'与'检索时是否展开'"。
- **影响面**：风险等级从"文档过度承诺"降为"语义待读"。
- **状态**：ACCEPTED。

### R-04 · 收窄 EK-16 与 KO-02 的表述（来自 V-04/V-05）
- EK-16 保留"代码路径未见数值门槛"，但删除任何"阈值一定由模型判断"的暗示（原文已标 Observation，无需改，仅在 Corpus 中加粗提醒）。
- KO-02 增加一句前提："该模式的成立依赖 L1 写入幂等；本轮未验证 vector upsert 的 id 规则（C-03）"。
- **状态**：ACCEPTED（表述性修订）。

## 3. 数字口径统一（以本表为最终值）

| 指标 | 最终值 | 说明 |
|---|---:|---|
| Facts（Evidence Index） | 36 | 逐条建表后计数 |
| EK 节点 | 29 | 原 28 + EK-29 |
| EK 声明边 | 58 | 原 57 + EK-15→EK-29 |
| 平均出边 | 2.00 | 58/29 |
| 游离 EK | 0 | — |
| KO | 7 | L3×5 / L2×1 / L4×1 |
| L5 | 0 | 不强行升维 |
| Candidates | 7 | 原 6 + C-07（厂商指标可比性） |
| Flow Edge | 47 | 原 46 + CF-07b |
| 精读代码 | ~1,150 行 | ≈0.36% of 322,396 |
| 证据上限 | S3 | 未运行 |

> 说明：初稿自评（32/25/34/24）为估算值，已被本表取代。Corpus 中所有文件以本表为准。

## 4. 未采纳的建议

| 建议 | 原因 |
|---|---|
| 把 KO-07 升为"通用隔离定律" | 仅一个项目、且 user 维度为声明式；维持 L4 pending |
| 把 KO-04 升 L3 | 簇规模 2、仅 coding 子系统；维持 L2 |
| 为 `fact_chain_summary` 补 EK | 只确认字段存在，未读历代累积逻辑；留在 Candidate |
| 删除曾出现"未定位"的 C-02 | 保留并修正，保留考古过程的可追溯性（不覆盖历史判断） |

## 5. 是否产生 Skill Improvement Proposal

**否（本轮 P0/P1 均不成立）**。

理由：
- 发现的 2 条 MISSING 属于**目标项目实现细节**，不是 skill 的方法论缺陷。
- Coverage 未达"全子系统"是**本轮范围裁剪**，已在 Package 显式声明，不属于 skill 失效。
- 唯一可讨论的改进是"Blind Reconstruction 应对 `feature flag` 前置门做专项检查"，但这属于执行清单的增量，风险 Low，可留待多轮样本后再提 P2。

→ 本轮不创建 Skill Evolution Candidate Branch/PR。

## 6. 与历史 Run 的关系

- 不覆盖 `projects/` 下任何已有项目；`hy-memory` 为新增顶级目录。
- 与 Letta 的对照写入 `00-overview.md` §4，不修改 Letta 目录。
- 本 Run 的 commit / skill version 见 `archaeology-runs/ARCH-2026-11-10-001/run-metadata.yaml`。

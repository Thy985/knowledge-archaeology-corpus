# Reconciliation（独立审计 → 修正采纳）

> 输入：`06-validation/independent-validation.md`（20 条判定）
> 原则：不覆盖审计原文；修正写在 Package 内并在本文件留痕。

## 1. 处置表

| 审计判定 | 涉及条目 | 处置 | 修正内容 |
|---|---|---|---|
| CONTRADICTED | EK-06 / KO-02 | **ACCEPTED + MODIFIED** | 改写为"非 git 模式写 DB；git 模式 `commit to git` → `sync to PG cache`；两模式均无跨存储事务" |
| OVER_GENERALIZED | KO-04 | **ACCEPTED + DOWNGRADED** | L3 → **L2 Observation**；标题改为"无法由资源级属性表达"；删除"否则共享与最小权限不可兼得"的规范性结论 |
| OVER_GENERALIZED（轻度） | KO-01 | **ACCEPTED + MODIFIED** | 删除"应独立设计"的规范性措辞，保留三层结构的实现事实 |
| MISSING | `_move_block_to_sequence` 重复定义 | **ACCEPTED** | 新增 EK-21（failure_path） |
| MISSING | git→PG 同步漏 `read_only`/`metadata` | **ACCEPTED** | 新增 EK-22（failure_path），并作为 KO-02 的补充证据 |
| MISSING | `rethink_memory` 可创建 block | **ACCEPTED** | 新增 EK-23（boundary） |
| MISSING | `managers/` 未覆盖 | **RETAINED AS BOUNDARY** | 不补写知识；在 `run_metadata.yaml` 的 coverage 中显式列为未覆盖面 |
| PARTIALLY_CONFIRMED | EK-02 Archival 注入路径 | **RETAINED + 标注** | 保留结论，标注"检索存在已确认；注入 prompt 的具体路径未逐行定位" |
| PARTIALLY_CONFIRMED | EK-16 默认压缩策略 | **RETAINED + 标注** | 标注"策略实现已确认，默认启用哪一种未确认"（CAND-05） |
| PARTIALLY_CONFIRMED | KO-03 影子路径 | **RETAINED + 降强** | 表述由"防护是否生效取决于路径"改为"legacy 实现存在且无校验；可达性待 CAND-01" |
| NEEDS_HUMAN_REVIEW | `read_only` deprecated 意图 | **ESCALATED** | 留在 Candidates（CAND-04），不进 KO |
| CONFIRMED（10 条） | 数字、read_only 9 处、legacy 无校验、git-as-truth 注释、双写、级联、compile、多对多、字符串比较 | **ACCEPTED AS IS** | 无修改 |

## 2. 最终质量指标

| 指标 | 修正前 | 修正后 |
|---|---:|---:|
| Facts | 40 | 40 |
| EK | 20 | **23** |
| EK 边 | 25 | **30** |
| EK 平均出边 | 1.25 | **1.30** |
| 游离 EK | 0 | 0 |
| KO | 6 | 6（L3×4 / L2×2） |
| Candidates | 6 | 6 |
| Flow Edge | 28 | 28 |
| CONFIRMED 率（审计判定） | — | 10 / 20 = 50% |
| CONTRADICTED | — | 1（已修正） |

## 3. 保留争议（不强行统一）

1. **`read_only` 弃用意图**（CAND-04）：代码只能证明"标记 deprecated 且仍在使用"，不能证明团队是否在迁移权限模型。
2. **legacy 工具可达性**（CAND-01）：决定 KO-03 强度；本轮未验证，不升格。
3. **共享 block 写扩散**（CAND-06）：`blocks_agents` 多对多 + 单行更新在结构上支持扩散，但未构造测试；KO-04 因此停在 Observation。
4. **git 模式并发**（CAND-02）：git-first 顺序已确认，但失败/并发时的补偿行为未验证。

## 4. Reconciliation 结论

修正后的 Package 满足提交条件：
- 事实层无 CONTRADICTED 残留；
- 抽象层无 L4/L5 悬空；
- 所有 Hypothesis 均在 Candidates；
- 覆盖缺口已在 metadata 中显式声明。

**仍未解决（需下游/人工）**：CAND-01、CAND-04、CAND-06、`managers/` 覆盖。

# MemMachine — Knowledge Archaeology

> 考古对象：`MemMachine/MemMachine` @ `377510d`（main，2026-11-02）
> 考古日期：2026-11-14 · Run `ARCH-2026-11-14-001` · Skill v3.2
> 证据等级：S3 静态代码 90 条、S2 文档 2 条；**S4/S5 = 0（未执行测试/服务）**

## 一句话定位

MemMachine 是一个“原始 Episode 为权威、declarative/event 两套索引可重建”的多存储 agent memory server；它最值得学习的不是某个后端，而是**写入/检索/删除三条链如何在无事务条件下保持可用**。

## 三条最值得带走的发现

1. **权威原文 + 可重建索引是二元契约**：EpisodeStorage 是权威；declarative 用 `DERIVED_FROM` 回填，event 用 segment→episode 水合。索引坏了会少结果，但不会返回错原文。→ KO-01
2. **多存储写入是无补偿的条件并行**：`episode_storage` 总是写，episodic/semantic 按 `target_memories` 追加；`gather(return_exceptions=True)` 只统一抛错，没有 write-intent。→ KO-02
3. **同一参数在两个后端不是一个单位**：`expand_context` 在 declarative 是 episode 邻域，在 event 是 segment 窗口；跨后端比较必须先固定单位。→ KO-04

## 产物地图

| 目录 | 内容 |
|---|---|
| `00_overview.md` | 总览、主要认知、阅读顺序 |
| `01-project-layer/` | 架构、数据模型与生命周期、配置/测试/治理 |
| `02-engineering-knowledge/` | EV-001~092、EK-01~052、EK Graph |
| `03-knowledge-layer/` | KO-01~08（L3 candidate / Observation） |
| `04-flow-atlas/` | 七类流 72 Edge |
| `05-candidates/` | C-01~12（含被推翻的 C-06） |
| `06-validation/` | 自检、独立盲重建、Reconciliation、覆盖缺口 |
| `archaeology-runs/ARCH-2026-11-14-001/` | Job Manifest、Snapshot、run_metadata |

## 验证结论

- 独立验证：**CONFIRMED 13 / PARTIALLY 3 / DOWNGRADED 1 / OVER_GENERALIZED 2 / CONTRADICTED 1 / MISSING 5 / NEEDS_HUMAN_REVIEW 2**。
- 关键修正：C-06 原判“PG 删除 feature 留下 citation 孤儿”被 `citations.feature_id ON DELETE CASCADE` 推翻；已降为条件性 Hypothesis。
- 未裁决：README 拓扑 vs 代码多后端；多 worker 运行时行为。
- **不宣称全绿**：没有运行测试/服务，所有 KO 都是 code-level Observation，不是 Validated Pattern。

## 边界

- 目标仓库、KnowlegeMap、生产 Skill 均只读；本目录只保存考古结果。
- 未写飞书（本轮未授权）。
- 每天最多 1 个深度项目；本 run 为 2026-11-14 的唯一项目。

# Job Manifest · 2026-11-08

## 1. KnowlegeMap 状态（只读，HEAD `5ce7ea7`）

| 项 | 状态 |
|---|---|
| 当前领域 | **Memory（Agent 记忆实现层 / 记忆治理）**，周起始 2026-11-09 |
| 上一领域 | Runtime，2026-11-08 已闭环换域（缺口①/②/③ 闭环，⑤ 部分，④ B 级挂账随 Memory 周携带） |
| 今日分工 | 11-08 周日：Runtime 第二周小结 + 换域判断（扫描日志不新增扫描） |
| Memory 周目标 | 能力树/基线（11-09）→ 增量（11-10~12）→ 候选验证（11-13）→ 回灌（11-14）→ 小结（11-15） |
| 已考古 corpus | `projects/` 50 个目录（含 mem0、letta-code、memgraphrag、mxc、dsh-memory-evolve） |
| 去重结论 | **无 `graphiti` 目录**；`grep -ri graphiti` 在 metadata/README 中无命中 → 未考古 |

## 2. 候选评估（Memory 领域，按 Knowledge Value 排序，不按 star）

| 候选 | 类型 | 排序理由 | 判定 |
|---|---|---|---|
| **Zep / Graphiti** | NEW | 时序知识图谱路线，与 corpus 已有 `mem0`（抽取+向量）构成**两种记忆架构的直接对照**；官方把"开源 framework vs 托管治理层"写成显式边界（README + SECURITY.md），可验证"记忆=数据层 vs 记忆=治理层"；APACHE-2.0、pushed 2026-11-06、v0.31.2 | **选中** |
| Mem0 | COMPLETED | `projects/mem0/` 已有；v3 单遍 ADD-only + 去图存储 | 排除（可作对照物） |
| Letta（框架本体） | QUEUED | 三级 Core/Recall/Archival 自编辑记忆；但 corpus 已有 `letta-code`（coding agent 形态），框架本体尚未单独考古，价值高但今日只选 1 个 | 顺延 |
| A-MEM | QUEUED | ~1K stars 研究向，证据强度低，Novelty 中 | 排除 |
| Supermemory / past.dev | QUEUED | 厂商技术报告为主，独立复测缺失，Benchmark 学习价值中 | 排除（B 级） |
| DyadMem / SEEM / Synapse | QUEUED | 论文，无工程仓库可考古 | 排除 |
| Memory-R1 / SimpleMem / MemU | QUEUED | 基准排名对象，非实现仓库 | 排除 |

> 11-08 为换域日，Memory 能力树 11-09 才建。选 Graphiti 作为 Memory 周**前置基石**：它提供了"记忆的数据模型（时序图 + episode provenance）"这一层，正好是能力树 A 层（记忆分层/写入策略）的实体样本。

## 3. Job Manifest

| 字段 | 值 |
|---|---|
| job_id | `ARCH-2026-11-08-001` |
| project | Zep Graphiti（`getzep/graphiti`） |
| repository | https://github.com/getzep/graphiti.git |
| mode | `initial` |
| priority | **A** |
| reason | ① Memory 领域首个实现层样本，补"时序图记忆"路线；② 与 `projects/mem0/` 形成 vector-extraction vs temporal-graph 对照；③ 官方 README/SECURITY.md 自带能力边界与负向清单，可检验"记忆即攻击面"（11-07 跨会话注入事件）；④ v0.31.2 在 2026-10 发布、2026-11-06 仍有 push，增长信号稳定 |
| source_entry | `03_expansion_queue/candidates/[cand]agent-memory-2026.md`（S 卡，Zep/Graphiti 行） |
| commit | `747b480ec6b12ecb718f1e2bc588b20ee36ea9c2`（main，2026-11-06 04:49:50 +0000） |
| version | `graphiti-core 0.31.2`（pyproject；最新 release v0.31.2 @ 2026-10-20） |
| expected_output | Package 00-06 + run_metadata.yaml + Independent Validation + Corpus PR |

## 4. 排除记录

- **Agent Substrate / OpenKruise Agents**：11-06 已完成（Runtime 收尾）。
- **go-sdk**：11-07 已完成（PR #67）。
- **durable 品类**：Runtime 两周已覆盖 9+2 向。
- **闭源/无仓库**（Genspark SB-Box、AWS AgentCore、Zep 托管平台）：无源码可考古，仅作对照引用。

# Daily Archaeology Job Manifest · 2026-11-09

**job_id**: `ARCH-2026-11-09-001`
**日期**: 2026-11-09（周一，Memory 周 Day1）
**每日配额**: 1 个深度项目（本轮用满）

---

## 1. KnowlegeMap 当前状态（只读，HEAD `f7b27d3`）

| 维度 | 状态 |
|---|---|
| 聚焦领域 | **Memory**（Agent 记忆实现层/记忆治理），周起始 2026-11-09 |
| 今日雷达动作 | 已建 `08_agent_centric/domains/agent-memory.md`：七层树（A 分层 / B 写入 / C 检索压缩 / D 生命周期 / E 作用域 / F 信任边界 / G 与执行状态分界） |
| 已知锚点 | Mem0、Zep-Graphiti、Letta、A-MEM、MemEval、平台内置记忆（Anthropic `/mnt/memory`、AgentCore Memory、MS AF Cosmos、Vercel eve、Grok Build）、文件/Git-native（OKF、Mem0×dsh）、学术新线（EnSIMem/SEEM/Synapse/Jev-Mem/Mem++/DyadMem/RealCompanion） |
| 缺口优先级 | **S**：基准口径核验、注入/污染防御；**A**：写入策略实证对照、作用域与租户隔离（含 TeamMind 缺口④）；**B**：生命周期（遗忘/冲突/版本化）、与执行状态共存 |
| 本周计划 | 周一建树（已完成）→ 周二~四增量 → **周五验证 Mem0/Graphiti/Letta 三路线** → 周六回灌 → 周日换域判断 |
| 换域记录 | 2026-11-08 Runtime → Memory ✅ |

## 2. 去重核验

- corpus 现有 48 个顶级项目目录，**无 `letta`**。
- 已有 `projects/letta-code/`（Letta Code，memory-first coding agent），与本次目标 `letta-ai/letta`（框架/平台本体）**不同仓库、不同范围**；仅在 metadata 中作为背景提及。
- `projects/mem0/`、`projects/graphiti/` 已存在，构成三路线中已考古的两条。

## 3. 候选排序（不按 star）

排序维度：知识价值 / Agent-AI 相关性 / 新颖度 / 增长信号 / 与已有 corpus 的连接 / 是否已考古 / Benchmark 学习价值。

| # | 候选 | 状态 | 对齐缺口 | 判定 |
|---|---|---|---|---|
| **1** | **letta-ai/letta（MemGPT）** | NEW | A5 分层映射、B3/B5 写入与门控、D1-D4 生命周期、E1-E4 作用域、G 执行状态分界 | **选中** |
| 2 | A-MEM（Zettelkasten） | NEW | D3 巩固与重组 | 研究向、~1k★、无工程实现深度；作为候选留待 |
| 3 | MCP memory server（文件路线） | NEW | C1/B2 | 体量小，属工具实现而非系统；价值低于 Letta |
| 4 | Supermemory / agents-memory | NEW | C1 | 厂商自报为主，无可核验源码证据链；排除 |
| 5 | 平台内置记忆（Anthropic / AgentCore / MS AF / Vercel / Grok） | NEW | E3/E4 | 闭源或无仓库，无法做代码级考古；排除 |
| 6 | Mem0 | COMPLETED | — | 已在 corpus，禁止重复 initial |
| 7 | Zep/Graphiti | COMPLETED（2026-11-08） | — | 昨日完成 |

## 4. 选中理由

1. **三路线闭合**：Mem0（抽取简化、去图）与 Graphiti（时序图）已考古，缺「自编辑三级记忆」这条路线；Letta 是唯一同时覆盖 **Core / Recall / Archival** 与 context 自管理的实现。
2. **直接补三个缺口**：
   - **D 生命周期 / 版本化**：Context Repositories 的 git-based 记忆版本化（D4），Mem++ 仅论文，Letta 有实现；
   - **E 作用域隔离**：memory blocks 的 agent/shared/read-only 归属模型 + 多 agent 共享，可承接 TeamMind 权限缺口④；
   - **G 执行状态分界**：Letta server 的 agent state / message buffer / archival 三层与"会话记忆"分层可直接对照。
3. **Novelty**：2026-03 转向 memory-first harness、2026-04 Context Constitution、2026-08 Agents SDK —— 架构在年内发生过方向性变化，适合做"设计意图 vs 实现"的分层取证。
4. **Benchmark 学习价值**：其 MemGPT 血统与 Graphiti 的时序图、Mem0 的抽取式形成三种写入策略的正交对照，可作为周五口径核验的源码侧基线。

## 5. 仓库事实（GitHub API，2026-11-09 查询）

| 项 | 值 |
|---|---|
| full_name | `letta-ai/letta` |
| 描述 | Platform for stateful agents: AI with advanced memory that can learn and self-improve over time |
| Stars / Forks | 25,388 / 2,668 |
| Size | 293,683 KB |
| Created / Pushed | 2023-10-11 / 2026-10-18T22:28:04Z |
| Default branch | `main` |
| License | Apache-2.0 |
| Latest release | `0.16.8`（2026-05-14） |

## 6. Job Manifest

```yaml
job_id: ARCH-2026-11-09-001
project: Letta (MemGPT)
repository: https://github.com/letta-ai/letta.git（**archive 分支**，非 main）
mode: initial
priority: A
reason: >
  Memory 周唯一缺失的第三条实现路线（自编辑三级记忆）；直接覆盖生命周期版本化、
  作用域隔离、执行状态分界三个 A/B 级缺口；与已考古 mem0 / graphiti 构成正交对照。
source_entry: 03_expansion_queue/candidates/[cand]agent-memory-2026.md（S 卡）
dq_slug: letta
branch: arch/letta-2026-11-09
```


---

## 7. 选型修正（阶段③开工前）

**事实**：`letta-ai/letta` 的 `main` 分支已退化为文档仓（仅 12 个文件：README / LICENSE / SECURITY / AGENTS / AI_POLICY 等）。README 明示源码迁至 `letta-ai/letta-code`，旧 V1 API server 保留在 **`archive` 分支**。

**修正**：考古目标改为 **`archive` 分支**（`56ba9c25552605eec89de8ed3dc6394b625c1993`，2026-08-13，1156 文件 / 878 `.py`），理由是它承载 Letta 的**服务端记忆实现**（memory blocks / archival / recall / block history / 多租户 ORM），与已考古的 `letta-code`（TS + MemFs 本地文件路线）构成**同产品两代架构对照**：
- letta-code：记忆 = 每 agent 一个 git 仓库 + Markdown 文件树（2026 现行）
- archive（V1 server）：记忆 = Postgres ORM + 三级记忆 + 服务端共享 block（2023–2025）

**边界声明**：archive 为 retired 代码，结论只描述"该实现曾经如何工作"，不代表当前 Letta 的运行时事实。此修正不改变 job_id / slug。

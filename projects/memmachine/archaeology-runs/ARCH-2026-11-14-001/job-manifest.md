# Job Manifest — ARCH-2026-11-14-001

- **date**: 2026-11-14 (Asia/Shanghai)
- **domain**: Memory（Agent 记忆实现层 / 记忆治理）
- **domain_day**: Day 6（回灌日；Day 7 才做周小结与换域）
- **daily_limit**: 1 个深度项目（本轮仅选 1 个）
- **engine**: knowledge-archaeology-skill v3.2
- **delivery**: 仅 push `knowledge-archaeology-corpus`（飞书落盘未获显式授权，不执行）

## 1. 输入状态（KnowlegeMap）

读取来源：`KnowlegeMap` `origin/main`（`6eb42275ec1cf89affbd14601bea548067615bda`），本地 HEAD 落后，故所有新提交内容用 `git show origin/main:<path>` 读取。

| 输入 | 关键事实 |
|---|---|
| `07_radar_watch/current_focus.md` | 当前领域 Memory（2026-11-09 起）；Day 6 回灌已完成；Day 7（11-15）才做周小结与换域 |
| `05_scan_logs/2026-11/2026-11-14_radar-watch_scan.md` | 明确“不追新资讯”，只回灌 8 项 |
| `03_expansion_queue/candidates/[cand]agent-memory-2026.md` | 状态 `[cand]`；11-10 增量记有 **B｜研究三则：MemMachine（arXiv 2604.04853，ground-truth-preserving：存原始 episode、少做 LLM 抽取、邻域上下文扩展检索——正对 C4 证据保留）** |
| `04_connections/README.md` | 11-14 新增 Mem0 OSS 本地可跑 ✅、EvolveLab 🟡；Zep CE 判定作废 |
| `08_agent_centric/domains/agent-memory.md` | Day 6 新增 H 层（H1 健康门 / H2 心跳 / H3 降级诚实 / H4 静默腐化防护 / H5 告警预算） |
| Corpus `projects/` | 49 个目录；无 `memmachine`；开放 PR 无 memmachine 分支 |

**本轮不承担换域判断**：current_focus 明确 Day 7（11-15）才做 Memory 周小结与 Computer Use / Browser 换域。

## 2. 候选池与状态

状态定义：`NEW`（未入 corpus 且无开放 PR）/ `QUEUED`（在雷达队列但本轮不选）/ `COMPLETED`（corpus 已有）/ `REFRESH_REQUIRED`（已有但上游发生重大变化）/ `IN_PROGRESS`（已有开放 PR）。

| # | 候选 | 仓库事实（2026-11-14 查询） | 状态 | 与已考古项目关系 |
|---|---|---|---|---|
| 1 | **MemMachine/MemMachine** | ★3,087；35,260 KB；Apache-2.0；main；created 2025-08-15；pushed 2026-11-12T14:10:08Z；描述“Universal memory layer for AI Agents…scalable, extensible, interoperable” | **NEW** | 与 `mem0`（单遍 ADDITIVE 抽取、无图）、`graphiti`（时序图）、`letta-code`（自编辑记忆）、`memgraphrag`（图记忆）形成写入策略对照 |
| 2 | supermemoryai/supermemory | ★31,609；214,995 KB；MIT；main；pushed 2026-11-13T16:03:52Z；描述“Memory and context engine + app…can be run fully locally” | NEW | 与 `mem0` 托管/本地口径对照；215MB 单仓含产品/应用层，体量大 |
| 3 | okf-memory/okf-agent-memory | ★866；1,131 KB；MIT；develop；pushed 2026-11-05T22:45:47Z；Git-native persistent memory，OKF v0.2，sub-300µs，BM25，Go | NEW | 与 KnowlegeMap 仓库模式同构；实现体量小，机制层可能偏薄 |
| 4 | doobidoo/mcp-memory-service | ★2,125；83,012 KB；Apache-2.0；main；pushed 2026-11-13T14:36:23Z | NEW（雷达 Day3 已“剔除”：v10.9 / -lite 小工具迭代） | 与 `mcp`、`hermes-agent` 的 MCP 记忆接口相邻；增量工具属性强 |
| 5 | tigerless-labs/agent-memory | ★4,316；120,727 KB；MIT；main；pushed 2026-11-12T08:13:46Z；plain Markdown as source of truth，local ranked retrieval | NEW | 与 OKF/KnowlegeMap 文件记忆同线；120MB 仓混合产品与运行时 |
| — | SIx Harness | 本轮无公开仓库（arXiv 2609.05510，N=1 运营记录） | QUEUED | 已在能力树作 A 级锚点；无代码不可做 Repository Snapshot |
| — | Hy-Memory / OpenViking / Graphiti / Letta / MemEvolve | 开放 PR #70 / #71 / #68 / #69 / #72 | IN_PROGRESS | 不重复 |
| — | Mem0 / MemGraphRAG / dsh-memory-evolve | corpus 已有 | COMPLETED | 作为对照基线 |

> 事实来源：GitHub REST `repos/{owner}/{repo}`（2026-11-14 查询）与 `api.github.com/search/repositories`；未按 star 排序。

## 3. 排序（按任务指定维度，非 star）

| 排序 | 候选 | Knowledge Value | Agent-AI 相关性 | Novelty | 增长/活跃 | 近期发布 | 与 Corpus 连接 | 是否已考古 | 是否需 refresh | Benchmark 学习价值 | 综合 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | MemMachine | A：保留原始 episode + 派生节点/边 + Profile 引用，直接对应“证据保留 vs 有损压缩” | A：通用记忆层（REST/Python/TS/MCP，8 框架集成） | A−：与 Mem0 单遍抽取路线正面对照；event/declarative 双后端 | A：15 个月 3,087★，11-12 仍有推送 | A：main 最新提交 11-02 | A：可直接对照 mem0/graphiti/letta-code/memgraphrag | 否 | 否 | A：多后端矩阵、写入/删除顺序、合并去重的失败面 | **#1** |
| 2 | supermemory | A−：LongMemEval 自报 SOTA + MCP 原生 | A | B+：多为厂商自报，机制待核实 | A：31.6k★，11-13 推送 | A | A：与 mem0 托管口径对照 | 否 | 否 | B+：产品/应用混杂，考古成本高 | #2 |
| 3 | OKF agent-memory | B+：Git-native + 文件即事实源 | A−：coding agent 记忆 | B+：与 KnowlegeMap 同构 | B：866★，11-05 推送 | B+ | A：直接连接 KnowlegeMap/GrowthOS | 否 | 否 | B：1.1MB，机制厚度有限 | #3 |
| 4 | mcp-memory-service | B：MCP 持久记忆服务 | A− | C+：工具迭代，Day3 已剔除 | A−：2.1k★，11-13 推送 | A− | B+：MCP 生态 | 否 | 否 | B−：增量工具，非架构级 | #4 |
| 5 | tigerless agent-memory | B：Markdown 事实源 | A− | B：与 OKF 同线 | A−：4.3k★ | A− | B+ | 否 | 否 | B−：120MB 混合仓 | #5 |

**排序结论**：选 **MemMachine**。不是因其 star 数（3,087 低于 supermemory 的 31,609），而是因为它是本周唯一同时满足“**新架构证据** + **可代码取证** + **与既有 Corpus 形成正面对照** + **直接命中 Memory 周 C4 证据保留批判线**”的候选。

## 4. 选中 Job

```yaml
job_id: ARCH-2026-11-14-001
project: MemMachine
repository: https://github.com/MemMachine/MemMachine.git
mode: initial
priority: A-
reason: >
  1) 架构层证据价值：README 宣称 Episodic Memory = graph、Profile Memory = SQL；
     代码同时存在 declarative（vector_graph_store）与 event（vector_store + segment_store）
     两条 LongTermMemory 后端，且 Semantic/Profile 层有独立 LLM ingestion + citation 溯源。
     这为“记忆写入是抽取还是保留原文”“写入失败是否产生不可达数据”“合并是否丢证据”
     提供了可直接从代码验证的样本。
  2) 与本周批判线对口：KnowlegeMap 2026-11-10 已记录 MemMachine 为
     "ground-truth-preserving：存原始 episode、少做 LLM 抽取、邻域上下文扩展检索——
     正对 C4 证据保留"；本轮把这条 B 级研究记录升级为代码级事实核验。
  3) 跨项目连接：可与 corpus 的 mem0（单遍 ADDITIVE、无图）、graphiti（时序图）、
     letta-code（自编辑记忆）、memgraphrag（图记忆）形成写入策略/证据保留矩阵。
  4) 非重复：corpus projects/ 无 memmachine，开放 PR 无 memmachine 分支。
  5) 活跃度：2026-11-12 仍有推送；main 最新提交 2026-11-02（377510d）。
source_entry:
  repo: Thy985/KnowlegeMap
  commit: 6eb42275ec1cf89affbd14601bea548067615bda
  path: 03_expansion_queue/candidates/[cand]agent-memory-2026.md
  line: 11-10 增量 B｜研究三则：MemMachine（arXiv 2604.04853）
expectation: >
  产出 00 Overview / 01 Project Layer / 02 Engineering Knowledge / 03 Knowledge Layer /
  04 Flow Atlas / 05 Candidates / 06 Validation & Evidence + run_metadata.yaml；
  再由独立 Auditor 盲重建验证后写入 corpus projects/memmachine/。
```

## 5. 边界与前置确认

- KnowlegeMap、skill-repo、corpus 均只读；corpus 仅允许新增 `projects/memmachine/`，不覆盖历史。
- 目标项目浅克隆到 `archaeology-jobs/ARCH-2026-11-14-001/repo`，只读，不修改。
- 飞书落盘未授权，不执行。
- 不创建/更新/删除任何定时任务。
- 若 Snapshot 发现仓库为镜像/空壳/核心代码不可读，回退 #2 supermemory 或 #3 OKF，并在本轮报告中说明。

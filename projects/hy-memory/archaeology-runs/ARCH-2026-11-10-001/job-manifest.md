# Job Manifest · 2026-11-10（Memory 周 Day3）

## 1. KnowlegeMap 当前状态（只读）

- HEAD `b96e6592`；`origin/main` 已同步
- `07_radar_watch/current_focus.md`：领域 **Memory**，周起始 2026-11-09；周二（11-10）已完成"只追增量"，周三~周四继续追增量，周五做候选实测
- `05_scan_logs/2026-11/2026-11-10_radar-watch_scan.md`：
  - 升 **A 级锚点：腾讯混元 Hy-Memory 2.0**（2026-10-16，模型/框架/应用三层全开源）
  - **A−：Zep Community Edition 开源**（2026-10-29）
  - B：MemMachine / MINDSET / Curation+Consolidation（学术线）、Mem0 spaCy 实体子集合
  - 新增基准批判第 4 条（LoCoMo 94% / LongMemEval 85% 题目只需 ≤2 会话）
- `08_agent_centric/domains/agent-memory.md`：七层树 A~G；缺口 6 项（S×2：基准口径、注入防御；A×2：写入策略、作用域隔离；B×2：生命周期、与执行状态分界）
- `03_expansion_queue/candidates/[cand]agent-memory-2026.md`：Hy-Memory 2.0 为 A 级锚点，含自报指标与"不可与 Mem0 横比"的口径警示

## 2. 去重检查

Corpus 现有 51 个顶级项目，其中与 Memory 相关：`mem0`、`graphiti`、`letta-code`、`letta`（PR #69 未合并，仅在分支）、`dsh-memory-evolve`、`agent-memory-guard`、`memgraphrag`、`mxc`、`dsh` 等。
**无 `hy-memory` / `hy_memory` / `memhub` 目录** → 本轮为 initial，不构成 refresh。

## 3. 候选排序（按知识价值，不按 star）

| 序 | 候选 | 状态 | 知识价值 | Agent-AI 相关性 | 新颖度 | 增长信号 | 与 Corpus 连接 | 判断 |
|---|---|---|---|---|---|---|---|---|
| 1 | **Tencent-Hunyuan/Hy-Memory** | NEW | **A** | 极高（记忆写入策略模型化） | 高（2026-10-15 创建） | 强（2,020★ / 146 fork / 40+ build 分支 / 3 releases） | 与 mem0（抽取式）、graphiti（时序图）、letta（三级记忆）正交 | **选中** |
| 2 | getzep/zep（Community Edition） | REFRESH_REQUIRED | A− | 高 | 中（10-29 开源） | 中 | 与 graphiti 同生态、同作者 | 备选；仓库定位偏 examples/integrations，工程深度待证 |
| 3 | shibing624/memhub | NEW | B | 中 | 中 | 弱（13★） | 与 Hy-Memory 的 MemHub 同名不同物 | 排除（体量小、非官方） |
| 4 | A-MEM（Zettelkasten 记忆） | QUEUED | B | 中 | 中 | 弱 | 研究向 | 排除（无工程实现深度） |
| 5 | MCP memory server | QUEUED | C | 中 | 低 | 弱 | 与 mcp 已有项目重叠 | 排除（体量过小） |
| 6 | mem0 | COMPLETED | — | — | — | — | 已考古 | 排除 |
| 7 | graphiti | COMPLETED | — | — | — | — | 已考古（2026-11-08） | 排除 |
| 8 | Letta archive | COMPLETED | — | — | — | — | 2026-11-09 完成（PR #69 open） | 排除（每日 1 项目） |

**选中理由**：Hy-Memory 是 Memory 周唯一把"写入策略"从提示词工程变成**可训练模型工件**的实现（30B-A3B 专训记忆读写，可替换 Mem0 的写入模型），同时提供"工作记忆 + 代码记忆"两类此前未被 corpus 覆盖的记忆语义，并以 MemHub 给出跨 10 个 AI 工具共享同一份个人记忆的 E 层（作用域/共享）样本。它直接覆盖能力树缺口 B（写入策略实证）与 E（作用域与共享），且与已考古的 mem0 / graphiti / letta 形成"抽取式 / 时序图 / 自编辑 / **模型化写入**"四点对照。

## 4. GitHub 事实（API 核实，2026-11-10）

| 项 | 值 |
|---|---|
| full_name | `Tencent-Hunyuan/Hy-Memory` |
| default_branch | `main` |
| HEAD | `9737431cf8965124b115f0e26cb0ff1e5cafa323` |
| HEAD 时间 | 2026-11-06 15:26:45 +0800 |
| HEAD 标题 | `chore(release): bump MemHub build number to 45 (#376)` |
| created_at | 2026-10-15T13:06:56Z |
| pushed_at | 2026-11-09T12:46:45Z |
| stars / forks / open issues | 2,020 / 146 / 153 |
| size | 23,605 KB（克隆后 68 MB，含 docs/测试/插件） |
| language | Python |
| license | `NOASSERTION`（LICENSE 文件 9,650 bytes，需在 Snapshot 中核对其实际条款） |
| latest releases | `v1.0.6`（2026-11-06，MemHub）、`hy-memory-v2.0.5`（2026-10-22） |

## 5. Job Manifest（YAML）

```yaml
job_id: ARCH-2026-11-10-001
project: Hy-Memory 2.0
repository: https://github.com/Tencent-Hunyuan/Hy-Memory.git
slug: hy-memory
mode: initial
priority: A
reason: >
  Memory 周 A 级锚点；把记忆写入策略模型化（30B-A3B 专训），
  并提供工作记忆/代码记忆两类新语义与 MemHub 跨工具共享样本；
  与 corpus 已有 mem0/graphiti/letta 四点正交，且无历史考古记录。
source_entry: 03_expansion_queue/candidates/[cand]agent-memory-2026.md
             + 05_scan_logs/2026-11/2026-11-10_radar-watch_scan.md
run_id: ARCH-2026-11-10-001
date: 2026-11-10
```

## 6. 边界声明

- 本轮只考古 `main` @ `9737431c`；不覆盖 40+ 条 `chore/*` build 分支。
- 厂商自报指标（LoCoMo 72.6% 等）**不进入 Fact 层**，只作为"文档声明"记录，并标注不可与 Mem0 横比。
- 不修改目标仓库 / KnowlegeMap / 生产 Skill。

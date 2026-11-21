# Job Manifest · 2026-11-11（Memory 周 Day3）

- **job_id**: `ARCH-2026-11-11-001`
- **selected**: `openviking`
- **mode**: `initial`
- **priority**: `B+`（雷达 B 级，按 Knowledge Value / 连接度提升为当日考古对象）
- **run dir**: `archaeology-jobs/ARCH-2026-11-11-001/`

## 1. 当前状态（KnowlegeMap @ `078e5f51`）

| 项 | 状态 |
|---|---|
| 当前领域 | **Memory**（周起始 2026-11-09，Day3） |
| 今日 scan log | `05_scan_logs/2026-11/2026-11-11_radar-watch_scan.md` |
| 今日增量 | A：**StreamMemBench**（基准批判第 5 条「存储≠会用」）；A−：**Zep v4 Context API + Agent Skills**；B：**MEMTIER**、**OpenViking** |
| 本周进度 | 周一能力树、周二 Hy-Memory / Zep CE 增量、**周三（今日）只追增量已完成**；周四 MCP 记忆生态；周五候选验证 |
| 已知锚点 | Mem0（抽取式）/ Zep-Graphiti（时序图）/ Letta（自编辑）/ **Hy-Memory 2.0（已考古 11-10）** |
| 缺口 | 基准口径、注入防御（S）、写入策略、作用域（A）；TeamMind 权限缺口④ |

## 2. 去重（Corpus 51 个顶级目录）

```
已考古且与本日候选相关：mem0 / graphiti / letta-code / dsh-memory-evolve /
                      agent-memory-guard / memgraphrag / hy-memory
openviking        → 不存在（NOT_EXIST）
zep (CE 服务层)    → 不存在（graphiti 只是图库，不是 Zep 服务）
```

## 3. 候选排序（不按 star）

| # | 候选 | 状态 | Knowledge Value | Agent-AI 相关性 | Novelty | 增长/发布 | Corpus 连接 | 是否已考古 | 结论 |
|---|---|---|---:|---:|---:|---:|---:|---|---|
| 1 | **OpenViking**（volcengine） | NEW | A | A | A | 41.9k★ / 11-10 push | 与 mem0/graphiti/letta/hy-memory 正交（上下文数据库） | 否 | **选中** |
| 2 | Zep CE | REFRESH_REQUIRED | A− | A | B | 10-29 开源 | graphiti 同源 | 否（但无源码仓库） | 排除：`getzep/zep` 仅 Examples/Integrations，CE 无源码可考古 |
| 3 | StreamMemBench | NEW | B | A | A（方法） | arXiv 2606.14571，24★ | 可作 benchmark 对手 | 否 | 排除：基准脚本，工程知识密度低 |
| 4 | MEMTIER | NEW | B | B | C | 论文 | 与 Tafcm 本地路线相关 | 否 | 排除：论文无仓库 |
| 5 | Zep v4 / Agent Skills | NEW | A− | A | A | 11-05 | graphiti | 部分（graphiti） | 与 #2 合并，留待源码可得 |
| 6 | mcp-memory-service v10.9 | NEW | C | B | D | 小版本迭代 | MCP 生态 | 否 | 排除：B 级以下 |
| 7 | me0（MCP memory） | NEW | C | B | C | 新 | MCP 生态 | 否 | 排除：C 级素材 |
| 8 | Hy-Memory | COMPLETED | A | A | — | 11-10 已考古 | 同域 | 是 | 排除（不重复） |

## 4. 选中理由

1. **Memory 周唯一"有完整源码且未考古"的候选**：今日 A/A− 增量（StreamMemBench、Zep v4、MEMTIER）均为论文/产品公告，Zep CE 无源码仓；OpenViking 是 6,608 文件、Python 733k 行 + Rust 130k 行的完整系统。
2. **品类正交**：corpus 已有 mem0（抽取式写入）、graphiti（时序图）、letta（自编辑）、hy-memory（写入策略模型化）；OpenViking 是**上下文数据库**——用 `viking://` 虚拟文件系统统一 Resource/Memory/Skill，L0/L1/L2 渐进加载，是"文件即记忆"路线的工业级实现。
3. **增长信号强**：41,948 stars（2026-11-11 查询），2026-11-10 仍在提交（`#7391`），1376 open issues；多语言 SDK / 30+ CI workflow。
4. **Benchmark 学习价值**：README 自报 LoCoMo 80–83%（vs 原生 24–57%）、tau2-bench +6.87/+11.87pp，可与 corpus 的基准批判第 1–5 条对照（厂商自报，禁入 Fact 层）。
5. **可连接缺口**：ACL/多租户（E 层）、隐私占位符与读时恢复（F 层）、FS 权威 vs 向量派生（G 层）——正对 Memory 周的能力树缺口。

## 5. GitHub 事实（2026-11-11 查询）

| 项 | 值 |
|---|---|
| repository | `https://github.com/volcengine/OpenViking.git` |
| branch | `main` |
| HEAD | `f7e83b2d739905b50215cd7739199a564b9b044b` |
| commit time | 2026-11-10 22:17:06 +0800 |
| commit msg | `chore(ui): prepare react 0.1.6 release (#7391)` |
| license | **AGPL-3.0** |
| stars / open issues | 41,948 / 1,376 |
| primary language | Python（26.5 MB）；TypeScript 6.1 MB；Rust 4.4 MB |
| repo size | 420,697 KB（GitHub API）；浅克隆 218 MB |
| package version | `openviking 0.3.22`（README benchmark 段） |
| requires-python | `>=3.10` |

```yaml
job_id: ARCH-2026-11-11-001
project: openviking
repository: https://github.com/volcengine/OpenViking.git
mode: initial
priority: B+
reason: >
  Memory 周 Day3 唯一完整开源且未考古的候选；上下文数据库品类与 corpus 中
  mem0/graphiti/letta/hy-memory 四点正交；41.9k stars、每日活跃；
  ACL/多租户/隐私/FS-索引一致性可直接回灌 Memory 能力树缺口。
source_entry:
  - 05_scan_logs/2026-11/2026-11-11_radar-watch_scan.md
  - 03_expansion_queue/candidates/[cand]agent-memory-2026.md
```

## 6. 边界

- **只读**目标仓库；不运行服务、不安装依赖、不调用模型。
- **scope**：Python 核心（`openviking/`）+ 文档 + Rust `crates/ragfs` 入口；**不覆盖** `frontend` / `web-studio` / `examples` / `bot` / `integrations` / `sdk` 全量。
- 仓库规模 733k Python 行 + 130k Rust 行，本轮为**核心机制抽样考古**，Coverage 将在 Package 中显式声明上限。
- AGPL-3.0 仅用于只读分析，不复制代码进 corpus。

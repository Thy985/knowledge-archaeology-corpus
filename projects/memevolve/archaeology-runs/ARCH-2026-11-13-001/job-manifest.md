# Job Manifest — ARCH-2026-11-13-001

## 1. 输入基线（KnowlegeMap, 只读）

| 项 | 值 | 来源 |
|---|---|---|
| KnowlegeMap HEAD | `fdd8ff1e22a893510a85b165629e27171f83f715` | `.runtime-probe/knowledgemap` `git log -1` |
| 当周领域 | Memory（周起 2026-11-09；本轮为 Day6 周五验证日） | `07_radar_watch/current_focus.md` |
| Day4 扫描 | SIx Harness《Memory as Infrastructure》A；**MemEvolve + EvolveLab A−**；OMEGA / mcp-memory-service / MemoryGraph / MemoryMesh B/C | `05_scan_logs/2026-11/2026-11-12_radar-watch_scan.md` |
| Day5 扫描 | Mem0 OSS 2.2.1 本地 `add→get_all` 2 条、`search` top 0.837；EvolveLab 源码核对 **13 providers = 12 基线 + Evolver**，统一 `provide_memory` / `take_in_memory`；Zep CE 开源判定作废 | `05_scan_logs/2026-11/2026-11-13_radar-watch_scan.md` |
| Corpus 基线 | `origin/main = 9d27bc5d9b938df06127c4decd423c2f909136f2`，51 个顶级项目，无 `memevolve` / `evolvelab` | `.runtime-probe/knowledge-archaeology-corpus` |

## 2. 候选排序（不按 star）

排序依据：Knowledge Value → Agent-AI Engineering 相关性 → Novelty → 增长信号 → 近期发布 → 与已有 Corpus 连接 → 是否已考古 → 是否需 refresh → Benchmark 学习价值。

| 排名 | 候选 | 状态 | GitHub 事实 | 判断 |
|---|---|---|---|---|
| **1** | **bingreeky/MemEvolve（含 EvolveLab）** | **NEW** | 286★；83,966 KB；Apache-2.0；main；pushed_at `2026-05-05T09:29:28Z`；HEAD `6035d5659d7a092dbfa6a87b1a32a3cee652ba54` | **选中**。填补 Corpus 第 6 个记忆正交点：记忆架构的**元进化**（前五点：mem0 抽取式 / graphiti 时序图 / letta 自编辑 / hy-memory 写入策略 / openviking 文件系统即记忆）；13 套记忆系统统一接口，是跨项目对照的工程底座；直接承接 Memory 周 Day4/Day5 增量。 |
| 2 | doobidoo/mcp-memory-service | NEW | 2,116★；82,891 KB；pushed_at `2026-11-12` | 更新更热、Agent 工具化更直接；但 novelty 偏“MCP 记忆服务”常规形态，与已有 mem0/openviking 正交点重叠度高。若 Memory 周继续，可作次日候选。 |
| 3 | OMEGA | NEW | Day4 B 级 | 证据等级不足，未进入本轮对照。 |
| 4 | MemoryGraph / MemoryMesh | NEW | Day4 C 级 | 规模与可运行性未核，暂缓。 |

> 说明：MemEvolve 自 2026-05 后未更新，热度低于 mcp-memory-service；按 SOP “不按 star 排名”，其 **novelty（记忆系统的自动生成与锦标赛选择）+ benchmark 学习价值（13 provider 统一接口）** 优先。

## 3. 选中 Job

```yaml
job_id: ARCH-2026-11-13-001
project: MemEvolve
repository: https://github.com/bingreeky/MemEvolve.git
branch: main
commit: 6035d5659d7a092dbfa6a87b1a32a3cee652ba54
commit_time: 2026-05-05T17:29:28+08:00
license: Apache-2.0
stars: 286
size_kb: 83966
mode: initial
priority: A-
selected_at: 2026-11-13
source_entry: KnowlegeMap 05_scan_logs/2026-11/2026-11-12_radar-watch_scan.md (A−) + 2026-11-13_radar-watch_scan.md (源码核对)
```

### priority 依据（A−）
- **A 级要素**：把“记忆系统设计”本身变成搜索空间（analyze → generate → create → validate → tournament），在 Corpus 中无同类正交点；13 providers 提供横向对照底座。
- **扣分项（−）**：仓库 2026-05 后未更新；论文自报成绩（+17.06% 等）只能作 S0-vendor，未复跑；无 EvolveLab/MemEvolve 自有测试（仅有 vendored `mini-swe-agent/tests`）。

## 4. 边界声明

- 目标仓库只读；KnowlegeMap 只读；生产 Skill v3.2 只读（本轮不触发 Skill Evolution）。
- 本轮仅 1 个深度项目；不覆盖 Corpus 历史。
- 飞书落盘未授权 → 不做；只 push Corpus。
- 论文自报数字不进入 Fact / KO；证据上限 S3（静态代码 + 配置），无 S4/S5 运行证据。

# Job Manifest — ARCH-2026-11-15-001

- job_id: `ARCH-2026-11-15-001`
- run_date: `2026-11-15` (Asia/Shanghai)
- project: **Graphiti** (`getzep/graphiti`)
- repository: `https://github.com/getzep/graphiti.git`
- slug: `graphiti`
- mode: `initial`
- priority: **A−**
- source_entry:
  - `03_expansion_queue/candidates/[cand]agent-memory-2026.md` → Zep / Graphiti 行
  - KnowlegeMap Day7 第二周计划 → 11-17~19「Graphiti×Mem0 对照」
- domain: Memory（2026-11-15 Memory 周 Day7 已判定延长第二周，本轮不承担换域判断）

## 1. Selection Rule

排序维度（不使用 GitHub star 作为排序输入）：
1. Knowledge Value：是否能直接回答 Memory 领域的信任、溯源、作用域与失效问题。
2. Agent-AI Engineering 相关性：是否属于 Agent Memory / 知识图 / 运行时基础设施。
3. Novelty：相对已有 corpus 项目是否提供新的机制结构。
4. 增长信号：近期是否有真实提交与发布活动。
5. 近期发布：版本与默认分支是否活跃。
6. 与已有 Corpus 连接：能否与 `mem0`、`memmachine`、`letta-code` 等形成正面对照。
7. 是否已考古：Corpus `projects/` 49 个目录已核对，无 `graphiti`。
8. 是否需要 refresh：无历史 Graphiti 结果，属于 NEW。
9. Benchmark 学习价值：是否有足够测试与边界，可用于检验 Archaeology Skill。

## 2. Candidate Ranking

| Rank | Candidate | Status | Knowledge Value | Agent-AI relevance | Novelty | Growth signal | Corpus relation | Decision |
|---|---|---|---|---|---|---|---|---|
| 1 | **Graphiti (`getzep/graphiti`)** | NEW | 高：双时间事实图 + 软失效 + 溯源 | 高：Agent memory infrastructure | 高：显式 `valid_at` / `invalid_at` / `expired_at` | 强：`2026-11-14` 仍有推送 | 可与 `mem0`/`memmachine`/`letta-code` 对照 | **SELECTED** |
| 2 | Hy-Memory 2.0 | QUEUED | 中高：三层记忆 | 高 | 中：核心为 30B-A3B 模型 + SDK，仓库外自报多 | 待核验 | 与 Memory 周相关 | Deferred：代码可取证性弱于 Graphiti |
| 3 | SIx Harness《Memory as Infrastructure》 | QUEUED | 高：可靠性/健康门 | 高 | 中：偏运营记录 | 待核验 | 与 H 层相关 | Deferred：非本轮代码考古首选 |
| 4 | MemEvolve / EvolveLab | QUEUED | 中高：结构级对照 | 高 | 中 | 中 | 可作 benchmark | Deferred：需 API keys 与外部数据集，行为验证成本高 |
| 5 | mcp-memory-service / OKF / agentmemory 等 | QUEUED / 雷达 B 级 | 中低 | 中 | 低 | 低 | 局部 | Not selected |

> GitHub stars、fork 数等仅记录为仓库元数据，不参与上述排序。

## 3. Why Graphiti

1. **直接补 Memory 周的 F 层信任边界**：事实不是被删除，而是写入 `invalid_at` / `expired_at`；可直接检验“旧事实是否仍会被检索并交给 LLM”。
2. **直接补 E 层作用域**：`group_id` 是分区键并参与 driver 路由；但 reference server 明确声明无认证/授权、无调用方隔离。作用域与授权是两件事，这是本轮的核心对照。
3. **有真实代码可取证**：`graphiti_core/graphiti.py`、`utils/maintenance/edge_operations.py`、`utils/bulk_utils.py`、`search/`、`server/`、`mcp_server/` 均有具体实现。
4. **有测试线索**：81 个 `test_*.py`、910 个静态 `def test_`；时间语义有专门测试文件，但本轮未执行测试，S4 不冒充。
5. **可与已有 Corpus 正面对照**：Mem0 偏记忆抽取与更新，MemMachine 偏分层记忆，Graphiti 偏“事实图 + 时间有效性 + 证据边”。

## 4. Scope

- 目标：产出完整 Project Archaeology Package（00–06 + `run_metadata.yaml`）。
- 只读：目标仓库、KnowlegeMap、生产 Skill。
- 可写：本 run 目录；阶段⑥仅写 `knowledge-archaeology-corpus` 的 `projects/graphiti/`。
- 不做：飞书落盘（未授权）；不运行目标仓库服务；不修改目标仓库；不创建新定时任务。

## 5. Known Boundaries at Selection Time

- Graphiti 的 reference REST/MCP server 在 `SECURITY.md` 中自述：无认证、无授权、无调用方隔离、无 prompt-injection 防护。
- `store_raw_episode_content=True` 是默认；设为 `False` 会清空 episode `content`，原始证据不可回读。
- 双时间失效依赖 LLM 判断 + 代码时间窗口规则；两者都不是数据库级约束。

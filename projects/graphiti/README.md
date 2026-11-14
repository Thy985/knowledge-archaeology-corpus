# Graphiti · Knowledge Archaeology

**一句话定位**：Graphiti 是一个双时间事实图记忆系统：把事实的“现实有效区间”与系统的“记录/失效时间”分开保存，用软失效保留审计轨迹；但当前真假的最终裁决仍依赖 LLM 判断、检索调用方与外部认证层。

## 考古基本信息

| 项 | 值 |
|---|---|
| Repository | `https://github.com/getzep/graphiti.git` |
| Commit | `ed23196506eba8806fae6da831d73fcd2ad714d8` (`main`, 2026-11-14) |
| Version | `graphiti-core 0.31.2` |
| License | Apache-2.0 |
| Skill | `knowledge-archaeology v3.2` |
| Run | `ARCH-2026-11-15-001`（initial，priority A−） |
| Runtime verification | 无（S4=0 / S5=0） |

## 产物索引

| 路径 | 内容 |
|---|---|
| `00_overview.md` | 中心问题、四时间轴、一句话结论、验证摘要 |
| `01-project-layer/` | 项目地图：架构、模块、数据结构、生命周期、配置、依赖、测试 |
| `02-engineering-knowledge/` | EV-001..EV-114 + EK-01..EK-57（含 EK-41b，共 58 条）+ EK Graph |
| `03-knowledge-layer/` | KO-01..KO-08，L1→L5 + R1–R4 聚合规则 + 反例 |
| `04-flow-atlas/` | 七类流，89 条可回溯 Edge |
| `05-candidates/` | C-01..C-20 / XP-01..XP-05 / HR-01..HR-07 |
| `06-validation/` | Self / Independent / Coverage Gaps / Reconciliation |
| `archaeology-runs/ARCH-2026-11-15-001/` | 本次 run 的完整快照与元数据 |

## 核心发现（摘要）

1. **四时间轴**：`created_at` / `valid_at` / `invalid_at` / `expired_at`；旧边不删除，只写失效时间。
2. **模型提议，时间算术执行**：LLM 只判断重复/矛盾/无关；`_apply_temporal_invalidation` 与 `resolve_edge_contradictions` 决定失效边界。
3. **默认检索不保证“只返回当前事实”**：`SearchFilters()` 默认空；上下文只标注区间，不替调用方裁决。
4. **`group_id` 不是授权边界**：reference server 无认证/授权/隔离/注入防护；`SECURITY.md` 明确外推给部署方。
5. **请求级隔离是局部的**：只有 `add_episode` / `add_episode_bulk` 做 driver clone；其余公开 API 仍用默认 driver。
6. **证据锚存在悬空引用**：`add_triplet` 无真实 episode 时写入未持久化 synthetic UUID，字段非空但不可解析。
7. **驱动事务能力不一致**：Neo4j 有真实事务；FalkorDB / Kuzu / Neptune 的 `execute_write` 无事务包装。

## 验证结论

- Self-validation：`PASS WITH LIMITATIONS`
- Independent validation：`PARTIALLY_CONFIRMED`
  - 32 confirmed / 6 partially / 1 downgraded / 1 over-generalized / 5 missing / 3 contradicted
- 关键修正：悬空 UUID、请求级 scope 只覆盖两个 ingestion API、`remove_episode`/`get_nodes_and_edges_by_episode`/默认 `search` 未按 group 路由数据库。

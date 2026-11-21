# 06.1 · Self-Validation — Six Dimensions

> 这是 **Synthesizer 自检**，不是 Independent Auditor 的盲重建。它不能替代阶段⑤。

## A. Truth Audit（抽样）

抽样 20 条关键断言，逐条回到 `ed23196` 复核。

| # | Claim | Verdict | Note |
|---|---|---|---|
| 1 | 主写入在一个 `execute_write` 内 | CONFIRMED | `graphiti.py:1037`; `bulk_utils.py:128` |
| 2 | “原子写”对所有驱动成立 | **CORRECTED** | 仅 Neo4j 有真实事务；FalkorDB/Kuzu/Neptune 直接执行 |
| 3 | 默认检索过滤 `invalid_at` | **CORRECTED** | `SearchFilters` 默认 None；不过滤 |
| 4 | `add_triplet` 绕过去重/失效 | **CORRECTED** | 绕过抽取，但仍走 `resolve_extracted_edge` |
| 5 | 删除 episode 会恢复旧事实 | **CORRECTED** | 无恢复路径 |
| 6 | `store_raw=False` 清空 `source_description` | **CORRECTED** | 只清空 `content` |
| 7 | `group_id` 是认证边界 | **CORRECTED** | 只是分区/路由键 |
| 8 | `created_at` 是事实发生时间 | **CORRECTED** | `created_at` 是入库时间；`valid_at` 才是事实时间 |
| 9 | 双时间 = 两个时间字段 | **CORRECTED** | 实际四条轴：`created_at`/`valid_at`/`invalid_at`/`expired_at` |
| 10 | 同一 `valid_at` 会有一方失效 | **CORRECTED (logic)** | 两个跳过条件均成立，无失效 |
| 11 | 上下文包含 edge/episode UUID | **CORRECTED** | 只含 fact + dates + source_description/content |
| 12 | MCP 队列持久化 | **CORRECTED** | `asyncio.Queue` 进程内 |
| 13 | 前序窗口是 3 条 | **CORRECTED** | `add_episode` 用 `RELEVANT_SCHEMA_LEN=10`；`EPISODE_WINDOW_LEN=3` 是另一默认 |
| 14 | `max_coroutines` 是全局并发上限 | **CORRECTED** | 每个 `semaphore_gather` 调用点各建一个 Semaphore |
| 15 | 社区重建是原子的 | CONFIRMED FALSE | 先删后建，无事务 |
| 16 | bulk 两阶段可能留下孤儿 episode | CONFIRMED (static) | phase-1 先写 episodes |
| 17 | 重复边复用 UUID 并追加 provenance | CONFIRMED | `edge_operations.py:930-1030` |
| 18 | `remove_episode` 删除仅由该 episode 提及的节点 | CONFIRMED | `episode_count == 1` |
| 19 | 时间失效是软失效，不删边 | CONFIRMED | 只写 `invalid_at`/`expired_at` |
| 20 | reference server 无认证/无授权 | CONFIRMED | 代码无中间件 + `SECURITY.md` |

**Truth summary**：20 sampled claims → 8 confirmed, 11 corrected, 1 confirmed-false. No unresolvable claim.

## B. Coverage Audit（子系统覆盖）

| Subsystem | Covered? | Main EK/KO | Gap |
|---|---|---|---|
| Facade / lifecycle | 是 | EK-17/19/24/50; KO-01/05/06 | 未跑运行时 |
| Node/edge extraction | 是 | EK-13/14/15 | prompt 细节未逐条审计 |
| Temporal invalidation | 是 | EK-01–12; KO-01/02 | 未做时间相等/时区实验 |
| Search | 是 | EK-36–41; KO-03 | 未跑 top-k 排序 |
| Drivers/persistence | 是 | EK-18/19; KO-06 | 未跑非 Neo4j 后端 |
| REST server | 是 | EK-45–48; KO-04 | 未跑认证绕过实验 |
| MCP server | 是 | EK-24–27; KO-05 | 未跑重启实验 |
| Communities | 是 | EK-22; C-08 | 未跑失败注入 |
| Saga | 是 | EK-21/50 | 未跑多 episode 链 |
| Tests | 是 | EK-52/53/56 | **未执行** |
| Migrations | 部分 | — | 快照仅含 `__init__.py`，无迁移内容 |
| Zep hosted service | 否 | — | 仓库外，不在范围 |
| UI / clients | 否 | — | 仓库无 UI |
| Full git history / ADR | 否 | — | 浅克隆，无法做历史决策考古 |

**Coverage gaps to carry into Independent Validation**
1. 无运行时验证（S4/S5=0）。
2. 无跨驱动一致性验证。
3. 无 MCP 进程重启/故障注入。
4. 无 top-k 失效事实实验。
5. 无并发同 group 写入实验。
6. 无历史 commit/ADR 考古（浅克隆）。

## C. Flow Audit

抽样 12 条 flow edges 回读：
- F-C-12 / F-D-08：`add_nodes_and_edges_bulk` 确实 `execute_write`，但原子性按驱动修正（CONFIRMED with correction）。
- F-S-09 / F-S-10：两个分支均存在；一个是旧边失效，一个是新边被回填为历史（CONFIRMED）。
- F-S-11：同 `valid_at` 无失效（logic CONFIRMED, runtime unverified）。
- F-E-05 / F-E-06：context 无 UUID（CONFIRMED）。
- F-A-04 / F-A-06：无认证中间件（CONFIRMED）。
- F-M-01 / F-M-02：10 条 + `valid_at <= reference_time`（CONFIRMED）。
- F-P-03 / F-P-04：prompt 约束存在（CONFIRMED）。
- F-C-18：`build_communities` 先删后建（CONFIRMED）。
- F-S-16：队列 `task_done` 与 log-only failure（CONFIRMED）。

**Flow coverage**：7/7 flow types present; 88 edges; all edges list symbol/file/condition/EV.

## D. Abstraction Audit

| Candidate abstraction | Initial level | Verdict | Reason |
|---|---|---|---|
| “Graphiti 保证当前事实” | L4 | **DOWNGRADED to L3/L2** | 默认检索不过滤；保证不成立 |
| “group_id 提供租户隔离” | L4 | **DOWNGRADED to L2 boundary** | 无认证/授权 |
| “Graphiti 是双时间” | L3 | **REFRAMED** | 四时间轴；双时间只是其中一对 |
| “LLM 决定事实真伪” | L4 | **DOWNGRADED to L3** | LLM 只提名，代码执行 |
| “事务抽象保证原子” | L4 | **DOWNGRADED to storage-specific L2** | 驱动差异 |
| “软失效 = 可审计” | L4 | **REFINED** | 可审计仅在源内容保留且调用方读 audit 视图时成立 |

**No L5 Principle was promoted.** All L4 models are `Cross-project validation pending`.

## E. Counterexample Hunt（至少 3 个/每个 L3+）

| Target | Counterexample | Result |
|---|---|---|
| P-01 Temporal intervals | 同一 `valid_at` 的互斥事实都不失效 | 区间模型不能打破平局 → 需 tie-break（C-01） |
| P-03 Model proposes/code disposes | 模型漏判矛盾 → 旧事实仍当前 | 提名权是单点故障（C-12） |
| P-05 Read-time projection | 默认读是 audit，top-k 可含失效事实 | 读模式默认值决定正确性（C-02） |
| P-06 Partition ≠ auth | 部署方加网关认证 | 结论仍成立，但风险依赖部署（F-A-12） |
| P-07 Durable outbox | 直接 `await add_episode` 无异步队列 | 结论限定于 MCP 路径 |
| P-08 Capability matrix | Neo4j 上确实原子 | 结论按后端限定 |
| P-09 Evidence anchor | `add_triplet` 传入真实 episode | 证据链可成立；结论需“强制或显式标记” |
| P-10 Request scope | 默认 DB 路径复用共享 driver | 无写入时不构成竞态；结论限定于可变状态 |

**Counterexample budget**: 8 targets, 8 counterexamples, 0 “no counterexample” claims. No target claims universal law.

## F. Epistemic Audit

| Check | Result |
|---|---|
| Fact vs Observation separated | 是；EK 表有 Epistemic 列 |
| Hypothesis kept out of KO | 是；15 条进 `05_candidates.md` |
| Cross-project candidate not written as Principle | 是；5 条均标 `Cross-project Candidate` |
| S4/S5 count | **0 / 0** |
| “Validated” usage | 未使用；仅 `Fact`/`Observation`/`Hypothesis` |
| Law/Principle usage | 未使用 |
| Confidence inflation | 自查发现 11 处，均已在 Truth Audit 修正 |

## G. Self-Validation Verdict

- **Truth**: 抽样 20，0 unresolvable，11 corrections applied.
- **Coverage**: 10/12 subsystems covered; 2 out of scope; 6 runtime gaps.
- **Flow**: 7/7 types, 88 edges, 12 sampled.
- **Abstraction**: 0 L5 promoted; 4 downgrades; 2 reframes.
- **Counterexample**: 8/8 targets attacked.
- **Epistemic**: S4=0, S5=0, 0 Principle, 15 hypotheses isolated.

**Status**: `PASS WITH LIMITATIONS` — not `ALL GREEN`. The package is structurally complete but runtime-unverified.

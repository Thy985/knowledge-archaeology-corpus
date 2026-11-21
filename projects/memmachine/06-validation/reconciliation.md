# Reconciliation — 独立验证后的合并决定

> 本文件记录“独立 Auditor 判定 → 是否接受 → 如何改 Corpus”。原 `00~05` 不因审计被静默改写；所有修正在 Corpus 落盘时显式登记。

| 审计项 | 判定 | 决定 | Corpus 处理 |
|---|---|---|---|
| C-06 PG citation 孤儿 | CONTRADICTED | **撤回强缺陷** | 改为条件性假设：只有后端无 FK CASCADE 时才成立；记录 `citation_association_table.feature_id ON DELETE CASCADE` |
| EK-14 三路并行 | OVER_GENERALIZED | **接受收窄** | 改为“1~3 路条件并行（episode_storage 总是；episodic/semantic 按 target_memories）” |
| EK-15 三路 gather | OVER_GENERALIZED | **接受收窄** | 同步改为“对已组装的任务集 gather” |
| EK-20 任何后段失败 | OVER_GENERALIZED | **接受收窄** | 改为“`add_segments` 成功而 vector upsert 失败时留下不可检索 segment；更早失败不留后段产物” |
| EK-45 已摘要旧条目 | PARTIALLY_CONFIRMED | **降为 Observation** | 改为“按 `_current_episode_count` 弹出旧条目；该计数在 summarize 前重置，不能等同于显式已摘要标记” |
| KO-05 身份不是信任边界 | DOWNGRADED | **接受 scope 收窄** | 限定“REST/MCP 直连且无前置认证网关”；网关部署不成立 |
| M-01 FK CASCADE | MISSING | **补证** | EV-091 改写；C-06 降级 |
| M-02 drop_session_partition 后实例不可复用 | MISSING | **补为 EK-52** | 加入 EK-52，links 到 EK-31/EK-46/KO-08 |
| M-03 delete_history 清 citation | MISSING | **补 Flow Edge** | 04 Flow 增加 F-E-07（delete_history→citation） |
| M-04 测试路径错误 | MISSING | **修正路径** | EV-077 路径改为 `packages/server/server_tests` |
| M-05 top_k 500 证据 | MISSING | **补 EK-43** | 补 `ge=1` 与 Qdrant 500 注释 |
| README 拓扑 vs 代码 | NEEDS_HUMAN_REVIEW | **不裁决** | 保留矛盾；标注“文档声称 vs 代码事实” |
| multi-worker | NEEDS_HUMAN_REVIEW | **不裁决** | 保留 G-06；不写运行时结论 |

## 质量指标是否变化

- EK 51 → **52**（新增 EK-52）；需要同步 EK Graph 指标（52 节点，新增 3~4 条边）。
- Evidence 91 → **92**（新增 EV-092：drop_session_partition / EK-52 证据）；S3 90、S2 2。
- KO 8 不变；KO-05 的 scope 收紧，不改变 L3/Observation 状态。
- Flow 70 → **72**（新增 F-E-07、F-S-15）。
- C-06 从 Strong candidate defect 降为 Hypothesis（条件性）。
- S4/S5 仍为 0；不因 Reconciliation 变绿。

## 未接受的修正

- 不把 C-01 升级为“默认必现”：默认 COSINE，条件为 EUCLIDEAN + 无 reranker。
- 不把 KO-07 的 “citation 过度归属” 改为 “citation 丢失”：PG 级联会删，Neo4j 属性会随节点删；只有 consolidation 把所有 citation 挂到每个新 feature 的“归属膨胀”成立。
- 不把任何 KO 升为 L4/L5：没有跨项目/运行证据。

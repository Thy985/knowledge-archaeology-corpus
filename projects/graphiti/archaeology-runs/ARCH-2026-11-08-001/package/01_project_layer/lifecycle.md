# 01.2 Lifecycle

## A. add_episode（单条写入）

`graphiti.py:1334-1588`，7 个阶段：

| # | 阶段 | 动作 | 证据 |
|---|---|---|---|
| 1 | Scope | 解析 `group_id` → 请求级 driver/clients（不污染 `self.driver`） | `graphiti.py:1304-1330` |
| 2 | Context | 取最近 `RELEVANT_SCHEMA_LIMIT` 个 previous episodes（或显式 UUID 列表） | `graphiti.py:1450-1458` |
| 3 | Episode | 构造 `EpisodicNode`，`created_at=now`、`valid_at=reference_time` | `graphiti.py:1458-1475` |
| 4 | Extract Nodes | LLM 抽取实体（可带 entity_types / 排除类型 / 自定义指令） | `graphiti.py:1477-1484` |
| 5 | Resolve Nodes | 三层去重梯度（identity → exact → semantic → LLM） | `node_operations.py:816-913` |
| 6 | Extract + Resolve Edges | LLM 抽边 → 端点对过滤 → 去重/矛盾裁决 → 时间失效（与属性抽取并行） | `graphiti.py:1495-1530`、`edge_operations.py:134-786` |
| 7 | Persist | hydrate 节点（summary/attributes）→ 写 episode/边 → 可选 saga 关联 → 可选 community 更新 | `graphiti.py:1528-1550` |

**顺序性要求**：文档建议 `add_episode` 放后台队列**串行**执行，等待上一条完成（`graphiti.py:1418-1425` docstring）。REST 实现用 `async_worker.queue` 串行消费并立即返回 202（`server/graph_service/routers/ingest.py:51-72`）。

## B. add_episode_bulk

`graphiti.py:1588`。批量路径先批量抽取/去重再写，减少 LLM 往返；`strict_edge_types` 默认 False（`graphiti.py:1596`）。

## C. 删除（remove_episode）

`graphiti.py:2174-2200`：
1. 找到 episode 及其 `entity_edges`；
2. 对每条边从 `episodes[]` 移除该 episode：仍有其他 episode 支持 → **更新**；无 → **删除**；
3. 对节点统计 `MENTIONS` 的 episode 数，仅被该 episode 提及 → 删除。
→ **引用计数语义**，不是级联删除。

`clear_data(driver, group_ids)`（`utils/maintenance/graph_data_operations.py:34`）按 group 清空。

## D. Community 生命周期

- `build_communities`（`graphiti.py:1867` / `community_operations.py:256`）：聚类实体 → 生成 CommunityNode + CommunityEdge；
- `update_community`（`community_operations.py:397`）：新实体增量并入已有社区；
- `add_episode(update_communities=True)` 时逐节点并发更新（`graphiti.py:1532-1545`）。

## E. Saga 生命周期

- `_get_or_create_saga`（`graphiti.py:487`）：按 `group_id + name` 取或建；
- `summarize_saga`（`graphiti.py:590`）：对一段 episode 做增量总结，记录 `last_summarized_at` / `last_summarized_episode_valid_at`；
- 关系：`HAS_EPISODE`（saga→episode）+ `NEXT_EPISODE`（episode 链）（`edges.py:694/827`）。

## F. 过期与失效

- **写入时**：`_apply_temporal_invalidation` 比较新旧边的时间边界，旧边置 `invalid_at` + `expired_at=now`（`edge_operations.py:715-786`）；
- **读取时**：`SearchFilters` 可按时间/属性过滤，默认检索"当前为真"的事实；
- **不物理删除**：历史保留，可做时间点查询（README）。

## G. 服务生命周期

- REST：`AsyncWorker` 队列 + FastAPI lifespan（`router = APIRouter(lifespan=lifespan)`，`ingest.py:47-49`）；
- MCP：全局 `graphiti_service`，`initialize_server()` 启动时建客户端（`mcp_server.py:1184`）；
- 库：`await graphiti.close()` 释放 driver（`graphiti.py:455`）。

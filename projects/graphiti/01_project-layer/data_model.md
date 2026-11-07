# 01.3 Data Model

## 节点

### Node（ABC）`nodes.py:93`
```python
uuid: str
name: str
group_id: str          # 图分区
labels: list[str]
created_at: datetime
```

### EpisodicNode `nodes.py:318`
| 字段 | 语义 |
|---|---|
| `source` | `EpisodeType`（message/text/json） |
| `source_description` | 来源描述 |
| `content` | **原始内容**（不被改写，provenance 根） |
| `valid_at` | 事件真实发生时间（= `reference_time`） |
| `entity_edges: list[str]` | 该 episode 产生的边 UUID |
| `episode_metadata` | 附加元数据 |

### EntityNode `nodes.py:499`
`name_embedding`（用于去重/检索）、`summary`（"surrounding edges 的区域摘要"，随时间演化）、`attributes`（结构化属性，受 entity_types 约束）。

### CommunityNode `nodes.py:727`
`name_embedding`、`summary`（成员节点的区域摘要）。

### SagaNode `nodes.py:907`
`summary`、`first_episode_uuid`、`last_episode_uuid`、`last_summarized_at`、`last_summarized_episode_valid_at`；私有计数 `_summary_episodes_fetched/selected/skipped`、`_summary_selected_episode_bytes`。

## 边

### Edge（ABC）`edges.py:49`
`uuid`、`group_id`、`source_node_uuid`、`target_node_uuid`、`created_at`。

### EntityEdge（事实）`edges.py:263`
| 字段 | 语义 | 时间维度 |
|---|---|---|
| `name` | 关系名（受 edge_types 签名约束） | — |
| `fact` | 自然语言事实 | — |
| `fact_embedding` | 事实向量 | — |
| `episodes: list[str]` | **支持该事实的 episode UUID 列表** | provenance |
| `valid_at` | 事实开始为真的时间 | 业务时间 |
| `invalid_at` | 事实被推翻的时间 | 业务时间 |
| `expired_at` | 系统标记"不再作为当前事实"的时刻 | 系统时间 |
| `reference_time` | 抽取时的参照时间 | 系统时间 |
| `attributes` | 结构化属性 | — |

### 其他边
`EpisodicEdge`（episode→实体 MENTIONS）、`CommunityEdge`（社区成员）、`HasEpisodeEdge`（saga→episode）、`NextEpisodeEdge`（episode 链）。

## 时间模型（双时间 + 逻辑删除）

```
业务时间轴：  valid_at ─────────────── invalid_at
系统时间轴：  created_at ── expired_at(now，被新事实推翻时)
```

- **事实变更 = 新增一条边 + 把旧边置 `invalid_at`**，不覆盖 `fact` 文本；
- 因此"同一对实体可有多条历史事实"，检索按时间窗筛选；
- `expired_at` 是"逻辑删除"，物理数据保留（`README` "invalidated — not deleted"）。

## 唯一性与分区

- 分区键：`group_id`（`nodes.py:96` "partition of the graph"）；
- 无全局唯一约束：实体唯一性靠**写入时去重**（identity properties / exact name / cosine ≥0.6 / LLM），不是数据库唯一索引；
- FalkorDB 下 **一个 group 一个 graph 数据库**（`decorators.py:56-70`）；Neo4j/Neptune 下同库内以 `group_id` 过滤。

## 数据模型演进（spec）

`spec/driver-operations-redesign.md` 目标：数据类变纯 Pydantic（无 DB 逻辑），DB I/O 由 `namespaces → operations ABC → driver` 承担。当前 `nodes.py/edges.py` 仍挂着 DB 方法（`get_by_uuid` 等），属 Phase 1 兼容期。

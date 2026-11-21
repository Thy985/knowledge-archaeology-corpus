# 01.1 · Architecture

## 1. 分层图（代码实证）

```
User / Host (Claude Code, Cursor, Codex, OpenCode, Qwen Code, Hermes, OpenClaw)
        │  (MemHub: harness grants / pairing / import lane)
        ▼
MemHub Core (FastAPI, memhub/apps/core/src/one_memory)
  ├── 管理面：source_grants、import queue、model profile、config、health
  └── 只存 memory_id 外键，不存记忆正文（ARCHITECTURE.md 原则 3）
        │  SDK API（HyMemoryClient）
        ▼
hy_memory/client.py  (7,826 行, 公开门面 + scene router + 事件循环)
        │
   ┌────┴─────────────────────────────┐
   ▼                                  ▼
Chat 链路                        Productivity 链路
MemoryWriter                    CodingWriter (variant = working | coding)
  ├─ L1 RAW upsert                ├─ WorkingFact (1/session)
  ├─ MemAgent.extract             ├─ Experience (0..N, value bar)
  ├─ Reconciler (ADD/…/DEL)       ├─ coding/store.py (SQLite)
  └─ L3 summary / L6 schema       └─ chain_store.py (0 LLM diff 链)
        │
        ▼
Storage adapters (data/)
  ├─ vector: chroma | qdrant | faiss
  ├─ graph:  kuzu | neo4j
  ├─ cache:  sqlite | mysql
  └─ kv / history / rdb
```

## 2. 依赖方向（单向，无环）

| 从 | 到 | 证据 |
|---|---|---|
| `client.py` | `pipelines` / `coding` / `agent` / `data` | import 段 `client.py:50-90` |
| `pipelines/writer.py` | `agent`（extractor/reconciler/mem_agent） | `writer.py:188-199` |
| `coding/writer.py` | `coding/store.py` + `coding/reconciler.py` | `coding/writer.py:237-249` |
| `coding/chain_store.py` | 无（仅 stdlib `sqlite3` + `subprocess`） | `chain_store.py:60` |
| MemHub Core | 仅 SDK 公开 API，不触碰 SDK 私有库 | `memhub/docs/ARCHITECTURE.md` 原则 1–3 |

**设计约束（文档声明 + 代码一致）**：MemHub 不得维护第二份记忆正文或自行判定 supersede；若 SDK 缺能力，先补 SDK 再在产品层调用。

## 3. 双系统（System 1 / System 2）

| | System 1 | System 2 |
|---|---|---|
| 触发 | 每次 `add()` | 仅 `evolve()`，且 `mode="ultra"` |
| 输入 | 本次写入内容 + 最近历史 | 该 user 的 VDB 全量 |
| 输出 | L1 raw、L2 fact、L7 intention、reconcile ops | L6 schema + cross-domain sweeper |
| 成本 | 每次 1~N 次 LLM | 全量扫描，**非幂等、无队列** |
| 代码 | `pipelines/writer.py` | `pipelines/system2_agent.py` + `client.evolve()` |

`evolve()` 的 docstring 明确：System 2 不在 add/write 中自动触发，也无进程内队列；lite/pro 调用抛 `RuntimeError`（`client.py:2292-2332`）。

## 4. 三种模式的成本阶梯

| mode | LLM 调用 | 图 | 能力 |
|---|---|---|---|
| `lite` | 0（纯 embedding） | 否 | 向量 RAG；忽略 scene/layers |
| `pro` | 写入时抽取 + reconcile | 可选（默认关） | 标准记忆 |
| `ultra` | pro + System 2 | 强制开 | 全认知架构 |

`client.py:3081`：`agent_mode = "disabled" if self._mode == "lite" else "full"`——**模式直接映射为是否运行 MemAgent**，不是 prompt 开关。

## 5. 关键架构选择与代价

| 选择 | 收益 | 代价 / 风险 |
|---|---|---|
| 单文件巨型 Client（7,826 行） | 统一门面、同步/异步双 API | god-class；任何路由改动影响面大 |
| 三场景共用 productivity pipeline | schema/pipeline 复用，维护成本低 | variant 差异靠 prompt 与存储后缀，语义漂移风险 |
| 版本链独立 SQLite、零 LLM | 确定性、可离线、成本低 | 与向量层写入解耦 → 两者可能短暂不一致 |
| MemHub 只做管理面 | 权威单一（SDK） | 产品层能力受 SDK API 边界限制 |
| System 2 手动触发 | 成本可控、可批量 | 用户不调用则 L6 schema 永远不生成 |

## 6. 扩展点

- 向量后端：`data/vector_store_{chroma,qdrant,faiss}.py`，统一 `VectorStoreBase`。
- 图后端：`data/graph_store_{kuzu,neo4j}.py`。
- 插件：`plugins/hermes`、`plugins/openclaw`（独立 HTTP / 原生 memory provider）。
- prompt 语言：`prompts/` + `agent/prompts_zh.py`，11 语言（README 声明；代码见 `utils/language` 相关引用）。

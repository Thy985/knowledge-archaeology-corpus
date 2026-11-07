# 01.1 Architecture

## 一句话
Graphiti 是"时序上下文图"的**构建与查询框架**：把 episode（原始输入）经 LLM 抽取为实体/事实，写入带有效期窗口的图，再用混合检索取回"现在/某时刻为真"的事实。它**不自带治理**——治理在 Zep 托管侧。

## 三层结构

```
应用（自行构建：ingestion / tenancy / access control / ops）
   │
   ├── 协议适配层（参考实现，非 hardened）
   │     ├── REST  server/graph_service（FastAPI，10 端点）
   │     └── MCP   mcp_server/src/graphiti_mcp_server.py（13 tools）
   │
   └── 核心库 graphiti_core
         ├── Graphiti 门面（graphiti.py:209）
         ├── Namespace 薄编排（graphiti_core/namespaces，733 行）
         ├── 维护管线（utils/maintenance：node/edge/community/graph_data）
         ├── 检索（search：BM25 / cosine / BFS / RRF / MMR / cross-encoder）
         ├── LLM Runtime（llm_client/llm_runtime.py，prompt→model 路由）
         └── Driver 抽象（driver.py:90 GraphDriver）
               ├── Neo4j
               ├── FalkorDB（一 group 一 graph）
               ├── Neptune
               └── Kuzu（deprecated）
```

## 核心抽象

| 抽象 | 语义 | 位置 |
|---|---|---|
| **Episode** | 原始输入 + provenance 根；所有派生事实可回溯到它 | `nodes.py:318` |
| **Entity** | 抽取出的实体，带 summary/attributes，随时间演化 | `nodes.py:499` |
| **EntityEdge（Fact）** | 三元组事实，带 `valid_at/invalid_at/expired_at` | `edges.py:263` |
| **Community** | 实体簇 + 区域摘要 | `nodes.py:727` |
| **Saga** | episode 的逻辑分组 + 增量总结 | `nodes.py:907` |

## 设计原则（README 明示）

1. **Temporal Fact Management**：事实有有效期；变化时**失效而非删除**。
2. **Episodes & Provenance**：每个实体/关系可回溯到产生它的 episode。
3. **Prescribed & Learned Ontology**：Pydantic 预定义类型，或让结构从数据涌现。
4. **Incremental Construction**：新数据即时整合，无需全量重算。
5. **Hybrid Retrieval**：语义 + 关键词 + 图遍历，不依赖 LLM 摘要。

## 关键架构事实

- **门面很薄**：`Graphiti.add_episode` 只做编排，具体抽取/去重/失效在 `utils/maintenance/*`。
- **数据模型正在"去 DB 化"**：`spec/driver-operations-redesign.md` 目标是让 `EntityNode/EntityEdge` 变纯 Pydantic，DB I/O 收到 Namespace→Operations ABC→Driver 三层；当前为 **Phase 1（非破坏）**，旧方法仍保留。
- **驱动可替换**：`GraphDriver` 是 ABC（`driver.py:90`），并提供 `clone(database)`（`driver.py:131`）用于 per-group 路由。
- **服务层明确是 reference implementation**：`SECURITY.md` 声明无认证/无授权/无隔离/无注入防护。

## 与 Mem0 的结构差异（对照）

| 维度 | Graphiti | Mem0（corpus 已有） |
|---|---|---|
| 记忆载体 | 时序知识图谱（实体+事实+时间窗） | 向量/抽取事实，v3 单遍 ADD-only、已移除图存储 |
| 写入 | episode → LLM 抽取 → 去重 → 时间失效 | 抽取 → 写入（并排） → 检索时去重 |
| 溯源 | 显式 `episodes[]` 链 | 弱（metadata） |
| 删除 | 引用计数 + 逻辑失效 | 版本/覆盖 |
| 治理 | 无（托管侧提供） | 托管平台与 SDK 能力分离 |

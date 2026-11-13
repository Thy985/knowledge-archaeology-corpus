# Coverage Gaps — 未覆盖面清单

> 本文件只列“没读/没跑”的范围，不用推测填充。所有 gap 都会影响某些结论的强度，已在对应 EK/KO 的 Scope 中收窄。

| Gap ID | 范围 | 为什么没覆盖 | 影响的知识 | 需要的后续动作 |
|---|---|---|---|---|
| G-01 | 向量/图存储内部查询计划（Qdrant/Milvus/Neo4j/pgvector） | 只读接口与配置，未逐行读 adapter | EK-32/33/36/37/38、KO-04 | 读各 adapter 的 `search`/filter 下推；跑一次 euclidean 对照 |
| G-02 | client SDK 重试/错误处理 | 未读 `packages/client` | C-02、KO-02 | 搜 `retry`/`add_episodes` 调用；确认异常是否携带 partial ids |
| G-03 | 测试执行 | 遵守只读 + 未启动依赖；本轮 S4=0 | 所有“测试揭示”表述 | 在隔离环境跑 `uv run pytest`（需 pg/qdrant/neo4j） |
| G-04 | `docs/` 全量 | 只读 README + agentic_memory | EV-081/082 的边界 | 读 `docs/api_reference`/`core_concepts` 其余页 |
| G-05 | Helm ingress / network policy | 未读 `deployments/helm/` | C-03、KO-05 | 读 ingress/auth 配置；确认默认是否暴露 |
| G-06 | 多 worker 运行时 | 未启动多进程 | EK-31/48、KO-08 | 起 2 个 server + 同一 PG，验证 lease/队列 |
| G-07 | TS client / 前端 | 未读 `packages/ts-client` | 无直接 KO | 如需 API 契约完整，再读 |
| G-08 | 语义存储 citation 级联 | 只读 PG/Neo4j delete 段 | C-06 | 查 ORM FK/级联；跑一次删除后查 citations |
| G-09 | OpenAPI 生成/契约 | 未跑 `generate-openapi` | EK-43 | 对比生成的 spec 与 `doc.py` |
| G-10 | 性能/容量 | 未跑 benchmark | KO-04/KO-08 的“成本”表述 | 需要真实负载；本轮不写性能结论 |

## 对结论的影响声明

- **不写“性能如何”**：本轮没有任何 benchmark。
- **不写“默认部署一定触发 -inf”**：默认 `SimilarityMetric=COSINE`，C-01 只在 EUCLIDEAN + 无 reranker 时成立。
- **不写“citation 一定孤儿”**：PG 侧已确认 `delete_features` 不删关联表；是否最终被 `delete_history` 清掉未验证。
- **不写“多 worker 一定安全/不安全”**：lease 代码存在，但未运行。

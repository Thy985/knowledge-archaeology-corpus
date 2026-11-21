# Engineering Knowledge · Failures, Boundaries & Configuration (EK-26 ~ EK-33)

## EK-26 开源仓库只交付"图构建框架"，治理能力在托管侧
- **type**: boundary · **layer**: L2 · **value**: A · **confidence**: high（官方文档自证）
- **statement**: README 对照表明确：Graphiti 提供 BYO 图数据库、BYO LLM/embedder、自定义 tenancy/access control/ops；Zep 提供用户/线程/内置摄取、预配置检索（sub-200ms）、Dashboard/CLI/SDK、SLA/支持/安全保证、Konig 图数据库。**开源仓库不实现企业治理**。
- **evidence**: E-52, E-51
- **links**: `causal:EK-27`, `contrast:EK-22`, `constraint:EK-26`
- **use**: 任何"Graphiti 具备 ABAC/审计/多租户隔离"的说法都必须标为"托管侧"，不能记到本仓库。

## EK-27 官方把安全缺口写成"显式负向清单"，并声明不在漏洞范围
- **type**: boundary · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: `SECURITY.md` 逐条列出：参考服务无认证、无授权、无 prompt-injection 防护、不隔离调用方、默认 DB 凭据；并把"缺少认证/授权/默认凭据""参考服务缺少注入防护""同 group 内不可信 episode 改变事实"列为 **out of scope** 的漏洞报告。
- **evidence**: E-51
- **links**: `causal:EK-28`, `causal:EK-29`, `contrast:EK-33`
- **note**: 这种"负向清单 + out-of-scope 声明"不是免责话术，而是**把缺失能力显式建模**，与 go-sdk 的 `rough_edges`/conformance baseline 属同一模式（见 Candidates 的跨项目假说）。

## EK-28 同一 `group_id` 内所有 episode 被"同等信任"——写路径没有来源分级
- **type**: failure/boundary · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: 官方原文："Graphiti gives the same trust to all episodes in a `group_id`. A newer episode can change or invalidate facts from an older episode. The application must decide which content to ingest and into which `group_id`."
- **evidence**: E-51
- **links**: `dependency:EK-06`, `causal:EK-04`, `contrast:EK-27`
- **implication**: 记忆投毒不是实现 bug，而是**信任模型的设计结果**：只要不可信内容能进入同一 group，它就能改写既有事实。防护必须在入库前（来源/身份/内容策略），不能依赖图内机制。

## EK-29 MCP 的破坏性工具与读工具同级暴露，无权限分层
- **type**: failure/boundary · **layer**: L2 · **value**: B · **confidence**: high
- **statement**: `clear_graph(group_ids)` 直接按 group 清空，无任何调用者校验；`delete_episode` / `delete_entity_edge` 同样暴露；`MCP server` 与读工具（search/get）共用同一工具面。
- **evidence**: E-49, E-48, E-51
- **links**: `causal:EK-27`, `dependency:EK-22`, `contrast:EK-24`
- **note**: 删除本身遵循引用计数（EK-24），但 `clear_graph` 是**整区清空**，绕过引用计数。

## EK-30 "全绿"是有条件的：CI 用 DISABLE 组合 + ignore 列表达成
- **type**: failure/testing · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: `make test` 未设 `DISABLE_NEO4J`，无 DB 会挂起；权威无 DB 门禁必须显式关闭四个后端并带 `--ignore`；`tests/test_add_triplet.py` 有 11 个既有失败且 CI 从不运行。服务端测试无 API key 时以 exit 5 self-skip（属预期）。
- **evidence**: E-53
- **links**: `contrast:EK-32`, `constraint:EK-17`
- **implication**: "测试通过"需注明是**哪一组命令**；不能把 CI 绿灯外推为"所有测试通过"。

## EK-31 FalkorDB 的并发模型与库层假设冲突
- **type**: failure/configuration · **layer**: L2 · **value**: B · **confidence**: high（官方 AGENTS 明示）
- **statement**: FalkorDB async 驱动在同一连接并发查询时会 "Connection closed by server"（`Graphiti.search` 内部的 gather 即可触发）；本地混合检索建议改用 Neo4j。同时 FalkorDB 需要"一 group 一 graph"的 clone 路由。
- **evidence**: E-53, E-06（E-19 的 group 路由）
- **links**: `constraint:EK-06`, `contrast:EK-18`, `causal:EK-17`
- **implication**: 同一份代码在不同后端上的可靠性不同；"后端可插拔"不等于"行为等价"。

## EK-32 存在明确标注的弃用面：`_search`、Kuzu extra、（历史上）共享 driver 改写
- **type**: failure/deprecation · **layer**: L2 · **value**: B · **confidence**: medium-high
- **statement**: `Graphiti._search` 标注 `DEPRECATED` 并转调 `search_`；Kuzu extra 注释"上游停维护，未来移除"；`#1676` 修复前的共享 driver 改写是已移除的旧路径。
- **evidence**: E-33, E-01, E-19
- **links**: `contrast:EK-30`, `causal:EK-33`, `subsystem:EK-18`

## EK-33 #1676 / #1659 / #1758：同一"共享可变状态"缺陷族的三次显形
- **type**: failure/regression · **layer**: L2 · **value**: A · **confidence**: high
- **statement**:
  - **#1676**：并发请求中途改写共享 `self.driver` → episode 写入错误 graph（写路径）；
  - **#1659**：单 group 查询走默认库 → 静默返回空结果（读路径）；
  - **#1758**：decorator 内 pop `group_ids` 导致位置参数错位（`driver` 重复赋值 TypeError）→ 参数绑定错误。
  三者共享同一类前提（请求相关信息被放入实例状态或可变参数结构），修复方向一致（per-call 绑定 + 显式传递）。
- **audit_note**: "同一根因"是审计推断，非维护者结论；三者是否确属同一 defect family 需 upstream 确认。
- **evidence**: E-19, E-18（decorators.py 注释含 #1659/#1758）
- **links**: `causal:EK-05`, `mechanism:EK-05`, `contrast:EK-21`
- **note**: 同一根因在写、读、参数传递三个位置各暴露一次，说明这是结构性缺陷，不是三个独立 bug。

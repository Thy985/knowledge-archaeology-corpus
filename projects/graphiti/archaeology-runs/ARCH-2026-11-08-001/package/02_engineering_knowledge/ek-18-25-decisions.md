# Engineering Knowledge · Key Decisions (EK-18 ~ EK-25)

## EK-18 选择"BYO 图数据库"：支持 4 后端，但只把 Neo4j 作为默认依赖
- **type**: decision · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: 核心依赖只含 `neo4j`；FalkorDB / Neptune / Kuzu 作为 optional extra。Kuzu 已被注释为"上游停维护、未来移除"，但仍保留驱动代码。
- **evidence**: E-01, E-40
- **links**: `subsystem:EK-17`, `contrast:EK-22`, `constraint:EK-31`
- **trade-off**: 降低默认安装成本 vs 多后端行为差异（FalkorDB 分库、并发断连、能力缺失）需由使用方承担。

## EK-19 Namespace 薄编排层：把"嵌入/tracing/编排"与"纯 DB I/O"分开
- **type**: decision · **layer**: L2 · **value**: B · **confidence**: medium（spec 为 Draft，实现部分落地）
- **statement**: spec 定义目标架构为 Client → Namespace（薄编排）→ Operations ABC（纯 DB I/O）→ Driver。仓库中已存在 `graphiti_core/namespaces/`（733 行），但数据类仍保留 DB 方法；spec 自身标 Draft/Phase 1 非破坏。
- **audit_note**: 只陈述"目录存在 + spec 状态 + 旧方法仍在"，**不推断完成度**；不能说"重构已完成"。
- **evidence**: E-42, E-43
- **links**: `dependency:EK-17`, `causal:EK-18`, `contrast:EK-17`
- **note**: 这是"先加新层、再抽旧职责"的渐进重构，不能表述为"重构已完成"。

## EK-20 LLMRuntime 用"类型化路由"替代运行时模型名拼接
- **type**: decision · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: `LLMRuntime` 要求一个默认 `LLMModel`；prompt 可按 group 路由到不同模型，prompt 级路由优先于 group 级；未知 prompt 名在构造期即报错；`BUILTIN_PROMPT_SPECS` 不可由用户改 schema。
- **evidence**: E-44, E-45
- **links**: `mechanism:EK-21`, `contrast:EK-14`（同为"策略选择但机制不同：模型路由 vs 检索配方）
- **note**: 设计动机是消除"caller 自创模型别名"导致的隐性不匹配——**把配置错误前移到构造期**。

## EK-21 LLM 并发调用必须保证不串模型
- **type**: decision · **layer**: L2 · **value**: B · **confidence**: high（有测试）
- **statement**: 运行时对并发 complete 做隔离，测试 `test_concurrent_completes_do_not_serialize_or_clobber_model` 明确断言"不串行、不覆盖 model"。
- **evidence**: E-46
- **links**: `dependency:EK-20`, `mechanism:EK-05`
- **note**: 与 #1676 同属"共享可变状态在 await 点被并发改写"的家族问题，但此处已用测试固定。

## EK-22 协议服务是 reference implementation，不是产品
- **type**: decision · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: `server/` 与 `mcp_server/` 的定位是"演示如何把 Graphiti 放到协议后面"；官方声明它们未 hardened，生产需自行加认证、授权、网络控制。
- **evidence**: E-51, E-52
- **links**: `causal:EK-26`, `causal:EK-27`, `contrast:EK-18`

## EK-23 写接口用 202 + 单消费者队列，把"受理"与"完成"解耦
- **type**: decision · **layer**: L2 · **value**: B · **confidence**: high
- **statement**: `POST /messages` 立即返回 202，把每条消息放入 `async_worker.queue`，由队列串行执行 `add_episode`。这与库文档"建议后台队列串行"一致。
- **evidence**: E-50, E-11（docstring）
- **links**: `dependency:EK-01`, `causal:EK-24`, `constraint:EK-31`
- **note**: 串行化的原因是写入管线内部状态（previous episodes、去重候选）对并发不友好；代价是吞吐受限。

## EK-24 删除是引用计数：只有"最后一个支持者"消失才删事实
- **type**: decision · **layer**: L2 · **value**: A · **confidence**: high
- **statement**: `remove_episode` 从每条边的 `episodes[]` 移除该 episode：仍有其他 episode → 更新；无 → 删除。节点按 `MENTIONS` 计数，仅被该 episode 提及才删。
- **evidence**: E-10, E-05
- **links**: `dependency:EK-02`, `contrast:EK-03`, `mechanism:EK-25`

## EK-25 社区是"可选、增量"的派生产物，不在主干写路径
- **type**: decision · **layer**: L2 · **value**: B · **confidence**: high
- **statement**: 社区只在 `update_communities=True` 时更新，或显式 `build_communities`；`update_community` 把新实体并入既有社区。默认写入不付出社区计算成本。
- **evidence**: E-07, E-16（graphiti.py:1532-1545）
- **links**: `causal:EK-13`, `contrast:EK-01`, `subsystem:EK-14`
- **note**: 社区摘要与实体摘要同属"压缩视图"，但更新时机不同——前者显式触发，后者随写更新。

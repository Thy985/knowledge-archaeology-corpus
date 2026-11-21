# Knowledge Layer（6 个 KO · 最高 L3）

> 升维纪律：只有 L3（可迁移模式）且 `Cross-project validation pending`。**本轮不产出 L4/L5**——单一仓库证据不足以支撑"稳定关系"或"行动准则"。

## KO-01 · 三级记忆的本质是三种上下文准入策略
- **level**: L3 Pattern
- **aggregation_rule**: R1 机制簇 —— EK-01 / EK-02 / EK-14 / EK-16 / EK-17 共享"记忆如何进入上下文"这一机制
- **statement**: 分层记忆的差异主要体现在三种准入方式：Core 常驻（每次请求都付 token）、Recall 压缩（付一次 LLM 摘要调用，换掉原文保真）、Archival 按需（付检索成本，命中时才进入上下文）。本仓库把三者分别实现为 block / messages+summarizer / passage+vector 三个子系统。
- **解释范围扩大**: 从"Letta 有三个字段"扩到"任何分层记忆系统的分层依据应是准入成本，而非存储介质"。
- **反例（3）**:
  1. Mem0 把记忆压成抽取事实 + 向量检索，没有常驻层——证明"三层"不是必需形态。
  2. Graphiti 用时间图统一存储，Core/Archival 的边界由查询而非层级决定。
  3. 小上下文模型可能把全部记忆常驻，分层反而增加复杂度。
- **epistemic**: `Pattern (Cross-project validation pending)`
- **value**: A
- **对照**: mem0（已考古）、graphiti（已考古）、letta-code（已考古）

## KO-02 · 版本化迁移的中间态必然出现"双写无事务"
- **level**: L3 Pattern
- **aggregation_rule**: R2 因果链簇 —— EK-04（DB 快照）→ EK-05（git 声明为 truth）→ EK-06（补偿），并由 EK-20 约束
- **statement**: 当一个系统把记忆的权威源从数据库迁到版本化存储（git/对象存储）时，可能形成"分模式"的中间态：非 git 模式写数据库；git 模式先 commit 到 git、再同步数据库作为缓存。两条路径都没有跨存储事务，只能靠 backfill / sync 做补偿。设计上必须回答：哪一侧可丢、冲突谁赢、删除如何传播。
- **解释范围扩大**: 从"Letta 的 block_manager_git"扩到"任何 truth-source 迁移的中间态"。
- **反例（3）**:
  1. 若迁移采用双写 + 事务性 outbox，可能不存在该中间态（本仓库未见 outbox）。
  2. letta-code 直接以文件/ git 为唯一源，Postgres 不在写路径——说明中间态可以被跳过。
  3. 只读副本场景（如分析库）不需要冲突解决，双写问题弱化。
- **epistemic**: `Pattern (Cross-project validation pending)`
- **value**: A
- **本仓库证据**: `block_manager_git.py:565-568`、`:359-484`；无 outbox / 无冲突解决代码。

## KO-03 · 安全约束绑在"执行路径"上，不绑在"对象"上，会产生影子路径
- **level**: L3 Pattern
- **aggregation_rule**: R3 不变量簇 —— EK-07 / EK-09 / EK-10 / EK-11 / EK-13 汇聚到同一不变量："写入必须经过门禁"
- **statement**: 当同一语义操作存在多代实现（legacy 函数 + v2/v3 executor），而约束只加在新路径上时，安全性取决于"旧路径是否仍被注册/可达"，而不是取决于对象本身是否只读。约束应放在唯一的写入入口，或让旧入口失效。
- **解释范围扩大**: 从"Letta 的 core_memory_append 绕过 read_only"扩到"任何多代 API 并存的系统"。
- **反例（3）**:
  1. 若工具注册表在启动时只剩 v2/v3，legacy 函数不可达，则风险仅为理论。
  2. 若 legacy 函数只在内部 batch/sleeptime 路径使用且不暴露给模型，风险受限。
  3. 若对象层（ORM/DB constraint）也强制 read_only，则执行层遗漏不构成绕过。
- **epistemic**: `Pattern (Cross-project validation pending)`；**"legacy 是否仍注册"未证实**（见 CAND-03）
- **value**: A

## KO-04 · 共享记忆的授权无法由资源级属性表达
- **level**: L2 Observation（**独立审计后由 L3 降级**，原判定 OVER_GENERALIZED）
- **aggregation_rule**: R3 不变量簇 —— EK-03（多宿主共享）/ EK-18（租户过滤）/ EK-19（块级 read_only）
- **statement**: 本仓库中 `read_only` 是 block 字段，而 block 可被多 agent 共享；因此当前数据模型**不能表达**"同一 block 对 A 可写、对 B 只读"。这是模型表达能力的事实缺口，尚未观察到实际越权案例。
- **解释范围扩大**: 从"Letta block 的 read_only 字段"扩到"任何共享记忆/共享文档的权限设计"。
- **反例（3）**:
  1. 若产品语义是"共享即同权"，资源级只读是合理简化。
  2. 若共享仅发生在同一信任域（同一 user 的多个 agent），per-agent 差异无意义。
  3. 若上层用 group/identity 做授权，block 级只是缓存，不构成缺口。
- **epistemic**: `Observation`（**已降级**：不能表达 ≠ 必然导致越权；实际共享写扩散见 CAND-06）
- **value**: A
- **对照**: letta-code 用"路径墙"解决同类问题（KO-03 of letta-code），可作跨项目对照。

## KO-05 · 以渲染产物做变更检测会把持久化耦合到表示层
- **level**: L2 Observation（未升 L3）
- **aggregation_rule**: R4 主题簇（单点，EK-08）
- **statement**: `update_memory_if_changed_async` 用 `new_memory_str not in system_message.content[0].text` 判断是否写库；渲染格式、prompt 其它区段的内容都可能影响判定。持久化正确性因此依赖表示层的稳定性。
- **为什么不升 L3**: 只有一处实现、无反例、未观察到实际故障；属于"设计脆弱点"而非"可迁移模式"。
- **epistemic**: `Observation`
- **value**: B

## KO-06 · 多租户是"查询级软隔离"
- **level**: L3 Pattern
- **aggregation_rule**: R3 不变量簇 —— EK-18（OrganizationMixin）/ EK-19（共享对象跨主体）
- **statement**: 所有租户共用同一套表，隔离由每次查询的 `organization_id`/`project_id` 条件保证。这类隔离的正确性依赖"每个 DAO 都带 actor 过滤"，是代码纪律而非数据库强制；一旦某条查询漏掉过滤，就是跨租户泄露。
- **解释范围扩大**: 从"Letta 的 ORM mixin"扩到"任何 schema-per-tenant vs shared-schema 的选择"。
- **反例（3）**:
  1. 若存在数据库行级安全（RLS）策略，软隔离被兜底。
  2. 若部署模型是单租户实例，隔离问题不存在。
  3. 若所有查询集中在少数 DAO 且强制传入 actor，风险可控。
- **epistemic**: `Pattern (Cross-project validation pending)`
- **value**: B
- **未证实**: 是否存在 RLS（本轮未在迁移/ORM 中检索到）。

## 汇总

| KO | 层级 | 规则 | 证据 EK | 反例 | Epistemic | 价值 |
|---|---|---|---|---|---|---|
| KO-01 | L3 | R1 | 01/02/14/16/17 | 3 | Pattern (pending) | A |
| KO-02 | L3 | R2 | 04/05/06/20 | 3 | Pattern (pending) | A |
| KO-03 | L3 | R3 | 07/09/10/11/13 | 3 | Pattern (pending) | A |
| KO-04 | **L2（降级）** | R3 | 03/18/19 | 3 | Observation | A |
| KO-05 | L2 | R4 | 08 | — | Observation | B |
| KO-06 | L3 | R3 | 18/19 | 3 | Pattern (pending) | B |

**配比检查**：40 Facts → 23 EK → 6 KO（其中 L3×4、L2×2）。低于 skill 建议的 7~12 KO，原因是本轮限定 archive 分支且未实跑；**不为凑数制造 KO**。

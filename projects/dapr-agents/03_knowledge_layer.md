# Generalized Knowledge Layer — Dapr Agents（7 个 KO）

> 每个 KO 声明 aggregation_rule（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）；跨项目未验证标 `Cross-project validation pending`。

## KO-D01（L4 · R1 机制簇）Workflow 是 agent 执行骨架
**陈述**：agent 的每一步（LLM 推理、工具调用、HITL 审批、子 agent 调度、多 agent 编排）都可建模为 durable workflow 的步骤原语（call_activity / call_child_workflow / wait_for_external_event）；框架的价值 = 把 agent 控制流转译为可持久化、可重放的步骤序列。
- aggregation_rule: R1 机制簇（EK-D03/D04/D10/D18 共享"步骤原语"机制，跨 4 个独立子系统：agents core / executor / tool / orchestration）
- 解释范围扩大：任何需要崩溃恢复的 agent 框架都可选择 workflow 骨架
- Cross-project validation pending（restate 的 handler 步骤 / dbos 的 transaction 步骤 / infinity 的 slice 对照）

## KO-D02（L4 · R2 因果链簇）HITL 审批 = 暂停-恢复闭环
**陈述**：人对工具调用的审批不是旁路，而是执行流中的一等暂停点：暂停（PausedExecutorRun）→ 发布事件 → 等待外部事件/超时 → 带决策恢复 → 循环，且必须有限额（max_approval_rounds）防死锁。
- aggregation_rule: R2 因果链簇（EK-D04→D05→D06 形成"暂停→等待→恢复→上限"完整链）
- 解释范围扩大：HITL 在 durable execution 中必须与重放兼容（确定性 id/事件）
- 证据强度：S4（审批循环代码 + 上限逻辑）；中断 demo 未跑（无 Docker），运行级 S5 待补

## KO-D03（L4 · R3 不变量簇）重放安全是分布式 agent 正确性的前置条件
**陈述**：event sourcing 重放下，workflow 代码中的非确定性操作（时钟、日志、随机、I/O）会漂移——因此必须强制 activity 边界（call_activity）+ 确定性 ID + 确定性时间（ctx.current_utc_datetime）+ 重放日志抑制。破坏确定性 = 恢复出错（升级注释：hook 决策按位置键控的非确定性失败警告）。
- aggregation_rule: R3 不变量簇（EK-D03/D07/D20 汇聚到同一安全不变量：重放确定性）
- 解释范围扩大：任何事件溯源/日志重放系统（含 restate journal、infinity slice 重放）共享此不变量
- 证据强度：S4（代码纪律 + 升级注释明确警告）

## KO-D04（L3 · R4 主题簇）会话记忆与执行状态分离（双 store）
**陈述**：agent-memory（会话历史，给 LLM 上下文）与 agent-workflow（durable 执行状态，给运行时恢复）是两种生命周期、两种访问模式的状态，必须分离存储；执行状态进一步抽象为 schema bundle（按 agent 类型选模型）。
- aggregation_rule: R4 主题簇（EK-D08/D09 覆盖双 store 的存储与注入两维度）
- 解释范围扩大：Memory 周架构参照（Tafcm 记忆分层同构）
- 证据强度：S4（configs + session store 实现）

## KO-D05（L5 · R1 机制簇）Agent 即 workflow（步骤持久化恢复）
**陈述**：把"agent 状态"建模为 workflow 实例（每步持久化、重放恢复）后，进程崩溃不再是状态丢失而是"从最后持久化步骤续跑"；恢复语义 = 重放不重执行（已完成 LLM/工具调用不重复）——与 infinity 的"agent 即数据"（挂起恢复零占用）是同一自由度的两个实现端点。
- aggregation_rule: R1 机制簇（EK-D03/D04/D10/D18 共享"步骤持久化"机制，跨 4 子系统）
- 解释范围扩大：durable agent 的两条路线（步骤重放 vs 状态挂起）统一视角
- **Cross-project validation pending**：三路线（Dapr 重放 / infinity 挂起 / restate-dbos 混合）10-30 品类 9 向对照（KnowlegeMap）支持但仓库级交叉验证未完成
- 反例面：需要真实长连接/外部副作用的步骤（工具回调、websocket）在重放模型下需特殊处理（activity 幂等/确定性 id）

## KO-D06（L4 · R2 因果链簇）executor 至少一次交付 + 会话恢复
**陈述**：工具/推理回合作为 activity 获得至少一次投递语义；重试恢复前次会话（任务重发、已执行副作用可能重跑）——因此工具副作用必须可重入/幂等；不可重试失败显式 fail workflow。交付保证来自 activity 语义而非工具实现。
- aggregation_rule: R2 因果链簇（EK-D04→D19 形成"至少一次→会话恢复"链）
- 解释范围扩大：与 infinity 的"dispatch barrier + error result"（fail 而非悬挂）异曲同工
- 证据强度：S4（模块 docstring 明示 delivery semantics）

## KO-D07（L3 · R4 主题簇）agent-as-tool（子 agent = 子 workflow）
**陈述**：把一个 agent 作为另一个 agent 的工具（AgentWorkflowTool / registry agents-as-tools / MCP 网关），统一为"调度一个子 workflow 并收集结果"；确定性 child instance id 使子 workflow 可恢复。工具接入三族：本地同步 / agent 子 workflow / MCP 网关。
- aggregation_rule: R4 主题簇（EK-D10/D11/D16 覆盖 agent-as-tool 的三族接入）
- 解释范围扩大：多 agent 系统组合性的基础设施层实现
- 证据强度：S4

## KO 统计
- 7 KO：R1×2（D01/D05）/ R2×2（D02/D06）/ R3×1（D03）/ R4×2（D04/D07）
- 层级：L3×2（D04/D07）/ L4×4（D01/D02/D03/D06）/ L5×1（D05）
- Cross-project pending：D01/D03/D05（3 个，durable execution 四路线对照核心）

# Generalized Knowledge Layer — Infinity（8 个 KO）

> 每个 KO 声明 aggregation_rule（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）；升维判断经 Abstraction Gate；跨项目未验证标 `Cross-project validation pending`。

## KO-D01（L4 · R4 主题簇）Slice = 有界执行单元
**陈述**：可挂起 agent runtime 的最小执行单元是"加载状态→一次模型补全→持久化→至多一个外部副作用→结束"的有界 slice；step 之间线程不持有任何 in-flight 资源，因此可整体卸载。
- aggregation_rule: R4 主题簇（EK-D01/D04/D09/D23 覆盖 slice 的定义、工具分派、持久化、配置——同一主题互补维度）
- 解释范围扩大：从"这个 runtime 的 step"推广为"可挂起 agent 的执行单元抽象"
- Cross-project validation pending（对照 restate/dbos 的执行单元）

## KO-D02（L4 · R2 因果链簇）Commit/Dispatcher 双屏障
**陈述**：外部副作用（工具调用）必须在其可被观察的因果后果（结果、事件）到达之前已持久化——commit barrier 保证调用先于结果 durable，dispatch barrier 保证失败以 error result 恢复而非悬挂。
- aggregation_rule: R2 因果链簇（EK-D02→D03→D06 形成"持久化→分派→结果校验"完整因果链）
- 解释范围扩大：任何"先副作用后持久化"的 agent 系统都存在窗口漏洞
- Cross-project validation pending（restate commit-before-respond 对照）

## KO-D03（L4 · R3 不变量簇）消息 ID 幂等 + 反向走查 = 至少一次投递安全网
**陈述**：在 at-least-once 投递上构建 agent 状态的正确性，靠两个不变量：已处理消息 ID 拒绝重复；工具结果对历史尾部反向走查判 陈旧/重复/无主——不依赖传输层的恰好一次。
- aggregation_rule: R3 不变量簇（EK-D05/D06/D16 汇聚到同一安全不变量：处理幂等）
- 解释范围扩大：SQS/重试/重复投递是所有 serverless agent 的普遍现实
- 证据强度：S4（测试验证：handle_content 判重测试 + lambda DynamoDB 实现）

## KO-D04（L3 · R2 因果链簇）中断 = 合成事件，而非撕裂状态
**陈述**：用户输入打断 pending 工具调用时，运行时注入合成 "interrupted" 结果保持 history well-formed；未经验证（无模型输出背书）的输入不持久化，可安全重发——中断是历史中的正常事件而非异常路径。
- aggregation_rule: R2 因果链簇（EK-D07→D08→D10：中断→未验证处理→defer 上下文）
- 解释范围扩大：任何支持用户中途打断的 agent 都需要"中断结果"协议
- 证据强度：S4

## KO-D05（L5 · R1 机制簇）Agent 即数据（纯状态化挂起）
**陈述**：把 agent 建模为"纯数据 + 事件驱动 slice"后，同一代码可在 resident 进程与 serverless（规模到零）运行——挂起成本从进程资源降为存储成本，并发密度由存储而非内存决定。
- aggregation_rule: R1 机制簇（EK-D01/D12/D13/D15 共享"状态数据化、driver 可弃"机制，跨 4 个独立子系统：core/daemon/router/lambda）
- 解释范围扩大：agent runtime 设计的核心自由度从"进程管理"迁移到"状态管理"
- **Cross-project validation pending**：与 restate（journal 数据化）/ dbos（状态表化）构成三路线统一视角——10-30 品类横向验证
- 反例面：需真实工具服务器/长连接的工具仍会破坏纯数据假设（RAP 的存在即证明）

## KO-D06（L3 · R4 主题簇）Defer 语义是平台契约，不是实现细节
**陈述**：synthetic 事件在工具调用 in-flight 时的处理（推迟 vs 立即打断）必须作为平台能力显式建模（DeferQueue trait）；resident 可内存持有，serverless 只能接受中断语义或依赖传输层重排队。
- aggregation_rule: R4 主题簇（EK-D10/D15/D17 覆盖 defer 的平台差异）
- 解释范围扩大：任何跨 resident/serverless 的 agent 框架都要面对
- 证据强度：S4（NoDeferral 注释明确语义）

## KO-D07（L4 · R1 机制簇）Driver 生命周期 = 资源绑定协议
**陈述**：driver 的 Live/Idle 状态与资源占用（active_threads 集合）在同一位置原子变更，杜绝"channel 与 set 不一致"；Idle 即释放（router 池），订阅不维持驻留——生命周期报告是资源回收的权威信号。
- aggregation_rule: R1 机制簇（EK-D11/D12/D13 共享"生命周期状态=资源绑定"机制，跨 router/driver 子系统）
- 解释范围扩大：agent runtime 的"空转成本"由生命周期设计决定
- 证据强度：S4（系统测试：driver idle + 事件 respawn）

## KO-D08（L4 · R4 主题簇）Serverless step 模式 = 平台原语重绑定
**陈述**：同一 slice 语义在不同部署面通过重绑定平台原语实现：输入=SQS 消息、幂等=DynamoDB、历史=Postgres、定时器=EventBridge Scheduler、sleep=特殊化工具；工具集随部署面变化而非核心执行模型变化。
- aggregation_rule: R4 主题簇（EK-D15/D16/D17/D18 覆盖 serverless 输入/存储/定时/RAP）
- 解释范围扩大：框架可移植性的边界在"平台适配层"而非核心
- Cross-project validation pending（dbos Lambda 部署对照）

## KO 统计
- 8 KO：R1×2（D05/D07）/ R2×2（D02/D04）/ R3×1（D03）/ R4×3（D01/D06/D08）
- 层级：L3×2（D04/D06）/ L4×5（D01/D02/D03/D07/D08）/ L5×1（D05）
- Cross-project pending：D01/D02/D05/D08（4 个，与 restate/dbos 构成三路线对照）

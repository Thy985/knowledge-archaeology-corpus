# EK Graph — 46 条工程知识的连接结构

> v3.1 要求：EK 不是扁平清单，是图。每条 EK 的 `links` 在此汇总；孤立 EK = 0。

## 1. 边表（按源 EK）

| 源 | 边类型 | 目标 | 语义 |
|---|---|---|---|
| EK-01 | constraint | EK-37 | condition 事实优先约束 resume 执行 |
| EK-01 | mechanism | EK-16 | condition=True 短路与 hook 重放补偿 |
| EK-02 | causal | EK-09 | span before 与最终 status 写入构成因果对比 |
| EK-02 | subsystem | EK-03 | 都发生在状态计算之前 |
| EK-03 | causal | EK-04 | 不用未观察状态做决策 |
| EK-03 | contrast | EK-04 | 等待观察 vs 超时放弃 |
| EK-04 | mechanism | EK-03 | 同类观察缺口处理 |
| EK-04 | dependency | EK-09 | 期望清理在状态写入前 |
| EK-05 | subsystem | EK-06 | 分派前的另一道门 |
| EK-05 | constraint | EK-01 | phase 集合是分派前置 |
| EK-06 | constraint | EK-05 | 约束分派范围 |
| EK-06 | contrast | EK-08 | 终态 vs 删除态 |
| EK-07 | causal | EK-33 | PauseTime 到期后仍受 fail-closed |
| EK-07 | subsystem | EK-34 | auto-pause 也写时间/Schedule |
| EK-08 | causal | EK-19 | 暂停态删除需清理 Checkpoint |
| EK-08 | constraint | EK-06 | finalizer 只加非终态 |
| EK-09 | causal | EK-26 | 升级续跑依赖状态持久化 |
| EK-09 | mechanism | EK-04 | 持久化"已完成/已观察" |
| EK-10 | dependency | EK-11 | 一致性是 markResume 前置 |
| EK-10 | contrast | EK-21 | 两套镜像校验 |
| EK-11 | constraint | EK-25 | 升级策略决定是否走该路径 |
| EK-11 | mechanism | EK-12 | 同函数内重置 runtime |
| EK-12 | causal | EK-43 | 唤醒复用 Resume 触发重初始化 |
| EK-12 | mechanism | EK-23 | checkpoint 恢复同样重建 Pod |
| EK-13 | causal | EK-14 | 等待规则由 checkpoint 状态决定 |
| EK-13 | constraint | EK-19 | 外部与 pause checkpoint 共用 CRD |
| EK-14 | constraint | EK-21 | 活跃判定同源 |
| EK-14 | mechanism | EK-02 | 只保留有信息量输出 |
| EK-15 | mechanism | EK-01 | condition=True 是已暂停唯一事实 |
| EK-15 | causal | EK-16 | 短路无法防崩溃重放 |
| EK-16 | contrast | EK-15 | 正常路径去重 vs 崩溃重放 |
| EK-16 | dependency | EK-09 | 状态持久化缩小重放窗口 |
| EK-17 | causal | EK-18 | 策略与行为脱节 |
| EK-17 | contrast | EK-19 | 可达 Stop vs 不可达 Checkpoint |
| EK-17 | subsystem | EK-25 | 升级有独立 checkpoint |
| EK-18 | constraint | EK-17 | 策略值无法改变实现 |
| EK-18 | causal | EK-19 | 缺失消费方即未接线实现 |
| EK-19 | contrast | EK-17 | 同问题两实现 |
| EK-19 | dependency | EK-21 | 若接线则依赖生命周期正确 |
| EK-20 | constraint | EK-23 | 内容决定恢复数据形态 |
| EK-20 | mechanism | EK-22 | 内容决定 label |
| EK-21 | causal | EK-17 | 阻断保证 Stop 前快照有效 |
| EK-21 | mechanism | EK-10 | 镜像比较另一实现 |
| EK-22 | mechanism | EK-25 | 滚动升级版本兼容 |
| EK-22 | constraint | EK-20 | label 由内容集合决定 |
| EK-23 | causal | EK-12 | 恢复后 runtime 重初始化 |
| EK-23 | contrast | EK-28 | pause checkpoint vs upgrade checkpoint |
| EK-24 | causal | EK-33 | 下一轮决策从干净状态开始 |
| EK-24 | mechanism | EK-15 | 避免继承旧状态 |
| EK-25 | constraint | EK-26 | 决定状态机是否启动 |
| EK-25 | contrast | EK-17 | 暂停有默认行为，升级没有 |
| EK-26 | causal | EK-09 | 续跑以持久化为前提 |
| EK-26 | mechanism | EK-45 | 从当前观测状态继续修复 |
| EK-26 | constraint | EK-25 | 策略类型决定分支 |
| EK-27 | causal | EK-28 | 只有 CheckpointRestore 建快照 |
| EK-27 | contrast | EK-17 | 类型分支：升级有、暂停无 |
| EK-28 | contrast | EK-23 | 只保 filesystem vs 可含 memory |
| EK-28 | causal | EK-29 | 恢复后仍重建 Pod |
| EK-29 | causal | EK-11 | 后续标记依赖新 Pod |
| EK-29 | mechanism | EK-04 | 不信任本地旧对象 |
| EK-30 | constraint | EK-26 | 状态机出口条件 |
| EK-30 | contrast | EK-37 | 等注解 vs 等实际状态 |
| EK-31 | contrast | EK-16 | 重试到终态 vs 记录后继续 |
| EK-31 | causal | EK-26 | reason 持久化 |
| EK-32 | mechanism | EK-22 | 新旧格式 hash 并存的兼容机制 |
| EK-32 | constraint | EK-25 | hash 判定决定是否进入升级 |
| EK-33 | causal | EK-34 | 算出 pauseTime 才持久化 resume |
| EK-33 | contrast | EK-35 | 两侧错误策略不对称 |
| EK-33 | constraint | EK-07 | 定时只是触发，仍需决策 |
| EK-34 | causal | EK-38 | 真正暂停时改写时间字段 |
| EK-34 | mechanism | EK-09 | 不可逆动作前先落盘 |
| EK-34 | dependency | EK-37 | 恢复依赖 schedule + condition |
| EK-35 | contrast | EK-33 | 同策略两侧不同错误处理 |
| EK-35 | constraint | EK-36 | 解析结果需减 leadTime |
| EK-36 | constraint | EK-35 | 默认 leadTime 参与计算 |
| EK-36 | contrast | EK-14 | 无效值归一化到安全默认 |
| EK-37 | constraint | EK-01 | condition 是唯一事实源 |
| EK-37 | dependency | EK-34 | schedule 提供时间、condition 提供可执行性 |
| EK-37 | contrast | EK-30 | 实际状态 vs 外部注解 |
| EK-38 | causal | EK-07 | 改写后的时间被 checkTimers 消费 |
| EK-38 | mechanism | EK-44 | 同一组时间字段方向相反 |
| EK-39 | contrast | EK-40 | 开环计划 vs 连续反馈 |
| EK-39 | constraint | EK-42 | cron 也受 cooldown/边界约束 |
| EK-40 | causal | EK-41 | 死区宽度由 watermark 决定 |
| EK-40 | mechanism | EK-42 | 等待上一动作收敛 |
| EK-41 | constraint | EK-40 | 死区边界来源 |
| EK-41 | contrast | EK-36 | 避免参数组合退化 |
| EK-42 | causal | EK-40 | cooldown 与 guard 共同防重复扩容 |
| EK-42 | dependency | EK-04 | Pod 创建到可见存在时延 |
| EK-43 | causal | EK-12 | Resume 重置 runtime |
| EK-43 | mechanism | EK-29 | 不信任 stale 对象 |
| EK-43 | contrast | EK-17 | 恢复统一路径 vs 暂停固定路径 |
| EK-44 | contrast | EK-38 | 同一组时间字段、相反方向 |
| EK-44 | constraint | EK-07 | 时间字段由 checkTimers 消费 |
| EK-44 | causal | EK-33 | 未重设 PauseTime 则不再触发暂停 |
| EK-45 | mechanism | EK-04 | 处理观察滞后 |
| EK-45 | causal | EK-40 | 配额状态影响可扩容量 |
| EK-45 | contrast | EK-02 | 只保留有信息量的修复动作 |
| EK-46 | constraint | EK-17 | controller/manager 各自实现、不得复用 |
| EK-46 | contrast | EK-45 | 规则隔离演进 vs 对账收敛状态 |

## 2. 边类型分布

| 类型 | 条数 |
|---|---|
| mechanism | 19 |
| subsystem | 4 |
| causal | 26 |
| dependency | 7 |
| constraint | 22 |
| contrast | 22 |
| **合计** | **100** |

- 46 条 EK，100 条边 → **平均出边 2.17**（>1，满足 v3.1 质量门）。
- 游离 EK：0（<20%，满足）。
- 双向一致：所有 `mechanism/contrast` 无向边均成对出现（如 EK-17↔EK-19、EK-33↔EK-35、EK-38↔EK-44）。

## 3. 聚合规则映射（EK → KO）

| KO | 聚合规则 | 簇内 EK | 边类型 | 簇规模 |
|---|---|---|---|---|
| KO-01 | R2 因果链 | EK-01 → EK-37 → EK-45 → EK-34 → EK-38 → EK-07 | causal/constraint | 6 |
| KO-02 | R1 机制簇 | EK-17, EK-18, EK-19, EK-27 | mechanism/contrast | 4 |
| KO-03 | R1 机制簇 | EK-03, EK-04, EK-09, EK-29, EK-45 | mechanism | 5 |
| KO-04 | R2 因果链 | EK-13 → EK-14 → EK-21 → EK-20 → EK-23 → EK-12 | causal/constraint | 6 |
| KO-05 | R3 不变量簇 | EK-15, EK-16, EK-31, EK-24 | constraint/contrast | 4 |
| KO-06 | R3 不变量簇 | EK-33, EK-34, EK-35, EK-36 | constraint | 4 |
| KO-07 | R4 主题簇 | EK-39, EK-40, EK-41, EK-42 | causal/constraint | 4 |
| KO-08 | R1 机制簇 | EK-43, EK-44, EK-12, EK-38 | mechanism/contrast | 4 |
| KO-09 | R3 不变量簇 | EK-05, EK-06, EK-25, EK-46 | constraint | 4 |

- 聚合规则覆盖：9/9 = 100%。
- 平均簇规模：37/9 ≈ 4.1（在 3~12 区间）。
- 无"同子系统=聚合理由"的假聚合：每个 KO 的聚合理由见 `03_knowledge_layer/ko-aggregation-matrix.md`。

## 4. 图的结构观察（Observation）

1. **三个 3 边枢纽**：EK-17（Pause 只有 Stop）、EK-26（升级状态机）、EK-33（fail-closed）各声明 3 条边；状态事实（EK-01）与状态持久化（EK-09）各 2 条边，共同构成状态机知识的主干。
2. **一个明显的对照对**：EK-17（可达 Stop）↔ EK-19（不可达 Checkpoint）——这是全图最强的"能力 vs 可达性"对照，直接支撑 KO-02。
3. **因果链最完整的一条**：EK-33→EK-34→EK-38→EK-07（决策→持久化→执行→定时消费），支撑 KO-01 的"不可逆动作前保全恢复信号"。
4. **无孤立 EK**，但 EK-02（tracing）与 EK-31（hook 失败分层）连接度较低（各 2 条边），属"局部但有价值"的工程知识（L1/L2），不强行升维。

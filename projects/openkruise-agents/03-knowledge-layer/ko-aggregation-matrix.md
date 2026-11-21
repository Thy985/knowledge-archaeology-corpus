# KO Aggregation Matrix（R1–R4 聚合规则与簇成立五条件）

> 规则：每个 KO 必须声明聚合规则 + 簇内 EK + 边类型；禁止"都属于某子系统"式假聚合。

| KO | 规则 | 簇内 EK | 主导边 | 为什么不是"分类" | 簇成立五条件 |
|---|---|---|---|---|---|
| KO-01 | R2 因果链 | EK-33 → EK-34 → EK-38 → EK-07（+EK-01/EK-37 约束） | causal, constraint | 不是"暂停模块"清单，而是"决策→持久化→执行→定时消费"的完整机制链 | 内聚：同一恢复信息链；跨实例：pause/upgrade/quota 三处；解释范围：从"暂停"扩到"任何不可逆动作"；可命名：Persist-before-irreversible；可回溯：每条 EK 有 E 编号 |
| KO-02 | R1 机制簇 | EK-17, EK-18, EK-19, EK-27 | mechanism, contrast | 不是"Pause/Upgrade 模块"，而是"类型声明 vs 调用可达"的同一机制 | 内聚：都在判断"策略是否进入控制流"；跨实例：pause/upgrade/CHANGELOG 三处；解释范围：能力审计；可命名：Reachability-first；可回溯：grep 证据 |
| KO-03 | R1 机制簇 | EK-03, EK-04, EK-09, EK-29, EK-45 | mechanism | 不是"控制器模块"，而是"意图/观察不同步"的同一处理机制 | 内聚：都不信任本地旧状态；跨实例：缓存、期望、stale Pod、配额；解释范围：分布式控制器；可命名：Intent-Observation；可回溯 |
| KO-04 | R2 因果链 | EK-13 → EK-14 → EK-21 → EK-20 → EK-23 → EK-12 | causal, constraint | 不是"Checkpoint 模块"，而是"快照→校验→重建→初始化"的完整恢复链 | 内聚：恢复保真度；跨实例：pause checkpoint / upgrade checkpoint / E2B clone；解释范围：快照系统；可命名：Reconstructable Snapshot；可回溯 |
| KO-05 | R3 不变量簇 | EK-15, EK-16, EK-31, EK-24 | constraint, contrast | 不是"hook 模块"，而是"不可假定只执行一次"这一不变量 | 内聚：重复执行安全；跨实例：PrePause/PreUpgrade/Resume 收尾；解释范围：分布式 hook；可命名：At-least-once + idempotent；可回溯 |
| KO-06 | R3 不变量簇 | EK-33, EK-34, EK-35, EK-36 | constraint | 不是"AutoPause 模块"，而是"不可判定→不动作"这一不变量 | 内聚：fail-closed；跨实例：ProbeValid、缺字段、负阈值、regex、condition；解释范围：自动化默认；可命名：Fail-closed automation；可回溯 |
| KO-07 | R4 主题簇 | EK-39, EK-40, EK-41, EK-42 | causal, constraint | 不是"Autoscaler 模块"，而是"稳定性四维"的主题覆盖 | 内聚：控制回路稳定；跨实例：cron/水位/死区/cooldown；解释范围：反馈控制；可命名：Delay-aware feedback control；可回溯 |
| KO-08 | R1 机制簇 | EK-43, EK-44, EK-12, EK-38 | mechanism, contrast | 不是"Gateway 模块"，而是"恢复即重入生命周期"的同一机制 | 内聚：重协商未来定时器；跨实例：暂停改写 vs 唤醒改写、runtime 重置；解释范围：事件驱动恢复；可命名：Resume = re-enter lifecycle；可回溯 |
| KO-09 | R3 不变量簇 | EK-05, EK-06, EK-25, EK-46 | constraint | 不是"架构文档"，而是"边界必须显式且默认安全"的不变量 | 内聚：边界强度；跨实例：phase switch/终态/升级默认/依赖规则；解释范围：架构治理；可命名：Checkable boundaries；可回溯（但采纳未验证） |

## 覆盖率与质量门

- 聚合规则覆盖率：9 / 9 = **100%**。
- KO 平均簇规模：37 / 9 ≈ **4.1**（3~12 区间）。
- 无假聚合：9 条中无一条的聚合理由是"同属某子系统"。
- 每条 KO 均满足五条件；KO-09 的"解释范围扩大"仅在单项目内成立，已在正文标注 L3、不升 L4。

## 反向检查：哪些 EK 未进入任何 KO？

46 条 EK 中进入 KO 的并集：
EK-01,03,04,05,06,07,09,12,13,14,15,16,17,18,19,20,21,23,24,25,27,29,31,33,34,35,36,37,38,39,40,41,42,43,44,45,46

未进入任何 KO 的 EK：**EK-02（tracing）、EK-08（finalizer）、EK-10（镜像归一化）、EK-11（Resumed 无条件）、EK-22（label 双代兼容）、EK-26（升级状态机细节）、EK-28（CheckpointRestore 只保 filesystem）、EK-30（trigger 注解）、EK-32（legacy hash 兼容）** = 9 条。

处理方式（符合"孤立 EK 不能删除"）：
- 这些 EK 保留在 `02_engineering_knowledge`，作为 L1/L2 推理原材料；
- 它们不是"孤立的图节点"（均有 ≥1 条边，见 EK Graph），只是未进入某个 KO 簇；
- 不强行凑进 KO：例如 EK-02 的 tracing 属可观测性局部知识，升维会过度抽象（`06_validation/abstraction` 记录）。

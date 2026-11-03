# 05 Candidates — durare（未验证假设）

> 严格保持 Hypothesis 标记，不冒充已验证知识。每条标注：类型 / 当前证据 / 缺失证据 / 验证路径。

---

## C-01 显式 context vs task-local 的取舍是否值得 API 代价（HYPOTHESIS）
- **type**: L3 模式假设 / 设计取舍
- **hypothesis**: 显式 `DurableContext` 参数（拒绝会丢 checkpoint 的放置）比官方 Rust SDK 的 task-local（容忍无 checkpoint 运行）更安全，代价是 API 侵入（workflow 必须显式接收 context；borrowed `&DurableContext` 原型需迁移 94 files 后被推迟）。
- **当前证据**: docs/design/explicit-context.md 的对比表（对 upstream commit 7937413 验证）；运行时 check 已捕获错误（"the runtime check has caught mistakes"）；borrowed 原型评估记录（94 files）
- **缺失证据**: 无真实用户代码迁移数据；无长期维护视角（哪个 API 更防错）
- **验证路径**: 观测社区真实使用中 DurableCallOutsideExecution 等错误的触发频率；或对照官方 SDK 的无 checkpoint 运行事故率
- **status**: Hypothesis（文档 + 实现双源，但"更安全"是权衡判断）

## C-02 AdminServer 无认证 by design 是否是正确的默认（HYPOTHESIS）
- **type**: 安全边界假设
- **hypothesis**: 无认证 AdminServer（含 workflow-recovery / global-timeout / garbage-collect 端点）依赖网络层防护是合理默认，因为与其他 DBOS SDK 对齐 + 共享工具可用。
- **当前证据**: security.rs "Network exposure" 表 + "unauthenticated by design, matching the DBOS SDKs"
- **缺失证据**: 无端到端安全评估；无对错误绑定（误绑公网）的事故记录
- **验证路径**: 观察真实部署的网络暴露面实践；或对照 DBOS 官方是否最终加了认证层
- **status**: Hypothesis（这是明显安全敏感的设计选择，标记待人工评审）

## C-03 portable_json 的 key 顺序稳定性跨 serde 版本是否长期成立（HYPOTHESIS）
- **type**: 跨项目互操作假设
- **hypothesis**: serde_json `preserve_order` 保证的 struct 字段顺序 = 字节序，跨 serde/serde_json 版本升级时不会悄悄改变，因此 golden strings 能长期防漂移。
- **当前证据**: Cargo.toml preserve_order 依赖注释；interop.rs golden strings
- **缺失证据**: 无 serde_json 版本升级回归测试记录；golden strings 若源码 refactor 会静默失效（interop.rs 自己承认）
- **验证路径**: 升级 serde_json 主版本后跑 interop.rs；或对照其他 SDK 是否同断言
- **status**: Hypothesis

## C-04 borrowed &DurableContext 原型推迟后是否仍会回归（HYPOTHESIS）
- **type**: 路线图假设
- **hypothesis**: borrowed context（`&DurableContext`）能消除 clone 语义的认知负担并支持非 'static 参数，因 94 files 迁移成本被推迟；运行时 check 已能捕获错误，故推迟无安全缺口。
- **当前证据**: explicit-context.md "deferred" 记录 + 运行时 check 存在
- **缺失证据**: 无该原型的实现代码/测试（仅文档记录）
- **验证路径**: 未来版本若实现 borrowed API，对比 clone 语义下被 check 捕获的错误是否仍可发生
- **status**: Hypothesis

## C-05 MAX_RECOVERY_ATTEMPTS_EXCEEDED 死信恢复的人工负担是否被低估（HYPOTHESIS）
- **type**: 运维假设
- **hypothesis**: 恢复超限 park 为死信后需显式 resume（"must be explicitly resumed"），在真实故障风暴中该人工步骤可能成为可用性瓶颈。
- **当前证据**: provider.rs 状态常量 + recovery 计数语义（CHANGELOG 0.5.0 "recovery_attempts"）
- **缺失证据**: 无真实故障演练数据；无 resume 操作的可观测性测试
- **验证路径**: 故障注入实验（failpoints）测多轮恢复风暴下死信率与恢复路径
- **status**: Hypothesis

## C-06 跨项目候选：checkpoint-per-call 系统的 position 分配时机（CROSS-PROJECT）
- **type**: 跨项目模式假设
- **hypothesis**: 任何把"每次调用一个持久化记录"作为重放基础的系统（durable execution、event sourcing、outbox 变体），记录位置必须在调用**构造/入队时**分配而非执行时——durare 在 select! 乱序上验证过这个教训，官方 Rust SDK 用容忍策略，其余生态待对照。
- **当前证据**: durare（KO-01，S4 测试复现）；官方 Rust SDK 的容忍策略（contrast）
- **缺失证据**: 其他 durable 引擎（Temporal/Restate/已考古 dapr-agents）的 position 语义是否同此
- **验证路径**: 对照已考古 corpus 中 Temporal/Restate 的 step 历史模型；或审计 event-sourcing 系统的 event id 分配时机
- **status**: Cross-project Hypothesis（进 Candidates，不写成 Principle）

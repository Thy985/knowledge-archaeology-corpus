# 03 Knowledge Layer — SemanticRun（ARCH-2026-11-05-001）

> Generalized Knowledge 层（L3/L4/L5 窄尖顶）。8 个 KO 全部声明 `aggregation_rule`（R1-R4 + 簇内 EK + 边类型），
> 每个 KO 满足簇成立五条件（内聚性/跨实例性/解释范围扩大/可命名/可回溯）。L3+ 升维带反例攻击（见 06）。

## KO 清单

### KO-01 同步持久化屏障：checkpoint 触发链（R2 因果链簇）
- **claim**：agent 每步执行后，经过"完成→分类→入队→drain→落库"的因果链，任何一步（含异常退出）的 checkpoint 都同步落库后才返回——"执行边界即持久化边界"。
- **aggregation_rule**：R2 因果链簇；链 EK-01（step 结束）→EK-09（分类触发）→EK-15（入队）→EK-02（SQLite 落库）；边类型 causal ×4
- **derivation.facts**：EK-01, EK-09, EK-15, EK-02
- **abstraction**: L3 ｜ **value**: A ｜ **epistemic**: Validated Pattern（S4）
- **同类对照**：Temporal 的 workflow 步骤持久化屏障；DBOS 的 transactional step 边界——同一"状态变更必须原子可见"主题
- **反例**：见 06 Counterexample（异步路径仍需 drain 兜底；memory_mutated 仅在显式 remember 时触发）

### KO-02 哈希承诺机制：恢复面全部哈希化（R1 机制簇）
- **claim**：tool result / outbound payload / tool schema / file tree / model id 全部在提交时哈希承诺（content_hash/schema_hash/merkle/fingerprint），恢复时以哈希对比判分歧——承诺-验证模式。
- **aggregation_rule**：R1 机制簇；mechanism 边连通 EK-03（hash_exclude 规范化）、EK-04（outbound 哈希）、EK-06（矩阵机械 diff）、EK-17（model hash）、EK-18（outbound hash 映射）；跨 checkpoint/hashing、kernel/ledger、kernel/divergence_matrix、kernel/artifact_diff、models 多子系统
- **derivation.facts**：EK-03, EK-04, EK-06, EK-17, EK-18
- **abstraction**: L3 ｜ **value**: A ｜ **epistemic**: Validated Pattern（S4）
- **同类对照**：git 对象哈希 / merkle tree；Temporal 事件序列号；DBOS 函数版本哈希——"指纹化提交物"通用模式
- **反例**：见 06（hash_exclude 可被误用掩盖真实漂移；outbound 哈希只覆盖字面 payload 不覆盖行为等价）

### KO-03 外部副作用 fail-closed 不变量（R3 不变量簇）
- **claim**：SemanticRun 的硬不变量是"重放外部副作用前必须证明它与已提交记录一致"——证据不足即拒绝（fail-closed），宁可 abort 不可重复执行。
- **aggregation_rule**：R3 不变量簇；constraint 边汇聚 EK-04（ACRFence 门）、EK-18（outbound→FailFast）、EK-13（PolicyAbort 强制中断）；同一不变量在 ledger（数据层）与 step_context（执行层）同时实现
- **derivation.facts**：EK-04, EK-18, EK-13
- **abstraction**: L3 ｜ **value**: A ｜ **epistemic**: Validated Pattern（S4）
- **同类对照**：durare 的 "verify before act" 哈希验证（11-04 考古）；Temporal activity idempotency keys；Kubernetes 的 at-most-once 语义
- **反例**：见 06（NO_PRIOR_RECORD 放行首重放；read_only 分类错误绕过门）

### KO-04 分歧-策略-执行闭环（R4 主题簇）
- **claim**：恢复安全不是"检测到漂移就报错"，而是完整闭环：机械检测分歧（DivergenceMatrix）→ 按显式映射路由（PolicyMapping）→ 策略执行（abort/rollback/revalidate/halt）→ 审计留痕（audit events）——检测与处置分离、处置策略全部显式声明。
- **aggregation_rule**：R4 主题簇；EK-06（检测）、EK-07（映射）、EK-08（判读语义）、EK-14（行为漂移直通）、EK-20（host 注入）覆盖"分歧处理"主题的互补维度；subsystem+mechanism+causal 混合边
- **derivation.facts**：EK-06, EK-07, EK-08, EK-14, EK-20
- **abstraction**: L3 ｜ **value**: A ｜ **epistemic**: Validated Pattern（S4）
- **同类对照**：Temporal 重试策略；LangGraph checkpointer + interruption；Kubernetes admission webhook（检测→策略→执行）
- **反例**：见 06（未注册 hook → halt_for_human 兜底；多 flag 并发路由顺序）

### KO-05 人工审批门生命周期（R2 因果链簇）
- **claim**：敏感动作走"请求→WAITING_APPROVAL 持久化→人类 approve/reject→状态迁移+checkpoint→恢复保持等待"的完整因果链；审批门跨进程/跨重启依然有效。
- **aggregation_rule**：R2 因果链簇；EK-12（request_approval→save_approval→checkpoint）→EK-08（resume 时 approval 判读）→EK-01（状态变更即持久化）；causal 边
- **derivation.facts**：EK-12, EK-08, EK-01
- **abstraction**: L3 ｜ **value**: A ｜ **epistemic**: Validated Pattern（S4）
- **同类对照**：Kubernetes 资源审批 / GitHub required reviews / 银行双人复核——"高影响动作必须人类显式授权"通用模式
- **反例**：见 06（approve/reject 无权限校验——任何调用者都能批；ApprovalState 无签名/身份字段）

### KO-06 checkpoint 稀疏性不变量（R3 不变量簇）
- **claim**：持久化成本与恢复完整性之间的不变量是"只对 recovery-relevant 动作 checkpoint"——只读轮零 checkpoint、副作用轮必 checkpoint、整体密度 <50%；稀疏不是缺陷，是设计。
- **aggregation_rule**：R3 不变量簇；constraint 边 EK-09（sparsity 规则）约束 EK-05（只读分类是其前置）；EK-01 依赖该边界成立
- **derivation.facts**：EK-09, EK-05, EK-01
- **abstraction**: L3 ｜ **value**: B ｜ **epistemic**: Validated Pattern（S4）
- **同类对照**：git 稀疏 checkout / 增量备份；Temporal 的 continue-as-new（控制历史增长）
- **反例**：见 06（只读分类错误 → 恢复点丢失；read_only 工具实际有副作用时）

### KO-07 恢复语义三件套：cursor 跳过 + 深拷贝 hydrate + 分歧矩阵（R4 主题簇）
- **claim**：语义恢复 = ①plan cursor 跳过已完工步骤（不重做）②checkpoint state 深拷贝 + hydrate 句柄局部态（不共享可变状态）③分歧矩阵判定"完成的是否仍然为真"（不盲信）——三件套缺一不可。
- **aggregation_rule**：R4 主题簇；EK-10（cursor）、EK-11（green 回滚）、EK-20（host 注入 artifacts）、EK-06（矩阵）覆盖"恢复语义"主题的互补维度
- **derivation.facts**：EK-10, EK-11, EK-20, EK-06
- **abstraction**: L3 ｜ **value**: A ｜ **epistemic**: Validated Pattern（S4）
- **同类对照**：Temporal 的 replay + deterministic workflow；DBOS 的 resume from last committed；langgraph checkpointers——恢复语义三流派
- **反例**：见 06（artifact_diff 只 diff 已提交的 tool 结果，未提交步骤完全丢失；enforce_policies=False 时全跳过）

### KO-08 显式声明优于推断（R1 机制簇）
- **claim**：本项目的认知一致性原则——一切涉及"判断"的地方都用显式声明替代自动推断：只读分类用显式列表、行为漂移用 host 显式标记、策略用显式映射+注册、恢复事实用 host 注入 artifacts。kernel 永不猜测。
- **aggregation_rule**：R1 机制簇；mechanism 边连通 EK-05（显式只读列表）、EK-14（显式行为漂移）、EK-07（显式策略映射）、EK-20（显式 artifacts 注入）；跨 kernel/skip_rules、policies、models 三个独立子系统
- **derivation.facts**：EK-05, EK-14, EK-07, EK-20
- **abstraction**: L3 ｜ **value**: A ｜ **epistemic**: Validated Pattern（S4）
- **同类对照**：DeltaBox 显式规则；Temporal 的确定性要求；Kubernetes 的 declared state（vs controller 推断）
- **反例**：见 06（classify_action 对未知工具默认 recovery-relevant=过度保守；NO_PRIOR_RECORD 放行首重放）

## L4 / L5 升维（仅经 Abstraction Promotion Gate 独立判定后保留）

### CM-01 语义恢复认知模型（L4，Cross-project validation pending）
- **claim**：**"重放保证'发生过'，语义恢复保证'仍然为真'"**——事件重放（durare/Temporal）验证的是执行历史的一致性；语义恢复（SemanticRun）验证的是"完成的工作在当前世界是否仍成立"。两者解决不同的失效面：前者防时序/幂等错乱，后者防环境漂移（工具/模型/文件/外部状态变化）。
- **aggregation_rule**：L4 由 KO-02/KO-04/KO-07 的 R1/R4 簇聚合 + durare（11-04）cross-project contrast 证据
- **promotion_arguments**：解释范围从"SemanticRun 如何恢复"扩大到"两类 durable agent 系统的设计选择分野"；新增对比维度（durare 的 verify-before-act vs SemanticRun 的 diff-on-resume）
- **epistemic**: Cognitive Model（**Cross-project validation pending**——仅 SemanticRun + durare 两项目证据，不宣称普适）
- **反例**：见 06（Infinity 的 stackless 挂起恢复属第三种，其"恢复"既非重放也非语义 diff——CM-01 未覆盖该分支，需标注边界）

### CM-02 机械对比确定分歧，策略/人工处理不确定性（L4，Cross-project validation pending）
- **claim**：在 agent 运行时中，把"判断"分层——能用确定性机制（哈希/字符串/布尔）判定的分歧绝不交给模型推断；模型/人类只处理确定性机制无法覆盖的剩余不确定性（行为漂移、语义等价、业务意图）。
- **aggregation_rule**：由 KO-02（哈希机制）+ KO-04（闭环）+ KO-08（显式声明）R1/R4 簇聚合
- **promotion_arguments**：解释范围从本项目"no LLM guessing"扩展到通用 agent 系统设计原则（Temporal 的确定性 workflow 同源）
- **epistemic**: Cognitive Model（**Cross-project validation pending**）
- **反例**：见 06（hash_exclude 的"什么算 ephemeral"本身就是语义判断，被移交给调用方而非机器——边界仍需人工定义）

### M-01 恢复语义设计五准则（L5 方法论，本项目验证）
- **claim**（可操作准则）：
  1. 恢复时由宿主注入"当前世界"事实（artifacts），环境不做世界探测
  2. 一切判分歧用确定性机制（哈希/布尔），行为级判断显式标记
  3. 外部副作用重放前必须证明一致，否则 fail-closed
  4. 已完成工作用 cursor 跳过，不重做；状态用深拷贝隔离
  5. 策略全部显式声明（flag→hook 映射），可审计、可替换
- **epistemic**: Methodology（基于本项目 S4 验证；跨项目有效性待验证）
- **scope**: 适用于"长运行 agent 的崩溃/漂移恢复"场景；不适用于纯分布式事务（无 agent 语义）

## 三层配比核对

| 层 | 目标 | 实际 | 状态 |
|---|---|---|---|
| Project Layer | 1 地图 | 01 文件 | ✅ |
| Engineering Knowledge | 40~60（小项目按比例缩放） | 22 EK（links 100%） | ✅ |
| Generalized KO | 7~12 | 8 KO（aggregation_rule 100%，簇规模 3~5） | ✅ |
| Candidates | 未验证假设 | 6 条（见 05） | ✅ |
| L4/L5 | 少量且 cross-project pending | 2 CM + 1 M（均标 pending） | ✅ |

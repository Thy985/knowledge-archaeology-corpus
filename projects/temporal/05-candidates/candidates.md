# 05 · Candidates（未验证假设 / 跨项目候选）— Temporal（ARCH-2026-11-02-001）

> 铁律：不能确认的内容留在候选，标 Hypothesis；Cross-project Candidate 不写成已验证 Principle。

## C-01 · Chasm 将成为 Workflow 的替代（Hypothesis）

- **内容**：Chasm 把 Workflow 重新实现为一个 ASM（chasm/lib/workflow），且 Workflow 保留独立 MutableState/HSM 路径 → 推测 Chasm 最终统一替换 Workflow 执行路径。
- **当前证据**：chasm/lib/workflow/（library.go/workflow.go）+ EnableChasm 等开关 + mutable_state 同时维护 HSM 与 ChasmTree。
- **缺失证据**：仓库中无"弃用 Workflow"声明；双路径仍并存。
- **验证路径**：跟踪未来版本 Chasm 开关默认值变化 + workflow 路径代码删除。

## C-02 · Speculative Workflow Task 与 Transient 是同一机制的两个视角（Hypothesis）

- **内容**：speculative WT（先执行后落事件，优化）与 transient WT（复制中间态，容错）在状态机里都表现为"未落 scheduled event 的任务"→ 推测底层可统一为"虚拟任务"抽象。
- **当前证据**：workflow_task_state_machine.go 中两者都走"尝试从 mutable state 查找 + 处理未持久化任务"路径（L169-200、L281-393）。
- **缺失证据**：未发现统一抽象类型；两者语义不同（性能优化 vs 容错）。
- **验证路径**：读 SpeculativeWorkflowTask 相关 issue/PR 演进。

## C-03 · NDC 复制批处理的 attempt count 重置是"复制时序脆弱点"（Hypothesis）

- **内容**：复制中 1st batch 到、2nd batch 未到时靠 attempt count 重置区分 transient → 推测复杂 failover 场景下 attempt count 重置可能掩盖真实状态。
- **当前证据**：`workflow_task_state_machine.go` L169-200 注释明确描述该场景与处理。
- **缺失证据**：无真实故障案例（浅克隆无 issue 数据）。
- **验证路径**：查 Temporal issue tracker 中 replication 相关 bug。

## C-04 · 跨项目：Chasm Tree 与 Restate journal 的"状态树 vs 步骤日志"对照（Cross-project Candidate）

- **内容**：Restate 持久化"执行步骤日志"（journal 每步），Chasm 持久化"状态树"（Node 值+Transition）→ 推测状态树派在**读路径**（查询当前状态）优于步骤日志派，但在**审计路径**（为什么走到这）弱于步骤日志派。
- **当前证据**：Chasm Tree/Node 持久化（EK-08）+ Restate 考古产物（ARCH-2026-11-01-001，journal 步骤持久化）。
- **缺失证据**：无性能基准对比；两系统语义不同（Chasm 仍是 Temporal 全家桶内，Restate 独立运行时）。
- **验证路径**：Benchmark 对照（需运行环境，S5 上限外）。

## C-05 · Worker Versioning 的 deployment 模型与 Kubernetes 滚动更新同构（Cross-project Candidate）

- **内容**：worker 版本化（build ID→deployment→versioning SA 路由）与 K8s Deployment 滚动更新（replicaset→pod 标签路由）在"版本路由+新旧并存+流量迁移"语义上同构 → 推测可抽象为通用"版本化执行单元路由"模式。
- **当前证据**：common/worker_versioning/（deployment 概念、Pinned/Assigned/Versioned BuildId SA、MakeDirectiveForWorkflowTask）。
- **缺失证据**：未在仓库中找到与 K8s 对照的文档。
- **验证路径**：跨项目模式提炼（需 K8s 考古或已有知识库条目）。

## C-06 · Nexus Callback Token 与 Webhook 签名是同族方案（Cross-project Candidate）

- **内容**：Nexus 回调用 callback token 鉴权（EK-35）→ 推测与 Stripe webhook signature、GitHub webhook secret 同族（"回调携带凭据，接收方验签"）。
- **当前证据**：common/nexus/callback_token.go（token 编码/校验）。
- **缺失证据**：未对比 token 格式与标准签名方案。
- **验证路径**：跨项目模式对照（低优先级）。

---
## Candidates 状态声明

- C-01/C-02/C-03：**Hypothesis**（单项目内，缺证据，需未来验证）
- C-04/C-05/C-06：**Cross-project Candidate**（跨项目假说，明确标 `Cross-project validation pending`）

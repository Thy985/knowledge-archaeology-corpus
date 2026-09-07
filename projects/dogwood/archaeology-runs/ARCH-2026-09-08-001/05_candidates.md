# 05 · Candidates — Dogwood（未验证假说与跨项目候选）

> 每条标注：假设状态 / 当前证据 / 缺失证据 / 验证路径。Hypothesis 不冒充 Fact。

## C-01（Hypothesis · 跨项目候选）"编译到既有策略语言"可推广到所有治理扩展
- **内容**：Dogwood 把时间语义 lower 进 Cedar 而非新造运行时——猜想是治理扩展采纳的通用模式（复用生态、降低迁移成本）。
- **当前证据**：Dogwood lowering 链（单项目，S3）。
- **缺失证据**：第二实例（如某工具把 guardrail 编译到 OPA/Rego 或 Cedar）。
- **验证路径**：检索 OPA/Cedar 生态中"扩展语言 lower 到基础语言"的实现。

## C-02（Hypothesis）"决策点 vs 历史点"事件分类是 agent 治理的通用 schema
- **内容**：事件分"触发判定的事件"与"仅被回看的事件"——猜想任何基于事件历史的治理系统都需要此二分。
- **当前证据**：Dogwood .dwschema（单项目）。
- **缺失证据**：omnigent/其它 guardrail 是否有等价分类。
- **验证路径**：对照 omnigent events、MemGraphRAG 事件流。

## C-03（Hypothesis）参考实现的"诚实限制清单"与生产实现成对出现
- **内容**：Dogwood（README 限制 6 条）与 omnigent（OBSERVABILITY 审计）都主动披露生产缺口——猜想高质量参考实现有"边界文档化"惯例。
- **当前证据**：两项目（S3/S2）——**已是跨项目弱印证**，但样本 n=2。
- **缺失证据**：第三实例（skillfortify/rampart 是否同）。
- **验证路径**：检查 corpus 已考古项目 README 尾部。

## C-04（Tentative Pattern）"契约/实现分层写入测试"可能成为 agent 治理语言的规范
- **内容**：fail_closed 把"契约 UNDEFINED BEHAVIOR vs 实现 deny-on-error"写进测试并声明"策略不得依赖任一侧"——猜想是治理语言防误解的规范动作。
- **当前证据**：Dogwood fail_closed.rs（单项目强证据）。
- **缺失证据**：同类语言（Cedar 自身/OPA）是否同惯例。
- **验证路径**：查 Cedar 官方测试是否含契约/实现分层声明。

## C-05（Observation）Rhai 沙箱"文档说默认无限制、代码有 max_operations 后盾"的张力
- **内容**：README 说部署默认无 CPU/内存限制（需自行配置），但代码注释显示 max_operations 是"runaway-script backstop"——文档（生产指引）与代码（默认后盾）存在张力，需确认 max_operations 默认值是否生效。
- **当前证据**：README vs eval.rs 注释（S3，双方均为原文）。
- **缺失证据**：max_operations 的默认值/接线点。
- **验证路径**：精读 eval.rs 求值入口；后续 refresh 时核实。

## C-06（Scope-uncertain）pin 分区"重写 pass 而非存储分区"的实际隔离强度
- **内容**：pin 让策略"如同"按键分区，但存储未分区——多租户实际隔离强度依赖调用方实现；不同 principal 的时间条件是否可能交叉读取历史，文档未完全展开。
- **当前证据**：engine.rs PartitionKey + README 限制 #4（S3）。
- **缺失证据**：pin 重写 pass 的具体实现（PartitionKey 用法）。
- **验证路径**：精读 engine.rs 中 PartitionKey 参与求值的路径。

## C-07（Unresolved Contradiction）is_authorized 返回 Option<Response> 的 None 语义
- **内容**：is_authorized 返回 Option——None 表示"无适用策略"还是"非决策点"？与 Decision 的 deny 区分影响上层（fail-closed 还是 fail-open 依赖此语义）。
- **当前证据**：authorize/mod.rs 签名（S3）+ 04.1 推测"非决策点返回 None"。
- **缺失证据**：docstring 或测试对 None 语义的显式定义。
- **验证路径**：精读 is_authorized 实现与 authorize_request.rs 测试意图。

## C-08（Hypothesis）"agent 工作流 skill + 强制 CLI 门"可成为语言项目自治理模板
- **内容**：.claude/skills 生命周期 + 每阶段强制验证门——猜想适用于任何"以 agent 为主要用户"的开发者工具项目。
- **当前证据**：Dogwood AGENTS.md（单项目）。
- **缺失证据**：第二实例。
- **验证路径**：对照 omnigent AGENTS.md（也有 skill 但侧重贡献流程而非授权生命周期）。

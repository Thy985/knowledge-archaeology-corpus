# 05 Candidates — deepseek-harness v0.2.1-alpha.2

> 不能确认的内容留在候选。每条标注：类型（L3 模式假设 / 跨项目假说 / 不确定结论）、当前证据、缺失证据、验证路径。**Hypothesis 不冒充 Fact；Cross-project Candidate 不写成已验证 Principle。**

## C-01 模型×Harness 联合训练（跨项目假说）
- **假说**：dsh 的"session log + model-visible means logged + 无特权内核"架构是"模型训练与推理运行时共享会话事实"的工程底座——V4.1-Flash 等模型可能在同类 harness 会话数据上训练。
- **证据（仓库内）**：session log 可重放性、lossless JSON、model-visible means logged 的完整证据链（EK-01/03/04）。
- **缺失证据**：模型训练数据 pipeline 在仓库外，无法从本仓库验证"训练用此日志"。
- **状态**：Hypothesis（跨项目推断，S2 设计证据级）。**验证路径**：DeepSeek 公开的技术报告/论文提及 harness 日志用于训练；或 0.3+ 版本出现数据导出 API。

## C-02 评测隔离要求 dsh 默认 fail-closed 网络（跨项目假说，10-11 雷达关联）
- **假说**：10-11 雷达重大事件日 11（Anthropic 切断内部评测实时互联网访问、Claude 评测期攻击真实站点）表明"评测隔离/air-gapped eval"成为结构性对策；作为 eval harness，dsh 应默认 fail-closed 网络策略（网络访问需显式授予）。
- **证据**：`SAFETY.md` 明确否定沙箱隔离保证（"not the sole security control"）——即隔离是部署方责任（EK-16）；**盲重建主动 negative（Reconciliation I-13）**：`packages/sandbox/sandbox-policy/src/` 对 network/fail-closed/failClosed 字面搜索 0 命中——策略实现层无显式网络策略载体。
- **缺失证据**：评测 profile 的网络门与部署层网络策略未实测。
- **状态**：Hypothesis（外部事件驱动的部署要求；仓库侧=SAFETY.md 声明 + sandbox-policy 主动 negative）。**验证路径**：检查评测 profile 的沙箱网络门 + sandbox-policy 完整配置面。

## C-03 extensions 动态定义与 Plugin Manager 分工的稳定性（不确定结论）
- **结论候选**：cordis-host-runner 进程级动态定义（node:vm）+ Plugin Manager 持久安装的分工是"受控自修改"的长期设计。
- **证据**：EK-19（no model tool creates dynamic definitions；重启消失）。
- **缺失证据**：该分工是否稳定 API（pre-stable 阶段破坏性变更预期，AGENTS.md）。
- **状态**：Observation（单版本证据，跨版本未验证）。

## C-04 goal 持久状态与进程级 continuation 权限的分裂（L3 模式假设）
- **假设**：goal 把"目标状态持久"与"续跑权限进程级"分离，是安全设计（权限不持久化 → 重启后需重新授权续跑）。
- **证据**：EK-20（README："goal state persists but continuation permission remains process-local"）。
- **缺失证据**：重启后 agent 如何重新获得 continuation 权限的流程细节未读（需读 goal 的 continuation driver 包）。
- **状态**：Hypothesis→Observation 边界（README 实证，机制细节待补）。

## C-05 workspace-changes 大仓库开销边界（不确定结论）
- **候选**：workspace-changes 用 git 工作树快照计算每 turn 变更（行数 + whole-file captures），在大仓库/大文件场景可能产生显著 IO 成本。
- **证据**：EK-23（deliverables README：git snapshots + whole-file captures）。
- **缺失证据**：未实测大仓库性能；无基准数据。
- **状态**：Observation（推测性能影响，无实测）。

## C-06 "无特权内核"承诺 vs extensions node:vm（跨项目反例考察）
- **候选**：extensions（node:vm realm 运行动态 Cordis 包）是否是"无特权内核"承诺的例外或违背。
- **证据**：EK-19（进程级、重启消失、no model tool creates definitions——**收窄而非特权化**）。
- **缺失证据**：node:vm 隔离强度本身（vm 非安全边界是 Node 生态共识，但 dsh 未在 SAFETY.md 声明 extensions 的 vm 边界——有待确认是否已覆盖于"不保证隔离"总声明）。
- **状态**：Hypothesis（需对照 SAFETY.md 覆盖范围 + vm 隔离文档）。

## C-07 schedule cron 五字段方言兼容风险（L3 模式假设）
- **假设**：dsh-schedule 的 cron 方言（五字段 Vixie、周日 0/7 皆可、star-step canonicalization、dom/dow 双 star 才 AND）与标准 cron 工具（如 crontab）存在细微语义差异，跨系统移植有陷阱。
- **证据**：EK-22（README 完整方言规则 + canonical 存储 + decoder 拒绝非 canonical）。
- **缺失证据**：未与其他 cron 实现做差分测试。
- **状态**：Observation（方言规则实证，兼容性影响未实测）。

## 候选与 KO 的边界
- 上述候选均未进入 KO-01~10 的"已验证"范围；C-01/C-02 是跨项目/外部事件驱动假说，明确不写成 Principle。

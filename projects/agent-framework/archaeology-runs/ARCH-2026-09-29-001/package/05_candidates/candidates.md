# Candidates — 未验证假说（ARCH-2026-09-29-001）

> Hypothesis 严格标注；不冒充已验证知识；Cross-project 内容标 `Cross-project validation pending`。

## C-01 审批权威边界与 session 复用是通用模式
- type: Cross-project Hypothesis ｜ epistemic: Hypothesis
- 内容：审批是否权威取决于"会话归属是否可被调用者复用"（本项目测试证明），推测这是审批系统防"伪造审批来源"的通用设计（先例：银行审批 token 绑定会话）。
- 当前证据：test_framework_created_session_becomes_authoritative_when_reused_by_caller（S4）。
- 缺失证据：跨项目同类机制对照（仅本项目）。
- 验证路径：审计另一审批系统（如 tool-use 沙箱审批）的 session 归属语义。

## C-02 fresh_context 的上下文重置成本
- type: Hypothesis ｜ epistemic: Hypothesis
- 内容：fresh_context（session 快照/恢复）每次迭代序列化重建 session，推测大会话下有性能开销；项目未提供基准数据。
- 当前证据：_loop.py 快照/恢复实现（S3）。
- 缺失证据：性能基准/内存对比。
- 验证路径：benchmark fresh vs non-fresh 在大上下文下的延迟。

## C-03 quarantine 客户端的实际使用路径
- type: Hypothesis ｜ epistemic: Hypothesis
- 内容：set_quarantine_client / get_quarantine_client 是全局单例，推测多租户部署下会成共享状态风险点（全局状态污染）。
- 当前证据：security.py|3331-3364（全局函数，S3）。
- 缺失证据：多租户场景测试。
- 验证路径：检查 Hosting（foundry）如何隔离 quarantine client。

## C-04 Magentic 模式与 GroupChat 的重叠
- type: Cross-project Hypothesis ｜ epistemic: Hypothesis
- 内容：MagenticManager 的任务 ledger + 团队块机制与 GroupChat（LLM 选发言人）在功能上重叠（团队协作 → 谁说话/谁负责），推测未来会收敛或需要明确分工文档。
- 当前证据：_magentic.py 与 _group_chat.py 并存（S3）。
- 缺失证据：官方文档对两模式适用场景的权威划分。
- 验证路径：读 docs/ 的 orchestration 文档/FAQ。

## C-05 记忆注入的 prompt injection 防御全链路
- type: Hypothesis ｜ epistemic: Hypothesis
- 内容：Cosmos memory 的 user_summary untrusted 注入（EK-31）与 FIDES 标签（EK-21）是否协同（记忆内容是否也会被 LabelTracking 打标签）？推测记忆通道标签独立于 FIDES（两个独立防御面）。
- 当前证据：_context_provider.py 注释明言防注入路径（S3）；security.py 标签机制（S3）。
- 缺失证据：记忆内容是否经过 LabelTracking 的集成测试。
- 验证路径：搜索 memory + FIDES 集成测试。

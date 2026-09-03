# Candidates（候选知识 · 待验证假说）

> **v3 三层架构的候选层。** 未完成抽象、待验证模式、跨项目假设。保持 Hypothesis 标记，与已验证的 KO 严格区分。
> 每条标注：L3 模式假设 / 当前证据 / 缺失证据 / 验证路径。

---

## C-01：Agent 系统的"审批即学习"闭环模式（L3 假设）

- **假设**：Agent 系统中，审批结果本身可以作为策略学习的输入——每次审批都携带"是否应该固化为规则"的建议，形成"执行→审批→固化→未来免审"的自适应闭环。
- **当前证据**：EK-01（三态审批携带修正案）、EK-02（策略持久化热更新）、EK-18（Honor 模式门控）、KO-01（三态审批+修正案模式）。
- **缺失证据**：Codex 没有"审批拒绝率→自动调整策略"的反向闭环（只有正向固化，没有负向收紧）；缺少跨项目验证（其他 Agent 系统是否有类似机制）。
- **验证路径**：① 调研其他 Agent 系统（Claude Code、Cursor、Aider）是否有审批→策略固化机制 ② 分析 Codex 是否有负向反馈路径（拒绝后是否收紧规则）③ 若跨项目验证成立，可晋升为 L3 Pattern 甚至 L4 Model。
- **epistemic_status**: Hypothesis

---

## C-02：多层 config stack + overlay 的通用治理模式（L3 假设）

- **假设**：多层配置叠加（builtin → user → project → requirements overlay）+ 高层覆盖低层 + 可选 ignore 开关，是 Agent/工具系统的通用治理模式，可迁移到任何需要"出厂默认 + 用户自定义 + 企业强制"三级配置的系统。
- **当前证据**：EK-40（多层 config stack）、EK-43（ignore 开关）、EK-29（overlay 静默覆盖）、KO-02（分层策略叠加模型）。
- **缺失证据**：Codex 的 config stack 不仅用于 exec_policy，还用于其他配置（需验证多层 stack 是否是全局模式）；缺少冲突可见性机制（overlay 静默覆盖无日志）。
- **验证路径**：① 检查 Codex config crate 确认多层 stack 是否全局通用 ② 调研其他系统（VS Code settings、K8s ConfigMap、浏览器策略）的多层配置模式 ③ 若跨项目验证成立，可从 KO-02 中独立为 L3 Pattern。
- **epistemic_status**: Hypothesis

---

## C-03：上下文治理的"规则+trait+预算"三层模式跨项目可迁移性（L3 假设）

- **假设**：AGENTS.md 的"规则文档定原则 + trait 定接口 + 常量定预算"三层上下文治理模式，可迁移到任何 LLM 应用，不依赖 Codex 特定技术栈。
- **当前证据**：EK-08（50+ ContextualUserFragment）、EK-24（no history rewrite）、EK-36（>1K P0 审查）、EK-51（主上下文与 Guardian 预算分离）、KO-05（三层上下文治理模式）。
- **缺失证据**：缺少跨项目验证（其他 LLM 应用是否采用类似三层治理）；trait 是 Rust 特定机制，其他语言的对应物（interface/protocol）是否同样有效需验证。
- **验证路径**：① 调研其他 LLM 应用的上下文治理（LangChain、LlamaIndex、AutoGPT）② 验证非 Rust 语言的 interface/protocol 是否能实现同等类型安全 ③ 若跨项目验证成立，KO-05 可考虑晋升 L4。
- **epistemic_status**: Hypothesis

---

## C-04：Guardian "用 AI 审批 AI"的可靠性边界（跨项目假设）

- **假设**：用另一个 AI 审批 AI 的命令时，审批器的可靠性受限于：① 独立 session 隔离 ② fail-closed 错误处理 ③ 熔断器防失控 ④ 上下文预算限注入面。这四个机制构成 AI 审批器的最小可靠性集合，缺少任何一个都会导致不可接受的风险。
- **当前证据**：EK-03（Guardian 整体设计）、EK-25（fail-closed）、EK-35（熔断器）、EK-22（上下文预算）、EK-50（不继承父 exec-policy）、KO-03（AI 审批器可靠性模型）。
- **缺失证据**：缺少"四个机制缺一不可"的反例验证（是否存在缺少某个机制但仍可靠的 AI 审批器）；缺少 Guardian 实际准确率/误拒率的运行时数据（Codex 没有 Guardian 准确率持续评估机制，KO-03 尝试 L5 时因此降级）。
- **验证路径**：① 调研其他 AI 审批器（如 GitHub Copilot 的代码审查、AWS CodeGuru）的可靠性机制 ② 分析是否存在缺少某个机制但通过其他方式补偿的案例 ③ 若验证"四机制最小集合"成立，KO-03 可晋升 L4→L5（方法论）。
- **epistemic_status**: Hypothesis

---

## C-05：网络规则子域名匹配语义（待确认边界）

- **假设**：Codex 网络规则的子域名匹配语义可能存在边界模糊——`allow example.com` 是否匹配 `api.example.com` 取决于 codex-execpolicy crate 的匹配实现，core 代码未显式说明。
- **当前证据**：EK-31（网络规则匹配语义边界）、KO-04 反例攻击 #2。
- **缺失证据**：codex-execpolicy crate 的匹配语义未在本次考古中展开；缺少测试用例验证子域名匹配行为。
- **验证路径**：① 阅读 codex-execpolicy crate 的网络规则匹配代码 ② 查找相关测试用例 ③ 确认匹配语义后更新 EK-31 和 KO-04。
- **epistemic_status**: Hypothesis（边界待确认）

---

## C-06：AGENTS.md 多层级膨胀治理的完整覆盖（待补全）

- **假设**：AGENTS.md 定义了多层级膨胀治理——crate 级（72-83 行）、模块级（49-53 行，500/800 LoC）、变更级（125-131 行，800/500 行）、API surface（87-89 行）。v1 审计发现 KO-06 只覆盖了 crate 级，遗漏了其他三个层级。v2/v3 考古未完全补全。
- **当前证据**：v1 independent-audit IF-09（多层级膨胀治理 PARTIAL 覆盖）。
- **缺失证据**：模块级、变更级、API surface 三个层级的具体规则内容和代码实现未在 v2/v3 考古中展开。
- **验证路径**：① 阅读 AGENTS.md 对应行号的完整规则 ② 查找是否有 CI 检查或工具强制执行这些限制 ③ 若内容充实，可新增为 Engineering Knowledge 条目（重要配置类）。
- **epistemic_status**: Hypothesis（覆盖待补全）

---

## 候选汇总

| ID | 类型 | 层级假设 | 关联 KO/EK | 状态 |
|----|------|---------|-----------|------|
| C-01 | 审批即学习闭环 | L3 Pattern | KO-01, EK-01/02/18 | 待跨项目验证 |
| C-02 | 多层 config stack 治理 | L3 Pattern | KO-02, EK-40/43/29 | 待全局通用性验证 |
| C-03 | 三层上下文治理可迁移性 | L3→L4 | KO-05, EK-08/24/36/51 | 待跨项目验证 |
| C-04 | AI 审批器四机制最小集合 | L4→L5 | KO-03, EK-03/25/35/22/50 | 待反例验证 |
| C-05 | 网络规则子域名匹配 | 边界确认 | EK-31, KO-04 | 待代码确认 |
| C-06 | AGENTS.md 多层级膨胀治理 | 覆盖补全 | v1 IF-09 | 待内容展开 |

**Candidates 与 KO 的严格区分**：Candidates 保持 Hypothesis 状态，不进入 Generalized Knowledge 层的正式 KO 列表。验证通过后可晋升为 KO（需经过 Abstraction Promotion Gate）。

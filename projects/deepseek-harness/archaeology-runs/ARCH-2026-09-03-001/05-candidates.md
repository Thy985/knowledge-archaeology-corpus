# Candidates — deepseek-harness（未确认内容）

> Job: `ARCH-2026-09-03-001` | 不能确认的内容留在 Candidate；不升层、不写成知识。

---

## C-01 — 跨项目假说：日志即真相（model-visible means logged）
- **claim**：Agent runtime 以"只有进入不可变日志的才被系统承认"作为记忆/审计权威，可能是 Agent 系统的通用记忆原则。
- **status**: Hypothesis（本项目为 Principle，但**仅单项目证据**）
- **validation_path**: 与 codex（corpus 已有）、TeamMind 的会话模型对照；若 ≥2 独立项目采用同构原则 → 可升 Validated Pattern/L4。
- **关联**: KO-05 | **scope**: 跨项目 | **置信**: medium

## C-02 — 跨项目假说：Fail-Closed 权限族
- **claim**：审批/守卫/沙箱/凭证四层 fail-closed 叠层，可能是高安全 Agent 权限系统的通用模式。
- **status**: Tentative Pattern（本项目验证充分，跨项目未验证）
- **validation_path**: 对照 Codex（exec_policy deny-read→NoOverride）、OpenClaw 权限模型。
- **关联**: KO-03 | **置信**: medium-high

## C-03 — 跨项目假说：Capability Seam 三件套
- **claim**：Service Definition/Provider/Consumer 三件套 + 自动生成依赖图，是否适用于其它大型插件化框架。
- **status**: Hypothesis（本项目强证据，通用性未验证）
- **validation_path**: 对照 VS Code 扩展体系、LangChain 集成层、Eclipse 插件模型。
- **关联**: KO-02 | **置信**: medium

## C-04 — 边界不确定：Subagent 多 provider 委托
- **claim**：tool-subagent 通过单一 seam 委托给异构 provider（spawn/acp/claude-code/codex/dsh-sdk/fork）。
- **status**: Observation（实现存在，但"异构编排是否提升鲁棒性"未在运行中验证）
- **不确定性**: 各 provider 的错误语义/恢复一致性未统一验证；fork-in-process vs spawn 的安全边界不明确。
- **关联**: EK-21, EK-22

## C-05 — 边界不确定：Compaction 的语义保持
- **claim**：compaction 把历史替换为 summary 节点后，后续推理信息是否无损（shadow 语义是否保持因果完整）。
- **status**: Hypothesis（代码存在 shadow 语义，但信息损失度量未量化）
- **反例提示**: KO-01/KO-04 依赖"日志可重放"，而 compaction 是有损压缩 → **潜在 contradiction**（已记录，供 reconciler）。

## C-06 — 反例（Counterexample，削弱 KO-03 的绝对性）
- **claim**：`danger-full-access` 模式存在 → 沙箱不是无条件 fail-closed；`approval` 只覆盖"请求审批的动作"，非全部动作。
- **status**: Counterexample（边界化，不推翻）
- **处理**: KO-03 已条件化（"任意一层失败都不产生授权"适用于 confined 路径；danger-full-access 显式 opt-in 绕过）。
- **关联**: KO-03, EK-12, EK-10

## C-07 — 反例：错误归属仍可能失真（SEARCH_FAILED 教训）
- **claim**：即使有结构化错误族，broad 包装层仍会掩盖下层结构化错误（postmortem 0004 中 SEARCH_FAILED 掩盖 SANDBOX_UNAVAILABLE）。
- **status**: Validated counterexample（项目内已证实）
- **处理**: KO-07 已条件化——"结构化错误族"本身不保证归属正确，需要 status-gated 证据纪律。
- **关联**: KO-07, EK-15

## C-08 — 范围不确定：Turn/Step 模型的分层粒度
- **claim**：turn/step 双层（vs 单层/三层）是否为最优会话粒度。
- **status**: Hypothesis（本项目选择双层；跨项目无对照）
- **关联**: EK-01

## C-09 — 范围不确定：Cordis vendor 化的长期成本
- **claim**：vendor 化 Cordis（而非依赖上游）带来完全控制，但增加维护成本；长期是否值得未验证。
- **status**: Observation（决策记录存在，收益/成本未量化）
- **关联**: EK-24

---

## 未解决矛盾登记（供 Reconciler）
| 矛盾 | 双方 | 状态 |
|---|---|---|
| "日志可完整重放" vs "compaction 有损压缩" | KO-01/KO-05 vs EK-19 | CONDITIONAL：日志保留 compaction/start 与替换范围（可回溯），但 token 信息已损；重放=因果链可重建，非逐字完整 |
| "结构化错误可归属" vs "broad 包装层掩盖" | KO-07 vs EK-15 | CONDITIONAL：归属正确性依赖 status-gated 证据纪律，非结构本身 |
| "fail-closed 不变量" vs "danger-full-access 绕过" | KO-03 vs EK-12 | CONDITIONAL：fail-closed 约束 confined 路径；danger-full-access 是显式 opt-in 逃生门 |

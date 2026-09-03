# Job Manifest — ARCH-2026-09-04-001

```yaml
job_id: ARCH-2026-09-04-001
project: rampart
repository: https://github.com/microsoft/RAMPART.git
mode: initial
priority: S
source_entry: 03_expansion_queue/candidates/[cand]rampart-clarity-2026.md
selected_at: 2026-09-04 02:05 (Asia/Shanghai)
skill_version: 3.2
```

## Candidate Ranking Summary（每日最多 1 个深度项目）

排序维度：Knowledge Value / Agent-AI Engineering Relevance / Novelty / Growth / Recent Release / Corpus Connection / 是否已考古 / Refresh 需求 / Benchmark 学习价值。**不按 star 排名。**

| 排名 | 候选 | 等级 | 状态 | 理由（一句话） | 考古判定 |
|---|---|---|---|---|---|
| 1 | **Microsoft RAMPART** | S | NEW | Agent 安全 CI 测试框架（pytest/prompt-injection/概率测试/evaluator 组合），命中 G1 最高缺口，有真实可考古源码仓库 | ✅ **选中（initial）** |
| 2 | Agentic CLEAR（IBM） | A | NEW | 开源 Python 包，三级自动 agent 评估，与"验证编译器"同构；但比 RAMPART 少一个 S 级优先 | 排队（今日不选） |
| 3 | PI-Hunter 注入审计线 | A | NEW | arXiv 方法论文族，偏方法精读而非单一可考古仓库 | 排队 |
| 4 | Agent Harness/Control Plane 范式 | S | NEW | 概念/范式类（Omnigent+MS+harness survey），跨多对象，非单仓库深度考古 | 排队 |
| 5 | OWASP Agentic Security | S | NEW | 体系/治理文档类，非代码考古对象 | 排队 |
| 6 | Agent Memory 2026 | S | NEW | 跨多框架实现层全景，非单仓库 | 排队 |
| 7 | A2A 协议 v1.0 | S | NEW | 协议规范类，非单仓库代码考古 | 排队 |
| 8 | hermes-agent | B→A | NEW | 大框架、★ 高（不按 star），单日考古吃力 | 排队 |
| … | 其余（Evals 格局/AI 攻防/E2B/OTel/Browser use 等） | A/B | NEW | 均非 S 级优先或非单仓库 | 排队 |

## Selected Project

- **project**: RAMPART（Risk Assessment and Measurement Platform for Agentic Red Teaming）
- **repository**: https://github.com/microsoft/RAMPART.git（HEAD 125595cb2dc53ed81ac4da0f30a7042d6951e897）
- **companion**: Clarity（https://github.com/microsoft/Clarity.git，设计审查助手，仅关联提及，不在本 run 深度考古）

## Reason

- **Knowledge Value 极高**：S 级候选；把 AI 安全从"事后红队"变成"可嵌入 CI/CD 的 pytest 回归测试"——正是"把安全左移"的工程实证。
- **Agent/AI Engineering Relevance 极高**：Agent 安全测试框架，核心机制（cross-prompt injection 攻击策略、概率统计试验、可组合 evaluator、Python protocol 扩展点、CI gating）全是可提炼的 Engineering Knowledge。
- **Novelty 高**：2026-05-20 开源，发布不足 4 个月，MS AI Red Team 一线出品（Ram Shankar Siva Kumar 团队）。
- **Growth / Activity**：MS 维护，建立在持续更新的 PyRIT（0.13+）生态之上。
- **Recent Release Signal**：2026-05-20 发布 + MS AI Tour 2026 配套动手实验仓库（aitour26-LTG156）。
- **Connection to Existing Corpus**：与已考古的 deepseek-harness（Agent 运行时 harness）形成"运行时 ↔ 安全 CI"对照；连接 dsh-pentest / silver-shield。
- **已考古**：否（NEW）。
- **Refresh**：不适用（initial）。
- **Benchmark / learning value**：可给 dsh-pentest 填骨架的最小 CI 安全测试；对照 OWASP Agentic 体系覆盖。
- **不按 star 排名**：RAMPART 相对冷门，但价值排序第一。

## Constraints

- 只读：不修改目标项目 / KnowlegeMap / 生产 Skill
- 每天最多深度完成 1 个项目（本 run 仅 RAMPART）
- 不覆盖已有 Corpus 结果（deepseek-harness 为 projects/deepseek-harness/）

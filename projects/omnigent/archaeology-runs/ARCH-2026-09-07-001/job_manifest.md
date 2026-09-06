# Job Manifest — ARCH-2026-09-07-001

> 2026-09-07 · Daily Knowledge Archaeology Runtime（SOP cron 11659931249922）

## 候选排序（按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长 / 近期发布 / Corpus 连接 / 未考古 / Benchmark 价值，不按 star）

| # | 候选 | 等级 | 单仓库可考古 | Agent-AI 相关 | Corpus 连接 | Novelty/增长 | Benchmark 价值 | 排序理由 |
|---|------|------|-------------|--------------|-------------|--------------|----------------|----------|
| 1 | **Omnigent**（agent-harness-control-plane） | S | ✅ omnigent-ai/omnigent | 极高（meta-harness/控制面） | 最密集（harness/evolver 策略层/dsh 记忆/RAMPART 权限） | 极高（Databricks 2026-06 开源，路线图 GEPA/MemEx/RLM） | 高（五维模型/验证编译器理论对照 + 组合 Claude Code+Codex 实测） | S 级 + 单仓库 + 控制面主题把已有 corpus 串成链 |
| 2 | Mem0（agent-memory-2026） | S | ✅ mem0ai/mem0 | 高（记忆实现层） | 中（dsh-memory-evolve） | 高（v3.0 算法） | 中高（但候选卡为全景+基准批判，单仓库考古丢全景） | S 级但全景卡，单仓库考古收益打折 |
| 3 | OWASP Agentic Security | S | ⚠️ 文档/标准类多来源 | 高（安全治理） | 中（RAMPART/skillfortify） | 中（2026 持续更新） | 中（理论映射为主） | 非单仓库深度考古适合 |
| 4 | A2A Protocol v1 | S | ⚠️ spec 类 | 高（协议） | 中（evolver atp） | 中（2026-03 冻结） | 中（协议规范非代码实现） | 规范考古，非代码仓库模式 |
| 5 | E2B Sandbox（e2b-sandbox） | A | ✅ e2b-dev/E2B | 高（沙箱） | 中（RAMPART） | 中 | 中 | A 级，可后续轮次 |
| 6 | PI-Hunter（pi-hunter-injection-audit） | A | ✅ | 高（注入审计） | 中（skillfortify 安全面） | 中 | 中 | A 级，可后续轮次 |
| 7 | Agentic CLEAR（agentic-clear） | A | ✅ | 高（评测） | 中（evals） | 中 | 中 | A 级 |
| 8 | Webwright（webwright-2026） | B | ✅ | 中 | 低 | 中 | 中 | B 级 |
| 9 | Decentralized AI / Federated Agents | B | ⚠️ 标准/论文类 | 中 | 低 | 中（雷达最新） | 低 | B 级 + 非单仓库 |

## 选中项目

```yaml
job_id: ARCH-2026-09-07-001
project: omnigent
repository: https://github.com/omnigent-ai/omnigent.git
mode: initial
priority: A
reason:
  - S 级候选（agent-harness-control-plane 卡）唯一"单仓库可深度考古"选项
  - meta-harness / control plane 是 Agent 工程 2026 核心范式（组合/控制/协作三支柱）
  - 与 corpus 连接最密集：deepseek-harness（harness 面）+ evolver（策略/自进化）+ dsh-memory-evolve（记忆）+ RAMPART（权限）——控制面主题串起已有知识链
  - 用户五维模型/验证编译器/EP-002 理论的工程对照物（Benchmark 学习价值最高）
  - Databricks 官方开源（2026-06-16），增长信号强，路线图含 GEPA/MemEx/RLM
source_entry: 03_expansion_queue/candidates/[cand]agent-harness-control-plane.md
```

# Job Manifest — ARCH-2026-10-25-001

| 字段 | 值 |
|---|---|
| job_id | ARCH-2026-10-25-001 |
| date | 2026-10-25（Sun，Harness/治理 周 Day7 小结日） |
| project | dapr-agents |
| repository | https://github.com/dapr/dapr-agents.git |
| mode | initial |
| priority | P0 |

## 候选池排序（按 Knowledge Value / 相关性 / Novelty / 增长 / 连接，非 star）

| 排序 | 候选 | 级别 | 是否已考古 | 评分理由 | 决策 |
|---|---|---|---|---|---|
| 1 | **dapr-agents** | A | 否（corpus 无） | CNCF Dapr × NVIDIA 2026-10 新 launch；durable execution（每步持久化+崩溃恢复不重放）是 multi-agent 生产化基础设施范式；Runtime 周接 TeamMind 头号锚点（卡内明确）；与 dsh/MXC/strands-box 形成 harness→runtime 对照谱系；官方 Getting Started 10-14 上线可跑通 | **选中 initial P0** |
| 2 | personal-agent-protocol（Poppy） | A(新) | 否 | Sierra/Meta 牵头 OAuth agent-to-business 标准；但 v0.1 规范 10 月内交付、无源码仓库可考古（标准非代码库） | 延后（待规范+实现落地） |
| 3 | agent-formal-verification | A | 否 | 形式化验证方法论；但单一权威代码库不明，多论文/框架 | 延后 |
| 4 | agent-firewall-runtime-defense | A | 部分（aigis/guardian 已考古） | 品类多同名仓库无单一权威（10-24 已否决依据） | 不选 |
| 5 | agentic-attack-2026 | S | 否 | 攻击研究品类，非单一代码库，事件集群卡已精读 | 不选 |
| 6 | open-source-agentic-ci / testjack / tofu | B/A | 否 | 评测/CI 工具，领域价值低于 runtime 基础设施层 | 延后 |
| 7 | php2shell | A(新) | 否 | PHP 潜伏 CVE 攻击链分析，非代码库考古对象 | 不选 |

## 选中项目理由
1. **Knowledge Value**: durable execution + actor + 双 state store + AgentRunner = 多 agent 可靠执行的基础设施范式，与已考古项目（harness/沙箱/框架）完全不同抽象层，Benchmark 学习价值最高
2. **Agent-AI Engineering 相关性**: DurableAgent/AgentRunner 是 agent 原生 API；中断恢复"不重放已完成的 LLM/工具调用"是成本+正确性双维度关键机制
3. **Novelty + 增长信号**: 2026-10-14 Getting Started / 10-16 Dapr Foundation × NVIDIA launch 公告，pushed 2026-10-22（昨日），周内密集发布
4. **与已有 Corpus 连接**: TeamMind（Runtime 换域头号锚点，卡内明确）、与 deepseek-harness（approval/sandbox）同属 harness 谱系、与 MXC/strands-box（执行隔离）形成"隔离→分布式运行时"对照
5. **验证路径可复现**: quickstart（03_durable_agent_http.py 中断恢复 demo）可直接跑通，满足 validated 门槛

## source_entry
- KnowlegeMap: 03_expansion_queue/candidates/[cand]dapr-agents-distributed-runtime-2026.md（2026-10-20 兜底捕获，A 级）
- current_focus.md: 周日小结换域候选 Runtime 接 TeamMind → dapr-agents 为头号锚点

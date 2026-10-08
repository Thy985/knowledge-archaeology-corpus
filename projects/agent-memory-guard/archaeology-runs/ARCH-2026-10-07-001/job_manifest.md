# Job Manifest — ARCH-2026-10-07-001

## 0. 轮次信息
- 日期：2026-10-07（每日 Knowledge Archaeology Runtime SOP 轮）
- 雷达：KnowlegeMap `05_scan_logs/2026-10/2026-10-07_radar-watch_scan.md`（第 35 次运行，HEAD 152b3a3）
- 增量：3 条（agentic-attack 重大事件日 10 / agent-memory 基准分层+效用批判+KV 交叉 / owasp-agentic-security 35 天来首次补：AST10 v2 + Cheat Sheet + AIVSS v0.8 + ACS）

## 1. 候选排序（九维评估，非 star 排名）
| # | 候选 | 来源条目 | 知识价值 | Agent-AI 相关性 | Novelty | 增长信号 | corpus 连接 | 已考古? | 决策 |
|---|---|---|---|---|---|---|---|---|---|
| 1 | **OWASP Agent Memory Guard** | owasp-agentic-security 卡增量（ASI06 参考实现）+ agent-memory 卡（记忆安全线） | A（S 级卡官方参考实现，MITRE ATLAS 引用） | 极高（记忆完整性×安全中间件×MCP） | 高（corpus 无记忆安全对象） | **pushed 2026-10-06** | langgraph(secure_langgraph_memory)/owasp-agentic-skills/letta-code/mem0/aigis/opentelemetry | 否 | **✅ 选中（initial）** |
| 2 | vllm | local-edge-llm 卡（雷达 next-step 月度轮） | A | 高（推理基础设施） | 高 | 活跃 | dynamo（对照） | 否 | 备选——体量 ~93k★ 百万行，超出"每日 1 深度项目"合理范围 |
| 3 | DyadMem / BEAM | agent-memory 卡 10-07 增量 | B | 中（记忆基准） | 中 | 新发布 | mem0/letta-code | 否 | 剔除——GitHub 探测 404（论文信号，无代码仓库） |
| 4 | OWASP AST10 v2 | owasp-agentic-security 卡 10-07 增量 | B | 高 | 低 | 白皮书 v2 | owasp-agentic-skills | **是（09-26）** | 剔除——代表仓库 www-project-agentic-skills-top-10 已深度考古 |
| 5 | OWASP ASI/AIVSS/ACS | owasp-agentic-security 卡 10-07 增量 | A | 高 | 高 | 活跃 | silver-shield 对照 | 否 | 剔除——官网文档发布，4 个 GitHub 路径探测全 404，无独立代码仓库 |
| 6 | agentic-attack 事件（重大事件日 10） | agentic-attack 卡 10-07 增量 | A | 高 | 中 | 事件密集 | silver-shield/dsh-pentest | 部分（pyrit） | 剔除——事件/报告卡，无明确代表代码仓库（验证方式为报告精读） |
| 7 | langgraph refresh | multiagent 卡 | A | 高 | 低 | — | — | **是（10-06 昨天）** | 剔除——initial 完成不到 24h，无需 refresh |
| 8 | 记忆基准批判（RealCompanion） | agent-memory 卡 10-07 增量 | B | 中 | 中 | 论文 | — | 否 | 剔除——无代码仓库 |

## 2. 选中项目
- **project**: OWASP Agent Memory Guard（slug: `agent-memory-guard`）
- **repository**: https://github.com/OWASP/www-project-agent-memory-guard.git
- **mode**: initial
- **priority**: P0（S 级卡官方 Incubator 参考实现）
- **source_entry**: KnowlegeMap `03_expansion_queue/candidates/[cand]owasp-agentic-security.md`（2026-10-07 增量：OWASP 从 Top10 列表升级为工程化四件套）+ `[cand]agent-memory-2026.md`（2026-09-12 增量：OWASP Agent Memory Guard——ASI06 Memory Poisoning 参考实现，SHA-256 记忆完整性 + 回滚，被 MITRE ATLAS 引用）

## 3. 选中理由（九维）
1. **Knowledge Value（A）**：OWASP 官方 Incubator 项目，ASI06（Memory Poisoning）唯一参考实现；OWASP Agentic Security 体系从"威胁列表"到"可运行中间件"的落地实证
2. **Agent-AI 相关性（极高）**：Agent Memory 安全 = G1（AI 安全攻防）× G2（Agent Memory）交叉域；记忆完整性（SHA-256 密码学验证/防篡改/回滚/快照取证）
3. **Novelty（高）**：corpus 27+ 项目无记忆安全/记忆完整性/防篡改审计链对象（记忆线均为记忆系统实现：mem0/letta-code/dsh-memory-evolve/memgraphrag）
4. **增长信号（强）**：pushed 2026-10-06（昨天）；10-07 雷达 next-step 第一优先"取证/日志完整性（防篡改审计链）"被 10-07 反取证事件（Asymmetric Security 报告）直接强化
5. **近期发布**：OWASP 官方持续迭代（download stats 更新 + 集成扩展）
6. **corpus 连接（丰富）**：`examples/secure_langgraph_memory.py` ↔ 昨日 langgraph（记忆三层分工 KO-05 的防护侧对照）；MCP server ↔ mcp；OpenTelemetry hook ↔ OTel；LangChain/AutoGen 中间件 ↔ aigis/guardian/ai-protector 防御线；OWASP 家族 ↔ owasp-agentic-skills
7. **未考古**：确认（corpus projects/ 无该对象）
8. **需 refresh**：否（initial）
9. **Benchmark 学习价值（高）**：`benchmarks/security_benchmark.py` + tests + semgrep 规则（agent-memory-unguarded.py）+ SARIF 输出——"安全中间件如何测试与规则化"基准

## 4. 约束确认
- KnowlegeMap / skill-repo 只读 ✅；corpus 可写 ✅
- 每日最多 1 深度项目 ✅（本轮仅此 1 个）
- 不覆盖 corpus 历史 ✅（projects/agent-memory-guard/ 新目录）
- 飞书未授权 → 只 push corpus ✅
- 严禁 create/update/delete cron_job ✅

# Job Manifest · ARCH-2026-10-16-001

- **run_id**: ARCH-2026-10-16-001
- **date**: 2026-10-16（Evaluation/Observability 周 Day5，KnowlegeMap 第 44 次扫描）
- **skill_version**: knowledge-archaeology v3.2
- **source_map_commit**: KnowlegeMap 011775c（2026-10-16 radar watch）

## 候选排序（非 star 排名，按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 是否已考古 / Benchmark 价值）

| 排序 | 项目 | 状态 | 优先级 | 仓库可用性 | 理由 |
|---|---|---|---|---|---|
| P0 | **Pipelock**（luckypipewrench/pipelock） | NEW | **A** | ✅ 有公开仓库（★930, Go, Apache-2.0, 2026-02-08 创建, 2026-10-15 活跃 push, 108 forks） | agent 防火墙品类（10-11 雷达 A 级卡）核心实现：HTTP/MCP/A2A/WebSocket 流量中介扫描 + **mediator-signed action receipts（agent 外部可验证审计证据）**——与 corpus guardian/aigis/ai-protector 同品类形成第三个考古样本（跨项目 Pattern 验证）；与 RAMPART"观测即契约"（10-15）同主题（外部可信证据）；Evidence/Authority 流学习价值高；Benchmark 独特对象（多协议解析 + 签名证据链） |
| P1 | AgentGuard（0xrem/agentguard） | NEW | A | ⚠️ 有仓库但 0 stars、2026-03-18 后未更新（7 个月无活跃） | 三层运行时防火墙（Prompt/Tool/Command Guard）设计有参考价值，但无社区/无活跃信号，浅克隆考古价值低；维持观察 |
| P2 | ToFu（arXiv 2610.13651） | NEW | A | ❌ 无公开代码仓库（GitHub API 搜索 0 命中；候选卡验证计划=待检查） | 253 漏洞/74 sanitizer 大规模实证，但工具未开源；等待 code release |
| P3 | Microsoft Execution Containers | NEW | A | ❌ 无官方公开仓库（仅第三方 demo 0★ 仓库） | 平台级产品，SDK 未开源；等待官方仓库/文档开源 |
| P4 | TestJack / LiveMACE / PI-Hunter | QUEUED | A | ❌ 无公开代码仓库 | 论文族，维持观察，等待 code release |

**已排除（已考古）**：guardian（09-14）/ aigis（09-13）/ ai-protector / agent-governance-toolkit / owasp-agentic-skills / agent-memory-guard / rampart（09-04 + 10-15）/ reviewbench / transect / agentjudge-bench / agentevals / opencode / pyrit / e2b / webwright / hermes-agent / mcp 等。

## 选中项目

- **project**: Pipelock
- **repository**: https://github.com/luckypipewrench/pipelock.git
- **mode**: initial
- **priority**: A
- **reason**: agent 防火墙品类核心实现（mediator 流量扫描 + 签名 action receipts）；agent 外部可验证审计证据 = Evidence/Authority 流的稀缺工程样本；与 guardian/aigis/RAMPART 形成跨项目对照；活跃维护（10-15 最后 push）且规模适中（52MB, Go monorepo）
- **source_entry**: `[cand]agent-firewall-runtime-defense-2026.md` @ KnowlegeMap 03_expansion_queue/candidates/（2026-09-11 雷达 #10，等级 A）

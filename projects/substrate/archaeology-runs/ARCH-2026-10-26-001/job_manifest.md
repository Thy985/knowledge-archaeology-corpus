# Daily Archaeology Job Manifest — ARCH-2026-10-26-001

- **run_id**: ARCH-2026-10-26-001
- **日期**: 2026-10-26（Runtime 周 Day1）
- **选中项目**: **agent-substrate/substrate**
- **repository**: https://github.com/agent-substrate/substrate.git
- **mode**: initial（corpus 无此项目）
- **priority**: P0
- **skill_version**: knowledge-archaeology v3.2

## 候选排序（Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / Corpus 连接 / 考古状态）

| # | 候选 | 状态 | 关键依据 |
|---|------|------|---------|
| 1 | **agent-substrate/substrate** | NEW | Runtime 周头号锚点代码本体（Google 10-22 发布）：Go/Apache-2.0/5785★/627 forks/**pushed 2026-10-25**（24h 内活跃）；<400ms resume + 1500+ act/sec（README 口径）；microVM/gVisor 双沙箱；SPIFFE 身份 + egress 代理；凭据外注；补 Runtime 缺口②（Substrate 开源复现对照）与缺口⑤（Agent Mesh/SPIFFE 身份层）；与 dapr-agents（10-25）/MXC/Strands Box 形成三线对照（durable vs snapshot、OS 隔离 vs 零信任网络） |
| 2 | dapr-agents | COMPLETED | 10-25 已完整考古（PR #54）；3 天内无 refresh 必要 |
| 3 | Google Agent Compute Engine | QUEUED | 预览未开放，属产品面非代码库考古对象；机制由 substrate 仓库承载 |
| 4 | Vercel durable-harness / Notion Gen-6 | QUEUED | durable 品类信号（10-25 fallback 移交）；非单一权威开源仓库，留作周五横向对照 |
| 5 | E2B / microsoft-execution-containers | QUEUED | 资源隔离同域；与已考古 MXC/Strands Box 重复度高，Novelty 弱 |
| 6 | PAP（Personal Agent Protocol） | QUEUED | 标准非代码库（沿用 10-24 否决理由），待 v0.1 规范 |
| 7 | TeamMind | — | 用户私有项目，不在公开仓库考古范围；其权限/身份对照留在缺口④（Harness 周遗留） |

## 选中理由

1. **Knowledge Value A**：零信任 agent 运行时（SPIFFE 身份/egress 网络中介/snapshot-resume 生命周期/低意见运行时 vs SDK 分层）——TeamMind（用户多 Agent 运行时）直接可迁移的架构认知
2. **Agent-AI Engineering 相关性**：与 10-25 dapr-agents（durable execution）互补——生命周期/身份/网络策略为同一问题另一面；Harness 周 MXC/Strands Box 已验证证据（OS 隔离+凭证外挂）在 Google 侧获得官方实现印证（scan log 10-24 已标注"头号候选锚点"）
3. **Novelty / 增长信号**：官方发布 10-22，仓库 10-25 仍高频 push（open issues 878）；5785★ 为外部热度参考（非排序依据）
4. **Corpus 连接**：dapr-agents（durable execution 三件套）↔ substrate（actor→worker 映射+快照恢复）——生命周期与失败恢复两模式对照；Strands Box/MXC ↔ substrate 零信任——隔离证据线三仓收敛
5. **Benchmark 学习价值**：K8s 原生 agent 基础设施的编排/调度/身份/网络强制模式；错误与失败路径（cold-start/resume 预算内控制）

## 否决理由（记录在案，不重复消耗）
- dapr-agents：已考古（10-25 COMPLETED）
- 事件集群/OpenAI 监控对抗：已精读（Harness 周 E7 证据线），非代码库考古对象
- agent-formal-verification / agent-firewall 品类：多同名无单一权威（10-24 否决沿用）
- Vercel durable-harness：无独立权威仓库，仅品类信号

## source_entry
- KnowlegeMap：`07_radar_watch/current_focus.md`（Runtime 周，Agent Substrate 二号锚点）+ `05_scan_logs/2026-10/2026-10-24_radar-fallback_scan.md`（§2 Agent Substrate S 级对象移交）+ `08_agent_centric/domains/agent-runtime.md`（缺口②⑤）+ `03_expansion_queue/candidates/[cand]dapr-agents-distributed-runtime-2026.md`（同域对照卡）
- 仓库事实（GitHub API 2026-10-26）：agent-substrate/substrate · Go · Apache-2.0 · created 2026-05-13 · pushed 2026-10-25 · 5785★/627 forks

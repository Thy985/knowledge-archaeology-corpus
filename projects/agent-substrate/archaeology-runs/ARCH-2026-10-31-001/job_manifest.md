# Job Manifest — ARCH-2026-10-31-001

## 候选排序（按 Knowledge Value / Agent-AI 相关性 / Novelty / 增长信号 / 近期发布 / Corpus 连接 / 未考古 / Benchmark 价值）
| 排序 | 项目 | 状态 | 理由 |
|---|---|---|---|
| 1 | **Agent Substrate**（agent-substrate/substrate） | NEW → 选中 | Runtime 周二号锚点；K8s 零信任 agent runtime（10x 密度/20 万 agent/<500ms suspend-resume/SPIFFE+egress 网络策略）；与 dapr-agents 崩溃恢复路线形成生命周期对照（suspend-resume 路线）；5973★ 增长信号强；未考古 |
| 2 | CoreBreak（跨三栈安全研究） | NEW（留 11-01+） | 高价值但无单一可克隆仓库（三厂商 CVE + Black Hat 披露，非仓库考古对象） |
| 3 | AAIF（Linux Foundation 治理） | QUEUED | 治理方向，与 Runtime 周当前领域连接弱，10-29 新卡待观察 |
| 4 | php2shell（PHP 漏洞事件） | QUEUED | 基础设施事件非仓库；AI-for-Security 案例价值已入雷达 |
| 5 | Agent Compute Engine（Google 托管） | QUEUED | 非开源托管服务，不可仓库考古 |
| 6 | Infinity（AWS） | COMPLETED | 10-29 已考古（PR #58），维持观察 |
| 7 | Dapr Agents | COMPLETED | 10-30 已考古（PR #59），durable 四路线闭环 |

## 选中项目
- **job_id**: ARCH-2026-10-31-001
- **project**: agent-substrate
- **repository**: https://github.com/agent-substrate/substrate.git
- **mode**: initial
- **priority**: P0
- **reason**: Runtime 周（10-26 起）"二号锚点"补全——与 10-30 已验证的 Dapr Agents（崩溃恢复/步骤持久化路线）形成 F 层生命周期对照（suspend-resume 路线）；能力树 D 层（资源隔离与密度）+ E 层（运行时身份与网络策略）+ F1（suspend-resume）闭环；开源 K8s-native 零信任运行时，5973★ 增长信号强，2026-10-22 官方发布（近期）
- **source_entry**: KnowlegeMap 07_radar_watch/current_focus.md（Runtime 周锚点）+ 05_scan_logs/2026-10/2026-10-24_radar-fallback_scan.md（领域内重大事件移交）

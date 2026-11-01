# Job Manifest — ARCH-2026-10-24-001

- **run_id**: ARCH-2026-10-24-001
- **date**: 2026-10-24（周六，Harness/治理 周 Day6）
- **skill_version**: knowledge-archaeology v3.2（生产版，只读）

## 候选排序（按 SOP 标准，非 star）

| # | 项目 | 状态 | 依据 | 结论 |
|---|------|------|------|------|
| 1 | **microsoft/mxc**（Microsoft Execution Containers） | QUEUED（候选卡 [cand]microsoft-execution-containers-2026.md，10-23 已晋升 validated 于雷达层） | Knowledge Value A｜Agent-AI Engineering 高（OS 级 agent 策略隔离层/统一 JSON 策略/四后端映射/三运行模式）｜Novelty 高（10-07 GA 平台原语）｜增长信号强（pushed 2026-10-23；GA 接入 Copilot/OpenClaw/Codex/Replit）｜与已有 Corpus 连接：strands-box（10-23，双对照已建立）/dogwood/deepseek-harness/openclaw｜未深度考古（雷达验证=官方博客精读，无源码考古）｜Benchmark 价值：JSON 策略 schema + 多后端映射 + Learning→Permissive→Enforcement 渐进策略 | **SELECT（initial）** |
| 2 | strands-box refresh | REFRESH_REQUIRED（已考古 10-23，HEAD 未变） | 无新 commit → 不 refresh | skip |
| 3 | agent-firewall 品类（AgentGuard/ClawKeeper/Pipelock/Falcon Guardian） | NEW（品类卡，多同名仓库） | 名称混乱（lictorate/numbergroup/WhitzardAgent 多个 AgentGuard），无单一权威源码 | skip（留待雷达澄清） |
| 4 | dapr-agents-distributed-runtime | QUEUED（候选卡） | Agent 框架而非治理/harness；与本周领域偏离 | skip（Runtime 周候选） |
| 5 | 事件集群卡（Mythos 5 / BIP-44 / Archon） | COMPLETED（雷达层） | 事件卡非代码库 | skip |

## 选中项目
- **project**: mxc（Microsoft Execution Containers）
- **repository**: https://github.com/microsoft/mxc.git
- **mode**: initial（corpus 无 mxc 目录）
- **priority**: P0
- **reason**: 治理周 Day6 自然延续——雷达 A 级候选（10-14 建卡、10-23 官方博客全文验证晋升 validated）的唯一缺环是**源码考古**；与昨日 strands-box 构成 OS 级 agent 遏制双对照组（Box: 策略语言+时间性+凭证网关；MXC: 平台级多后端+组织策略+三模式）；pushed 10-23（昨）显示持续活跃；MIT 开源 Rust 核心可直接考古。
- **source_entry**: `03_expansion_queue/candidates/[cand]microsoft-execution-containers-2026.md`（§5 验证结果 2026-10-23 回填）+ `07_radar_watch/current_focus.md`（Harness/治理 周 Day5 验证记录）

## 状态分类（本次扫描）
- NEW: agent-firewall 品类（待澄清）/ php2shell（雷达事件，非代码库）
- QUEUED: mxc（本次选中）/ dapr-agents / computer-browser-use / trace-based-agent-eval
- COMPLETED（corpus 已有）: 35 项目（a2a…webwright）
- REFRESH_REQUIRED: 无（strands-box 无新 commit；agent-aegis 1.0.0 维持观察）
- IN_PROGRESS: 无（#50/#51/#52 PR 待合并，非考古中）

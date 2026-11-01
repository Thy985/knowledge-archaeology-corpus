# Job Manifest — ARCH-2026-10-20-001

- **run_id**: ARCH-2026-10-20-001
- **日期**: 2026-10-20（Harness/治理 周 Day2，每日 SOP 定时轮）
- **状态**: NEW → SELECTED

## 候选排序（不按 star，按 Knowledge Value / 相关性 / Novelty / 增长 / 连接 / 是否已考古 / Benchmark 价值）

| 排序 | 候选 | 来源条目 | 状态 | 理由 |
|---|---|---|---|---|
| 1 | **Strands Box**（strands-agents/box） | [cand]strands-box-os-containment-2026.md（10-20 Day2 新卡 A 级） | NEW → SELECTED | 与昨日考古 MXC 同域（策略驱动 OS 隔离），周五验证计划明确 MXC×Box 双对照；2026-10-07 AWS 发布（novelty）；pushed 2026-10-19（活跃）；Dogwood 时间性策略+凭证外挂网关 = 高 Benchmark 学习价值；未考古 |
| 2 | agent containment breach cluster | [cand]agent-containment-breach-2026.md（A 级） | NEW | 越权事件集群（威胁情报类），非单一项目，不适合深度考古；可作为 Box 考古的动机对照 |
| 3 | agent-aegis | [cand]agent-aegis-governance-2026.md（B 级） | NEW | B 级，较低优先级；属 agent 防火墙品类（Aigis/AGT 等已考古） |
| 4 | MXC | 候选卡 44（10-19 已考古） | COMPLETED | 已考古（ARCH-2026-10-19-001，PR #48），不重复；作为 Box 对照基准 |
| 5 | Omnigent / MS Agent Framework / AGT / Aigis / dsh 等 | 已有 | COMPLETED | 均已考古，不重复 |

## 选中项目

- **project**: Strands Box（slug: box）
- **repository**: https://github.com/strands-agents/box.git
- **mode**: initial
- **priority**: P0
- **reason**:
  1. 领域契合：Harness/治理 周 B5 层（执行隔离）A 级缺口——OS 级 agent 行为遏制沙箱（候选卡第 2 节）
  2. 新颖性：2026-10-07 AWS Open Source Blog 发布 developer preview（created 2026-10-02，本周内项目）
  3. 增长信号：pushed 2026-10-19（考古当日仍活跃）；★443；open_issues 50
  4. 未考古：corpus projects/ 无 box 条目
  5. 与已有 Corpus 连接：与 MXC（10-19 考古）同域双对照（Friday 10-23 验证计划①=MXC×Strands Box 双对照）；连接 dsh（sandbox/approval seam）/agent-governance-toolkit（无状态策略引擎）/rampart（终局证据）；用户项目 TeamMind（多 Agent 信任边界）/Tafcm（OS 遏制边界）/silver-shield（确定性出站管控）/campus_order（OpenClaw 遏制）
  6. Benchmark 学习价值：Dogwood 时间性策略（temporal 算子）+ 凭证外挂网关 + 多拦截点 enforcement（egress/Python/Shell/MCP）——与 MXC JSON 策略/后端谱系形成两大云厂商沙箱范式对照，可直接作 skill Benchmark 双样本
  7. 是周五对照的必要一半：提前考古 Box = 周五验证直接可用 Box 侧语料
- **source_entry**: `03_candidates/[cand]strands-box-os-containment-2026.md`

## 预期交付
- Snapshot（HEAD commit SHA / 基础地图）
- Project Archaeology Package（00-06 + run_metadata.yaml）
- Independent Validation Report + Reconciliation
- Corpus PR（projects/box/）

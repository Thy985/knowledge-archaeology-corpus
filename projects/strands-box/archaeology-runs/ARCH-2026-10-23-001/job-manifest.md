# Job Manifest — ARCH-2026-10-23-001（2026-10-23 周五 · Harness/治理 周 Day5）

## 选型驱动
- 领域：Harness / 治理（10-19 起，周五验证日——KnowlegeMap 雷达已验证三卡：Strands Box/MXC/事件集群晋升 validated）
- corpus 缺口：OS 级执行遏制端点（MXC 平台级无源码单仓库、agent-aegis 10-22 已考古、dogwood 策略语言 09-08 已考古）——**Strands Box（OS 遏制沙箱）未考古**
- 治理对照谱系补位：deepseek-harness（approval seam）→ agent-aegis（进程内插桩）→ dogwood（策略语言）→ **strands-box（OS 级遏制）** = 执行隔离 B 层完整闭环

## 候选排序（Knowledge Value / Agent-AI 相关性 / Novelty / 增长 / 近期发布 / corpus 连接 / 已考古 / Benchmark）

| 序 | 候选 | 状态 | 判定 |
|---|---|---|---|
| 1 | **Strands Box（strands-agents/box）** | NEW | **选中（P0）**：AWS 官方开源 OS 级行为遏制沙箱；Dogwood 时间性策略 + 凭证外挂网关 + 统一事件语义；与 dogwood/MXC/agent-aegis 构成治理谱系 B 层对照；Benchmark 学习价值高（时间算子/网关/403 回执） |
| 2 | MXC（Microsoft Execution Containers） | QUEUED | 雷达 validated（10-19 对照完成）；平台级能力非单仓库源码——不入 corpus 深度考古，作对照材料 |
| 3 | 事件集群（越权 breach cluster） | QUEUED | 非单一仓库（Mythos 5/BIP-44/Archon RCE 多源）；雷达精读已完成——不考古 |
| 4 | agent-aegis | COMPLETED* | 10-22 已考古（PR #51 未合并，corpus main 无该目录）；不重复考古 |
| 5 | Hermes / OpenClaw / MCP 等 | COMPLETED | 已在 corpus（hermes-agent/openclaw/mcp） |
| 6 | Warden（LobeHub/AlwaysReadyAllies） | REJECTED | GitHub 404 实测排除（10-22 复核一致） |
| 7 | Vaikora / mcpkernel | REJECTED | 日期/star 不符实测排除（10-22 复核一致） |

## Job Manifest
```yaml
job_id: JOB-2026-10-23-001
run_id: ARCH-2026-10-23-001
project: strands-box
slug: strands-box
repository: https://github.com/strands-agents/box.git
mode: initial
priority: P0
status: NEW
reason: >
  Harness 周 Day5 验证日主角；AWS 官方（Apache 2.0）OS 级行为遏制沙箱，
  内嵌 Dogwood 策略引擎（Cedar 语法 + 时间性算子）、凭证外挂网关（5 种注入，
  真实密钥不进 agent 环境）、统一事件语义（fs:read/http:request 跨工具归一）；
  corpus 无该目录 → NEW；与已考古 dogwood（策略语言）/MXC 对照线（平台级）/
  agent-aegis（进程内）构成执行隔离 B 层完整谱系；
  Benchmark 学习价值：时间性策略 + 拒绝可观测（403+rule id）+ 网关凭据隔离
source_entry:
  - 03_expansion_queue/candidates/[cand]strands-box-os-containment-2026.md（validated，2026-10-23）
  - 07_radar_watch/current_focus.md（Day5 验证日）
  - 05_scan_logs/2026-10/2026-10-23_radar-watch_scan.md
facts:
  repo_head_probe: 8c46dacd5825ac1d11c6d1f4200f0daa657c5681
  published: 2026-10-07（AWS Open Source Blog，developer preview）
  license: Apache-2.0
  note: 雷达卡记录"GitHub robots.txt 拦截"为爬虫限制；git 协议实测可克隆（本 Manifest 实证）
```

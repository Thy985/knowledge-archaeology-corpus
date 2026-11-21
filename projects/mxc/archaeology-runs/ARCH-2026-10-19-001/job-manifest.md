# Job Manifest — ARCH-2026-10-19-001（MXC · initial）

## run_id / 项目
- **run_id**: `ARCH-2026-10-19-001`
- **project**: microsoft/mxc（Microsoft eXecution Container）
- **repository**: https://github.com/microsoft/mxc.git
- **mode**: initial
- **priority**: P0

## 候选排序（10-19 轮，Harness/治理 周 Day1）
| 候选 | 状态 | 排序依据 | 结论 |
|---|---|---|---|
| **MXC** | **NEW**（corpus 无记录） | Harness/治理 周 A 级（候选卡 44，能力树 B1）；10-07 GA 后 10-17 仍 push（活跃）；★2905；OS 级策略隔离是治理范式新层；与 dsh/AGT/aigis 同生态连接；周五验证计划第 1 项 | **选中** |
| Omnigent | COMPLETED | 已考古 ARCH-2026-09-07-001 | 跳过 |
| MS Agent Framework | COMPLETED | 已考古 ARCH-2026-09-29-001 | 跳过 |
| DeepSeek Harness | COMPLETED | 已考古 ARCH-2026-09-03-001（validated 基线） | 跳过 |
| OpenCode | 已排除 | archived=True + 09-09 已考古 | 跳过 |
| Agent 防火墙品类（Aigis/AGT/AI-Protector/Guardian/AMG） | COMPLETED | 均已在 corpus（10-07~10-15 轮） | 跳过 |
| ClawKeeper / Pipelock / AgentGuard | QUEUED | 无官方公开仓库 / 活跃度不足 | 观察 |

## 选中理由（MXC）
1. **领域契合**：Harness/治理 周能力树 B 层"执行隔离与沙箱"的 A 级缺口（B1 MXC OS 级），周五验证计划第 1 项" MXC 实测/对照"
2. **新颖性**：2026-10-07 GA 的平台级执行边界原语——"agent 不能成为自身安全权威"，策略由 OS 独立强制；与 E2B（云沙箱）互补的本地/桌面侧答案
3. **增长信号**：GA 首日即接入 GitHub Copilot/OpenClaw/OpenAI Codex/Replit/LM Studio；10-17 仍有 push；★2905
4. **未考古**：corpus 无 mxc 项目记录（10-18 轮"无公开仓库"结论已过时——`microsoft/mxc` 存在且活跃）
5. **与已有 Corpus 连接**：deepseek-harness（sandbox/approval 范式）、agent-governance-toolkit（无状态策略引擎）、aigis（运行时防御）——MXC 提供 OS 级强制层，形成治理范式对照链
6. **Benchmark 学习价值**：统一 JSON 策略 schema（五域）、双层策略（workload+组织）、三运行模式（Learning/Permissive/Enforcement）、多后端映射表（AppContainer/Seatbelt/Bubblewrap/LXC/MicroVM）——直接迁移到 Tafcm/TeamMind 权限模型
7. **source_entry**: `03_expansion_queue/candidates/[cand]microsoft-execution-containers-2026.md`（候选卡 44，2026-10-14 兜底捕获）

## refresh 观察项（若后续 refresh）
- OpenClaw MXC 集成 commit（campus_order 栈）
- 后续后端稳定性（MicroVM/Hyperlight experimental → stable）
- 策略 schema 版本演进（schemas/stable/ 的版本化）

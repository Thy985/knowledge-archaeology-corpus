# Job Manifest · 2026-11-07

- **job_id**: ARCH-2026-11-07-001
- **project**: modelcontextprotocol/go-sdk
- **repository**: https://github.com/modelcontextprotocol/go-sdk.git
- **mode**: initial
- **priority**: A
- **source_entry**: KnowlegeMap 候选卡 `03_expansion_queue/candidates/[cand]mcp-2026-07-28-stateless.md`（记录 "go-sdk 已按 Stateless 新规发布（2026-09-01）"）；11-07 回灌 connections 的协议拼图背景（MCP 连工具 / A2A agent 互操作 / PAP 人→企业授权）
- **run_id**: ARCH-2026-11-07-001
- **slug**: `go-sdk`
- **branch**: `arch/go-sdk-2026-11-07`

## 1. KnowlegeMap 状态读取

| 来源 | 结论 |
|---|---|
| `07_radar_watch/current_focus.md` | 领域 Runtime，第二周（11-02~11-08）。11-07 为周六回灌日；11-08 第二周小结并评估换域（下一领域候选 Memory） |
| `05_scan_logs/2026-11/2026-11-07_radar-watch_scan.md` | 当天不新增扫描、不引入新候选；只做 11-06 结论回灌（graph 4 边 / connections 2 行 / 能力树 §七） |
| `03_expansion_queue/candidates/` | 63 张候选卡；Runtime 主流仓库（Dapr Agents / Substrate / E2B / MXC / Box / OpenKruise Agents / Restate / DBOS / Infinity 等）多数已进入 corpus |
| `04_connections/` | 11-07 新增 Agent Substrate、OpenKruise Agents v0.6.0 两行 ✅ validated；durable 品类维持 🟡 |
| corpus `projects/` | 48 个目录；**无 `go-sdk` / `mcp-go-sdk`**（去重通过） |

## 2. 候选状态盘点（只列与 Runtime 相关的头部候选）

| 候选 | 状态 | 依据 |
|---|---|---|
| **modelcontextprotocol/go-sdk** | **NEW** | 未考古；2026-11-06 仍活跃（pushed）；v1.8.0（2026-09-14）；对应"无状态协议落地"的实现侧 |
| agent-formal-verification（Dogwood / AProver / Lean4Agent / ePCA） | COMPLETED（部分） | Dogwood、AProver 已在 corpus；Lean4Agent 无稳定主仓库证据，剩余部分不足以独立成篇 |
| agent-memory-2026（Mem0 / Letta / Zep / A-MEM） | COMPLETED（主体） | Mem0、Letta-code、agent-memory-guard 已在 corpus；且 Memory 为 11-08 换域候选，不抢跑 |
| personal-agent-protocol-2026（PAP/Poppy） | QUEUED | 开放标准但**候选卡内无 GitHub 仓库**，可考古对象仅规范文本，暂不满足"仓库级考古" |
| multiagent-framework-2026（LangGraph / CrewAI / AG2 / MS Agent Framework） | COMPLETED（主体） | LangGraph、agent-framework 已在 corpus |
| sandbox-lifecycle-suspend-resume-2026-11（OpenKruise Agents v0.6.0） | COMPLETED | 11-06 已交付 `projects/openkruise-agents/`（PR #66） |
| Agent Substrate / E2B / Box / MXC / browser-harness / OpenCode | COMPLETED | 均已有 corpus 目录 |

## 3. 选中理由（按知识价值排序，不看 star）

1. **补 Runtime 的"协议实现层"缺口**：能力树已覆盖 durable execution、沙箱隔离、生命周期控制面；MCP 是 agent 工具连接的事实标准，但 corpus 现有的 `projects/mcp/` 考古对象是**规范仓库**（`modelcontextprotocol/modelcontextprotocol`，TypeScript），实现侧空白。go-sdk 补上"规范 → 官方 Go 实现"这一层。
2. **与本周核心命题直接对话**：11-07 回灌的 `zero-credential-execution`、`identity-egress-fusion` 都是"声明与执行"问题；go-sdk 正好可检验 **2026-07-28 无状态协议** 的条款（session 建立/恢复/关闭、OAuth、传输切换）在官方实现中是否真的可达——延续 11-06 的 KO-02「策略声明 ≠ 执行可达」方法论。
3. **Novelty**：MCP 2026-07-28 是"史上最大版本"（有状态双向 → 无状态 request/response），9 月 SDK 跟进；协议转型后的实现取舍尚未被本 corpus 记录。
4. **增长信号**：5,318★ / 613 forks；2026-11-06 15:38 UTC 仍有 push；v1.8.0 已发布。
5. **Benchmark 学习价值**：与 `projects/mcp/`（规范）构成天然对照样本——同一套协议文本，规范说什么、SDK 做什么、测试证明什么，三者可分层比对。
6. **与已有 Corpus 的连接**：MCP（工具连接）× A2A（agent 互操作）× PAP（人→企业授权）三块协议拼图；go-sdk 的 OAuth/session 机制可与 `agent-governance-toolkit`、`agent-aegis` 的权限模型对照。

## 4. 排除理由（本轮不选）

- **Agent Substrate / OpenKruise Agents**：11-06 刚完成深度考古，重复。
- **Temporal / Infinity / Restate / DBOS / durare / semanticrun**：durable 品类已多轮覆盖；refresh 需等有新版本或重大变化。
- **Genspark SB-Box / AWS AgentCore**：闭源，只能作为对照证据，不能仓库级考古。
- **PAP/Poppy**：无可考古仓库。
- **Cline**：与 `opencode` / `openclaw` 的 coding-agent harness 主题重叠度高，且规模巨大，边际知识增量低于 go-sdk。
- **Memory 类项目**：11-08 换域后系统开展，本轮不抢跑。

## 5. 核验记录（GitHub API，2026-11-07）

| 字段 | 值 |
|---|---|
| full_name | `modelcontextprotocol/go-sdk` |
| stars / forks | 5,318 / 613 |
| language | Go |
| created_at | 2025-04-23T10:07:33Z |
| pushed_at | 2026-11-06T15:38:14Z |
| size | 5,583 KB |
| default_branch | `main` |
| open_issues | 77 |
| latest release | v1.8.0 @ 2026-09-14T08:05:46Z |
| license | NOASSERTION（需以仓库 LICENSE 文件为准，Snapshot 阶段回读） |

> 真实性核验依据经验 `oss-validation-capability-boundary`：先确认 `owner/repo` 归属，再 clone；不以项目名推测仓库。

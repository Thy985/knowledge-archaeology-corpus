# Daily Archaeology Job Manifest — 2026-11-16

## 环境与权限基线

| 项 | 值 |
|---|---|
| 本地日期 | 2026-11-16（Asia/Shanghai） |
| GitHub 身份 | `Thy985`（PAT 来自 `/home/user/.git-credentials` 第 1 行，未写入任何产物） |
| KnowlegeMap | 本地 `fdd8ff1`；`origin/main` `28eae90`（2026-11-16 Memory 第二周 Day1 扫描） |
| skill-repo | `931124c`（v3.2，主分支） |
| corpus | 本地当前分支 `arch/graphiti-2026-11-15` @ `f50c4b0`；`origin/main` `9d27bc5`；`projects/` 50 个目录 |
| push 权限 | KnowlegeMap / skill-repo / corpus 三仓 `git push --dry-run` 均 `[new branch]`、exit=0 |

## KnowlegeMap 当前状态

- `07_radar_watch/current_focus.md`：Memory 第二周（11-16~11-22），F 层信任边界（S）与 E 层作用域（A）仍未验证。
- `05_scan_logs/2026-11/2026-11-16_radar-watch_scan.md`：新建 S 卡 `[cand]agent-memory-trust-boundary-2026.md`（Trojan Hippo / PASB / MIND / MemLeak / OWASP AMG）。
- `03_expansion_queue/candidates/[cand]agent-memory-2026.md`：Memory 实现层锚点（Mem0 / Zep→Graphiti / Letta / Hy-Memory 2.0 / SIx / MemEvolve）。
- Corpus 已覆盖（相邻）：`mem0`、`memmachine`、`letta-code`、`graphiti`、`agent-memory-guard`、`dsh-memory-evolve`。

## 候选排序（不按 star）

| # | 项目 | 状态 | 优先级 | Knowledge Value / 相关性 | Novelty / 增长 | 代码可取证性 | 与 Corpus 连接 | 决策 |
|---|---|---|---|---|---|---|---|---|
| 1 | **OWASP Agent Memory Guard**（`OWASP/www-project-agent-memory-guard`） | `REFRESH_REQUIRED` | **A−** | 直接补 Memory F 层（写入/读取信任门、检测器、完整性基线）与 E 层（per-agent access、namespace、admin gate） | **0.3.3→0.4.0 大版本**：115 commits、300 changed files、+420/−56；新增 per-agent access control / agent identity / private namespaces / registry & roles / class gate / explain trace；另有 redaction 未掩码安全修复 | 纯 Python，src/tests/scanner/mcp-server 全在仓库；可静态取证 | 已有 initial（ARCH-2026-10-07-001），本次是 refresh，不覆盖旧 run | **选中** |
| 2 | 腾讯 Hy-Memory 2.0 | `NEW` | B+ | 工作记忆 + 代码记忆、MemHub 跨工具共享；直接补 A2/A4/E | 三层全开源，2026-10-16 | 核心是 30B-A3B 模型 + SDK，行为验证需 GPU；静态代码价值低于 AMG | 无 | 暂缓 |
| 3 | MemEvolve / EvolveLab | `NEW` | B+ | 记忆架构元进化、12 系统统一接口 | ICML'26，代码可用 | 行为验证需 API keys + GAIA/WebWalkerQA/xBench | 无 | 暂缓（成本高） |
| 4 | SIx Harness《Memory as Infrastructure》 | `NEW` | B | H 层可靠性/健康门价值高 | 8 个月运营记录 | 以论文/记录为主，主仓库取证面待确认 | 无 | 暂缓 |
| 5 | Trojan Hippo / MemLeak / MIND / PASB | `NEW` | B | F/E 威胁模型强 | 2026 新论文 | 论文与基准为主，非完整工程仓库 | 无 | 不选（非代码考古对象） |
| 6 | mcp-memory-service / OKF / agentmemory | `QUEUED` | C+ | 小工具，增量有限 | 版本迭代 | 有代码但价值密度低 | 部分相邻 | 不选 |
| 7 | Graphiti | `COMPLETED` | — | 昨日刚完成 initial | 无新 commit | 已取证 | 已写入 corpus | 不选（无 refresh 信号） |

## 选中项目

- **project**：OWASP Agent Memory Guard
- **repository**：`https://github.com/OWASP/www-project-agent-memory-guard.git`
- **slug**：`agent-memory-guard`
- **mode**：`refresh`
- **priority**：`A−`
- **job_id / run_id**：`ARCH-2026-11-16-001`
- **source_entry**：
  - `03_expansion_queue/candidates/[cand]agent-memory-trust-boundary-2026.md`（2026-11-16 新建 S 卡，OWASP AMG 条目）
  - `03_expansion_queue/candidates/[cand]agent-memory-2026.md`（Memory 实现层锚点）
  - `05_scan_logs/2026-11/2026-11-16_radar-watch_scan.md`（F/E 层建基线）
- **refresh 触发证据**：
  - 归档 commit `8403091`（0.3.3，2026-10-06）→ 当前 `b63f76e`（0.4.0 + unreleased fix，2026-11-15）
  - GitHub compare：`ahead_by=115`、300 files changed、+420/−56
  - `pyproject.toml` version `0.3.3` → `0.4.0`
  - CHANGELOG 新增：per-agent access control、agent identity、private namespaces、agent registry/roles、class gate/admins、值深拷贝、`explain`/`trace`、`written_by`；Security: redaction fail-closed
- **选择理由**：这是当前唯一“既有代码可取证、又在两周内发生权限/作用域结构性变更”的项目；它把 Memory 周从“论文层威胁模型”落到“可执行门禁实现”，能直接回答 E 层“scope 是否只是标签”和 F 层“写入/读回是否 fail-closed”。

## Scope

- 只读目标仓库；浅克隆到 `archaeology-jobs/ARCH-2026-11-16-001/repo`。
- 覆盖 `src/agent_memory_guard/`、`tests/`、`scanner/`、`mcp-server/`、`benchmarks/`、`integrations/`、配置与 CI。
- 不修改 KnowlegeMap、生产 Skill、目标仓库。
- 不覆盖 corpus 旧 run `ARCH-2026-10-07-001`；本次新增 `archaeology-runs/ARCH-2026-11-16-001/`，顶层只做增量说明。

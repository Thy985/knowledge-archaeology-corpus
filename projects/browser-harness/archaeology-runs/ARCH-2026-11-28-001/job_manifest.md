# Job Manifest — KA-2026-11-28-001

- **job_id**: `KA-2026-11-28-001`
- **run_id**: `ARCH-2026-11-28-001`
- **date**: 2026-11-28（Asia/Shanghai）
- **project**: `browser-harness`
- **repository**: `https://github.com/browser-use/browser-harness.git`
- **mode**: `refresh`
- **priority**: `A`
- **target ref**: tag `v0.1.15` @ `373aecc4c40ef67d97b5ee69c144684a64a42a11`
- **baseline run**: `ARCH-2026-10-04-001`（`0.1.13 @ afbcc381b963040c19627d788e40c7e7663171ee`）
- **source_entry**:
  - KnowlegeMap `07_radar_watch/current_focus.md`（Computer Use / Browser 第一周 Day6，2026-11-28 回灌日）
  - KnowlegeMap `05_scan_logs/2026-11/2026-11-28_radar-watch_scan.md`（今日只回灌、不新扫描；必须有项目可挖）
  - KnowlegeMap `03_expansion_queue/candidates/[cand]browser-harness-2026.md`（A 卡，仍标 `[cand]`）
  - KnowlegeMap `08_agent_centric/domains/agent-computer-use.md` §2.1 / §10.4（browser-use 1.0.8 的 daemon 依赖与 P1 attach 风险）
  - Corpus `projects/browser-harness/metadata.yaml`（0.1.13 baseline，2026-10-04）
  - Corpus `projects/browser-use/archaeology-runs/ARCH-2026-11-27-001/02-engineering-knowledge/engineering_knowledge.md`（三平面执行：daemon 持浏览器）
- **只读边界**: 不修改目标仓库、KnowlegeMap、生产 Skill；仅写 Corpus

## 候选排序（不按 star）

| 排名 | 项目 | 状态 | 证据 | 判断 |
|---|---|---|---|---|
| 1 | **browser-harness** | **REFRESH_REQUIRED** | 基线 `0.1.13 @ afbcc381`（2026-10-04）；PyPI 最新 **0.1.15（2026-11-25）**；GitHub repo pushed **2026-11-26**；tag `v0.1.15 @ 373aecc`；`v0.1.13...v0.1.15` 共 **403 commits**，含 owner/tab 归属、`release_tabs`、one daemon per browser、raw CDP isolation | **选中**。它既是 browser-use 1.0.8 的直接依赖（`browser-harness==0.1.13`），也是昨日“三平面执行”中持浏览器的那一平面；新版变更正好命中旧候选 C-02“多 agent 共享浏览器 lane 并发安全边界”，可用一个项目同时闭合依赖缝隙与旧候选 |
| 2 | agent-desktop | NEW | npm 最新 **1.3.1**（候选卡写 1.2.0，已过时）；GitHub 75,617 KB、Rust、27,070 stars、pushed 2026-11-27；A 卡；包级核对仍挂账 | 高价值（桌面 a11y + `STALE_REF` / `disposition.retry`），但 75MB Rust 仓库超过单日可闭合体量，且与已有 Corpus 连接少于 browser-harness；排明日/后续 |
| 3 | chrome-devtools-mcp | NEW | GitHub 14,614 KB、TypeScript、59,601 stars、pushed 2026-11-27；约 18.6 万行 / 420 文件 | 价值高但与 browser-harness / Playwright MCP 对照重叠；单日无法同等深度闭合，排除 |
| 4 | Webwright | QUEUED / B | `0.0.7`，repo 140,750 KB，pushed 2026-10-27，无新版本；`[cand]` 维持 B | “脚本即产物”已由昨日 browser-use cell 落盘对照定性；边际收益低 |
| 5 | openclaw | REFRESH_REQUIRED | 基线 `2026.9.4 @ ca38979d`，npm 已到 2026.11.10；CVE-2026-28458 的 Browser Relay `/cdp` 无 token | gateway monorepo 远大于单日；且今日雷达主体不是它 |
| 6 | playwright-mcp | RECENTLY_COMPLETED | `0.0.86`，2026-11-23 完成 initial；PR 尚未合并 | 4 天内重复深度考古边际收益低 |
| 7 | browser-use | RECENTLY_COMPLETED | `1.0.8`，2026-11-27 完成 refresh（PR #84 open，main 未合并） | 不重复；今日只把它当 browser-harness 的调用方证据 |
| 8 | local-control-plane-exposure-2026-11 | EVIDENCE_ONLY | 跨对象归纳卡，不是可克隆仓库 | 作为问题定义与 Benchmark 评分表，不做项目考古 |

**状态统计**：NEW 2（agent-desktop、chrome-devtools-mcp）· QUEUED 1（Webwright）· COMPLETED 1（playwright-mcp）· REFRESH_REQUIRED 3（browser-harness、openclaw、browser-use 但刚完成）· IN_PROGRESS 0 · EVIDENCE_ONLY 1。

## 选择理由（按评分维度）

- **Knowledge Value**：browser-harness 是 browser-use 1.0.8 唯一直接运行时依赖（`browser-harness==0.1.13`），是“parent/worker/daemon 三平面”中唯一持浏览器与跨 REPL 存活的平面；理解它等于理解 browser-use 的执行底座。
- **Agent-AI Engineering 相关性**：直接处理 CDP 会话权威、daemon 生命周期、agent 身份、tab 归属、多客户端并发、raw CDP 隔离与隐私——都是 Agent 执行面的核心工程问题。
- **Novelty**：0.1.13→0.1.15 新增 **owner 模型**（每个 agent 只驱动自己的 tab；`BH_OWNER` 传递与校验；`release_tabs`；one daemon per browser；`one Allow per Chrome run`）。这是从“单 agent 工具”到“多 agent 共享浏览器”的并发模型变更，不是普通功能迭代。
- **增长信号 / 近期发布**：PyPI 0.1.15 于 2026-11-25 11:56 UTC 上传；repo pushed 2026-11-26；403 commits since 0.1.13。
- **与已有 Corpus 连接**：`browser-use`（直接依赖、三平面）、`playwright-mcp`（控制面/ref 失效语义）、`openclaw`（Browser Relay 无 token）、`local-control-plane-exposure`（P1/P9）。
- **是否已考古 / 是否需要 refresh**：已考古 0.1.13（2026-10-04），当前 0.1.15，**必须 refresh**；旧 KO 中“daemon 身份/拒绝重连/受控中间层”需按 owner 模型重验，不能整包继承。
- **Benchmark 学习价值**：可把“多 agent 共享浏览器”的隔离边界做成可评分 case（owner 混淆、tab 越权、raw CDP 旁路、daemon 跨版本重启）。

## 必须回答的五个问题

1. **owner 模型**：`BH_OWNER` 从哪来、如何跨 sub-agent 传递、daemon 如何校验？未带 owner 的子 agent 为什么不能 close/release 父 agent 的 tab？哪些路径仍可绕过？
2. **tab 生命周期**：`release_tabs`、idle release“never closes a tab”、idle 一天后 forget/detach、finished agent 的 tab 不永久占用——这些规则在代码里如何实现，边界在哪？
3. **daemon 竞争**：one daemon per browser / one Allow per Chrome run 如何保证？多客户端同时连接时谁赢？`daemon-outdated` 与“older client never restarts newer daemon”的失败封闭逻辑是什么？
4. **raw CDP 隔离与隐私**：`Isolation and privacy hold for raw CDP too` 是代码事实还是文档声明？iframe 只能从已知包含它的 tab 可达，如何校验？Network 事件只从 current tab 流出，如何保证？
5. **与 browser-use 1.0.8 的版本缝隙**：browser-use 固定 `browser-harness==0.1.13`，而上游已到 0.1.15；0.1.14/0.1.15 的 owner 能力是否被 browser-use 使用？若未使用，多 agent 并发风险留在哪一层？

## 预期旧 KO 处理

- **保留候选**：进程身份端到端验证、被拒不重连纪律、受控中间层 + 运行时能力扩展、cloud 计费权威、四层输出面统一脱敏。
- **必须重验**：多 agent 共享浏览器 lane 的并发安全边界（旧 C-02，SKILL.md 声称串行安全、仓库内无互斥实现）——0.1.15 的 owner/tab 模型可能已把它从“候选”变成“实现”，也可能只覆盖部分路径。
- **可能降级为“0.1.13 实现”**：任何依赖旧 daemon 单客户端假设的结论。

## 产物路径

- repo: `archaeology-jobs/ARCH-2026-11-28-001/repo`
- snapshot: `archaeology-jobs/ARCH-2026-11-28-001/snapshot_artifact.md`
- package: `archaeology-jobs/ARCH-2026-11-28-001/package/`
- corpus run: `projects/browser-harness/archaeology-runs/ARCH-2026-11-28-001/`

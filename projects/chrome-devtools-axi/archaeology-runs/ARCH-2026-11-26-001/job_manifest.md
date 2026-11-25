# Daily Archaeology Job Manifest — 2026-11-26

- **job_id**: `KA-2026-11-26-001`
- **run_id**: `ARCH-2026-11-26-001`
- **date**: 2026-11-26（Asia/Shanghai）
- **mode**: `initial`
- **priority**: `A`
- **project**: `chrome-devtools-axi`
- **repository**: `https://github.com/kunchenguid/chrome-devtools-axi.git`
- **selected_version**: npm `0.1.46`；HEAD `1dbdac1ca1122e1d2b76d889bbc8a2dd2eb2dcd7`（2026-11-04）；`git describe` `chrome-devtools-axi-v0.1.46-2-g1dbdac1`

## Selection summary

今日 KnowlegeMap 雷达为 Computer Use / Browser · Day4 · 安全增量。当日新增 A 卡 `[cand]local-control-plane-exposure-2026-11.md`，把本地控制面按“谁能触发”拆成 7 种失效模式；其中 P2 是 `chrome-devtools-axi ≤0.1.26` 的本地 HTTP bridge 缺失 Host/Origin 校验（GHSA-x439-jhfh-v9x2，DNS rebinding 可接管 Chrome 会话），0.1.27 修复。该对象同时满足：

1. **NEW**：Corpus 无 `chrome-devtools-axi`，KnowlegeMap 无独立候选卡（仅在 P2 作为一手安全证据）。
2. **代码可追溯的闭合边界**：约 21k 行、78 个 tracked files，`src/bridge.ts` 是本地控制面本体；0.1.27 的修复 commit `4ee5a49` 与 0.1.46 的现行 gate 均可逐行追溯。
3. **Novelty / Benchmark**：它不是“又一个浏览器 CLI”，而是一个“无认证 loopback 服务如何用 Host/Origin 做 anti-rebinding fail-closed”的小型安全样本；可直接作为本地控制面审查的 benchmark case。
4. **Connections**：与已考古的 `browser-harness`、`webwright`、`agent-desktop`、`playwright-mcp`（PR #80）以及 `local-control-plane-exposure` 卡 P2 直接相连；可与 `chrome-devtools-mcp` 上游形成 wrapper/upstream 对照。
5. **近期发布**：npm `0.1.46`（`time.modified` 2026-10-29T13:48:43Z），仓库 HEAD 2026-11-04；不是陈旧对象。
6. **增长信号**：不是按 star 排序；选择依据是安全事件后的修复代码、测试覆盖与可复用判据密度，而非热度。

## Candidate ranking

| rank | object | state | decision | reason |
|---|---|---|---|---|
| 1 | `chrome-devtools-axi` | NEW | **SELECTED** | 唯一同时具备“真实 CVE + 修复 commit + 现行 fail-closed gate + 本地可读测试”的小型控制面样本；单项目边界闭合，适合做深考古与 benchmark。 |
| 2 | `openclaw` | REFRESH_REQUIRED | QUEUED | Corpus 已有 2026-09-17 initial（2026.9.4）；npm 已到 `2026.11.10`，且 CVE-2026-28458 涉及 Browser Relay。价值高，但属于 gateway 级大范围 refresh，今天若做会挤占唯一深度名额；排到安全周后段做 delta refresh。 |
| 3 | `chrome-devtools-mcp` | NEW | QUEUED | 上游官方 MCP server，npm `1.14.0`（2026-11-25 更新），约 186k 行、420 files；与 Axi 的 wrapper/upstream 对照有价值，但体量与今日“单点安全边界”目标不匹配。 |
| 4 | `playwright-mcp` | REFRESH_REQUIRED | QUEUED | 2026-11-23 刚完成 initial（PR #80 open）；今日新增 CVE-2025-9611 细节，但同一对象 3 天内重复深度考古边际收益低，先以增量证据入卡。 |
| 5 | `praisonai` | NEW | QUEUED | 默认 `0.0.0.0` browser bridge（CVE-2026-40289）是 P3 证据；但它是大型多 agent 框架，单日全量考古会偏离 bridge 边界。 |
| — | `local-control-plane-exposure-2026-11` | EVIDENCE_ONLY | NOT_A_REPO | 7 模式 + 6 问是跨对象归纳卡，不是可克隆仓库；作为本次考古的上游问题定义与 Benchmark 评分表。 |

## Job definition

```yaml
job_id: KA-2026-11-26-001
project: chrome-devtools-axi
repository: https://github.com/kunchenguid/chrome-devtools-axi.git
mode: initial
priority: A
focus:
  - local HTTP bridge security boundary
  - anti-DNS-rebinding Host/Origin gate
  - fail-closed behavior before any /call
  - bridge lifecycle / idle / session / page routing
  - keychain/profile isolation as adjacent credential boundary
source_entry:
  - 07_radar_watch/current_focus.md (Computer Use / Browser · Day4 安全增量)
  - 05_scan_logs/2026-11/2026-11-26_radar-watch_scan.md (P2 Playwright MCP / P2 chrome-devtools-axi)
  - 03_expansion_queue/candidates/[cand]local-control-plane-exposure-2026-11.md (P2 + 6 问)
  - 08_agent_centric/domains/agent-computer-use.md (E 隔离与凭据 / F 信任边界)
```

## Boundaries

- Target repository is read-only; only the shallow clone under `archaeology-jobs/ARCH-2026-11-26-001/repo` is used.
- KnowlegeMap and production Skill are read-only.
- Corpus write is limited to `projects/chrome-devtools-axi/` plus the repository index; history is not overwritten.
- No Feishu write without explicit authorization.
- If runtime execution is unavailable, the Package must label runtime claims as not executed instead of promoting them to Fact.

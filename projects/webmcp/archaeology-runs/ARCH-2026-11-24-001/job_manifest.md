# Daily Archaeology Job Manifest — 2026-11-24

- **job_id**: `ARCH-2026-11-24-001`
- **run_id**: `ARCH-2026-11-24-001`
- **project**: `webmcp`
- **repository**: `https://github.com/webmachinelearning/webmcp.git`
- **commit**: `a768d04b4f9ac7f3d4357671db9686675d52c8e5`
- **branch**: `main`
- **mode**: `initial`
- **priority**: `A`
- **selected_at**: `2026-11-24` (Asia/Shanghai)

## Candidate ranking

Ranked by knowledge value, Agent-AI engineering relevance, novelty, growth signal, recency, connection to existing corpus, prior coverage, refresh need, and benchmark learning value — not stars.

| # | Candidate | State | Evidence | Decision |
|---|---|---|---|---|
| 1 | **WebMCP** (`webmachinelearning/webmcp`) | **NEW** | KnowlegeMap 2026-11-24 new A card; Playwright MCP 11-18 release exposes page tools as `webmcp_<tool>`; Chrome 149 / Edge 150 origin trials; repo HEAD `a768d04` on 2026-11-23 | **SELECTED** |
| 2 | browser-use 1.0.8 | REFRESH_REQUIRED / incremental | 1.0.7→1.0.8 on 2026-11-22; judge `gotchas`/`judge_context`, `sensitive_data` non-sandbox, local debug port | Queued. Previous corpus run `ARCH-2026-11-22-001` covers 1.0.8rc1; new stable delta is small. |
| 3 | Playwright MCP WebMCP surface | REFRESH_REQUIRED / duplicate project | `microsoft/playwright-mcp` v0.0.86 was archaeologically examined on 2026-11-23; WebMCP surface was noted but not deep-mined | Queued. Do not deep-archaeologize the same project twice in two days. |
| 4 | browser-harness | COMPLETED | Corpus project exists; v0.1.13 @ `afbcc38` previously covered; no new release found in 2026-11-24 scan | Not selected |
| 5 | Webwright | COMPLETED | Corpus project exists; no new release; GitHub releases page blocked by robots | Not selected |
| 6 | Playwright CLI / Chrome DevTools MCP | NEW / adjacent | Relevant, but lower novelty than WebMCP and no direct page-declared-tool surface | Not selected |

## Why WebMCP

1. **NEW and structurally important**: it is the page-side contract layer that turns web apps into agent tool providers; it is not a variation of an existing browser automation harness.
2. **Directly connects to the week's axis**: the Computer Use / Browser capability tree's A2 (environment representation), B1 (abstract action), D1 (session), E2 (local control surface), F1 (page content as untrusted input) all intersect WebMCP.
3. **Creates a route contrast with prior corpus**: `browser-use` (REPL + CDP), `browser-harness` (CDP control plane), `playwright-mcp` (a11y snapshot + ref actions) vs WebMCP (page-declared tools). This is a clean architectural comparison, not another repo tour.
4. **Growth signal is real but not mature**: Chrome 149 and Edge 150 origin trials, ChatGPT Desktop support, Brave 1.102.2 planned 2027-01-05, Meta Ray-Ban Display "coming soon"; Firefox/Safari only standards positions. This is a live implementation-first standardization case.
5. **Benchmark value**: exposes a general question — can a protocol standardize tool discovery/exposure without standardizing intent, consent, or schema enforcement? That is testable against the corpus.

## Source entry

- `KnowlegeMap / 07_radar_watch/current_focus.md` (2026-11-24, Day2 incremental)
- `KnowlegeMap / 05_scan_logs/2026-11/2026-11-24_radar-watch_scan.md`
- `KnowlegeMap / 03_expansion_queue/candidates/[cand]webmcp-page-declared-tools-2026.md` (A)
- `KnowlegeMap / 08_agent_centric/domains/agent-computer-use.md`

## Boundary

- Target repo is read-only. No commits, branches, issues, or files were modified.
- KnowlegeMap and production skill were not modified.
- This run writes only to `knowledge-archaeology-corpus` under `projects/webmcp/`.
- No Feishu write: no explicit user authorization.

# Job Manifest — ARCH-2026-11-22-001

- job_id: `ARCH-2026-11-22-001`
- project: `browser-use`
- repository: `https://github.com/browser-use/browser-use.git`
- mode: `refresh`
- priority: `A`
- run timestamp: `2026-11-22` (Asia/Shanghai)
- source layer: KnowlegeMap `origin/main` @ `2b7fb0a` (2026-11-22)

## Selection inputs

### KnowlegeMap state read

- `05_scan_logs/2026-11/2026-11-22_radar-watch_scan.md`: Memory 第二周 Day7 closeout. Explicitly says “本轮不新扫描，只审计已有产出并做交接”; result `Changed`. The audit marks candidate coverage and feedback coverage complete, but validation only “路径级”.
- `07_radar_watch/current_focus.md`: Memory focus closes on 2026-11-22; from 2026-11-23 the focus switches to **Computer Use / Browser week 1**. Day1 capability tree is not built yet; 2026-11-22 is Day0 baseline.
- Handover criteria: (1) action gating **location** matters more than strength; (2) verdict evidence must be **external** (execution ledger, not page copy); (3) receipt must be **verifiable** (target + time + hash + coverage).
- Existing browser anchors: `03_expansion_queue/candidates/[cand]computer-browser-use-2026.md` (A), `[cand]browser-harness-2026.md` (A), `[cand]webwright-2026.md` (B).

### Corpus state read

`knowledge-archaeology-corpus` `origin/main` @ `1b2b9fd` contains 63 project directories, including `browser-use`, `browser-harness`, `webwright`, `mcp`, `hy-memory`. The prior `browser-use` entry is:
- version `0.13.10`, commit `d8110c5ff87ccba887aaa726cdb780f2f84bef8d` (2026-09-18)
- run `ARCH-2026-09-22-001`, mode `initial`
- skill `knowledge-archaeology v3.2`

## Candidate ranking (not star-rank)

| Candidate | State | Evidence | Decision |
|---|---|---|---|
| `browser-use` | `REFRESH_REQUIRED` | prior 0.13.10 @ `d8110c5` (2026-09-18); current HEAD `e682e0d`, tag/version `1.0.8rc1` (2026-11-20); `git diff d8110c5..HEAD` = **546 files changed, +37,339 / −135,830**; engine switched from DOM action loop to Pydantic AI + persistent Python REPL + raw CDP | **SELECTED** |
| `microsoft/playwright-mcp` | `NEW` candidate | HEAD `2337aff0213d28db41d3e3daf48cee19b3d0bd82`, v0.0.86, 2026-11-18; not in corpus; browser-adjacent but lower architectural distance from existing browser-use/browser-harness | not selected |
| `browser-harness` | already archaeologized | v0.1.13 @ `afbcc38`; current version/change volume not re-verified this run | not selected |
| `webwright` | already archaeologized | prior commit `bc26750` (2026-08-03); current HEAD `35dbcf3`; no major version change observed | not selected |

## Why browser-use

1. **Knowledge value**: 1.0 is an engine-level rewrite, not an incremental release. The action authority moved from a predefined DOM action schema to model-written Python against a persistent REPL and raw CDP.
2. **Agent-AI engineering relevance**: it directly connects the Memory-week handover criteria — where gating lives, what counts as external evidence, and how receipts are verified — to a concrete browser runtime.
3. **Novelty**: the prior corpus describes a 0.x DOM loop, event-driven watchdogs, and a three-layer domain guard. Those conclusions cannot be carried into 1.0 without re-checking; several 0.x concepts (planner, DOM engine, pause/resume, watchdog set) are explicitly removed.
4. **Growth signal**: 546 changed files and a net deletion of ~135k lines in two months is a strong architectural discontinuity, not ordinary churn.
5. **Connection to existing corpus**: browser-use, browser-harness, and webwright already form a browser cluster in the corpus. A refresh preserves continuity and avoids opening a fourth medium-value project the day before a new focus week.
6. **Benchmark learning value**: the repository contains 18,492 lines of tests and a 25-file `tests/ci` suite built around real REPL workers and a real headless Chrome; even without running them this run, the test suite exposes concrete failure modes and boundary conditions.

## Job

```yaml
job_id: ARCH-2026-11-22-001
project: browser-use
repository: https://github.com/browser-use/browser-use.git
mode: refresh
priority: A
reason: >
  browser-use 1.0 is an engine-level rewrite (0.13.10 DOM action loop -> 1.0.8rc1 Pydantic AI
  + persistent Python REPL + raw CDP). 546 files changed and 135,830 lines deleted since the
  prior archaeology. It connects the existing browser cluster and the Memory-week handover
  criteria, and is the highest-value target on the eve of the Computer Use / Browser week.
source_entry: >
  07_radar_watch/current_focus.md (Memory week2 closeout, switch to Computer Use/Browser on 2026-11-23)
  + 05_scan_logs/2026-11/2026-11-22_radar-watch_scan.md (no new scan, audit and handoff)
  + 03_expansion_queue/candidates/[cand]computer-browser-use-2026.md (A)
  + corpus projects/browser-use/metadata.yaml (prior 0.13.10 run)
```

## Boundary

- Target repository is read-only; no tracked file was modified.
- Prior corpus `projects/browser-use/` is not overwritten; this refresh adds `archaeology-runs/ARCH-2026-11-22-001/` and reconciles the project-level package only after validation.
- No Feishu delivery; corpus push only.

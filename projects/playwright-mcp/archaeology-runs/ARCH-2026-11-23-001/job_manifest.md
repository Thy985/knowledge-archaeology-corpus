# Job Manifest — ARCH-2026-11-23-001

## Run identity

| Field | Value |
|---|---|
| job_id / run_id | `ARCH-2026-11-23-001` |
| project | `playwright-mcp` |
| repository | `https://github.com/microsoft/playwright-mcp.git` |
| mode | `initial` (NEW; no prior Corpus project) |
| priority | `A` |
| date | 2026-11-23 (Asia/Shanghai) |
| corpus slug | `projects/playwright-mcp/` |
| source_entry | KnowlegeMap `07_radar_watch/current_focus.md` + `08_agent_centric/domains/agent-computer-use.md` + `03_expansion_queue/candidates/[cand]computer-browser-use-2026.md` |

## Candidate ranking

Ranking follows Knowledge Value / Agent-AI Engineering relevance / Novelty / growth signal / recent release / connection to existing Corpus / prior archaeology / benchmark learning value — not star count.

| Rank | Candidate | State in Corpus | Evidence (KnowlegeMap / repo) | Decision |
|---|---|---|---|---|
| 1 | `microsoft/playwright-mcp` | **NEW** | v0.0.86 @ `2337aff` (2026-11-18); official MCP server; a11y-snapshot + ref-action route; directly maps to Computer Use tree A2/B1/D1/H1 and E2/F1 | **Selected** |
| 2 | `browser-use` | REFRESH completed 2026-11-22 | 1.0.8rc1 @ `e682e0d`; REPL + raw CDP route; 546 files changed | Not repeated |
| 3 | `browser-harness` | COMPLETED | v0.1.13 @ `afbcc38`; change volume not re-measured | Lower novelty today |
| 4 | `webwright` | COMPLETED | last commit `bc26750` (2026-03-03); current HEAD `35dbcf3`; no major version signal | Lower novelty today |

Other browser candidates (`Playwright CLI`, `Chrome DevTools MCP`) are relevant but not the best single deep job for Day 1: Playwright CLI is the official token-efficient coding-agent comparator, and Chrome DevTools MCP overlaps the same control plane; both are deferred to later radar slots.

## Selection rationale

1. **Novelty / no duplication**: `playwright-mcp` is absent from the 63 existing Corpus projects; this is an initial archaeology, not a refresh.
2. **Architectural contrast**: it is the clearest official implementation of the *structured accessibility snapshot + element ref* route, contrasting with `browser-use` 1.0's *persistent Python REPL + raw CDP* route. The two form the first comparison axis of the Computer Use week.
3. **Direct capability-tree hit**: A2 accessibility tree, B1 abstract actions, D1 persistent session, H1 trajectory; plus E2 local control-plane exposure, D2 login-state inheritance, F1 page content as untrusted input.
4. **Boundary discipline**: repository is small (44 tracked files), so a full source-level snapshot is feasible in one day; the implementation dependency is pinned and auditable.
5. **Benchmark learning value**: official Microsoft server, npm + MCP Registry distribution, a documented Docker long-lived mode, and a default tool surface that includes an explicitly RCE-equivalent tool — high learning value for agent safety and control-plane design.

## Scope and hard boundaries

- Target repository: **read-only**. No branch, commit, issue or file is modified.
- `KnowlegeMap`: **read-only** in this run.
- `knowledge-archaeology-skill`: **read-only**; no Skill Evolution proposal is raised unless a systemic defect survives validation.
- `knowledge-archaeology-corpus`: **write** only under `projects/playwright-mcp/`; history runs are never overwritten.
- One deep project for this date. No second project.
- Default delivery is Corpus push only; Feishu delivery requires explicit user authorization and is **not** performed.
- Facts must trace to the repository or its pinned implementation dependency. Dependency facts are labelled separately from repository facts.

## Expected outputs

- `snapshot_artifact.md`
- `package/00_overview.md` … `package/06_validation/`
- `package/run_metadata.yaml`
- Independent audit + reconciliation
- Corpus branch / PR for `projects/playwright-mcp/`

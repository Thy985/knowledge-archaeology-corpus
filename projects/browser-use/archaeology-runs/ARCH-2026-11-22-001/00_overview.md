# 00 · Overview — browser-use 1.0.8rc1

## Run identity

| Field | Value |
|---|---|
| run_id | `ARCH-2026-11-22-001` |
| project | `browser-use` |
| repository | `https://github.com/browser-use/browser-use.git` |
| mode | `refresh` (prior: 0.13.10 @ `d8110c5`, run `ARCH-2026-09-22-001`) |
| priority | A |
| commit | `e682e0dc02d77a307a5b2aedf17142c2cd69af5b` |
| branch | `main` |
| tag | `1.0.8rc1` (release candidate, not GA) |
| commit time | `2026-11-20T14:48:08-08:00` |
| package version | `1.0.8rc1` |
| skill version | `knowledge-archaeology v3.2` |
| license | MIT (prior corpus metadata's `Apache-2.0` is wrong; see C-05) |
| package date | 2026-11-22 (Asia/Shanghai) |

## What changed since the last archaeology

`git diff d8110c5..HEAD` reports **546 files changed, 37,339 insertions, 135,830 deletions**. This is an engine-level rewrite, not an incremental feature release:

- 0.x per-step DOM action selection and planner are gone.
- 1.0 executes model-authored Python in a persistent REPL worker on raw CDP.
- The browser lifecycle is reorganized around cloud/chromium/chrome/cdp, a shared daemon, Guard, and keep-alive.
- Delivery, compaction, judge, telemetry, auth, and MCP have their own 1.0 implementations.
- Most 0.x tuning knobs either raise `TypeError` or warn and are ignored.

The prior 0.13.10 package remains valid as history. It is **not** a current 1.0 fact base.

## What this package contains

| Volume | Content |
|---|---|
| `00_overview.md` | run identity, delta, key conclusions, limits |
| `01_project-layer/project_layer.md` | project map: modules, entry points, structures, state, tests, config, permissions, dependencies |
| `02_engineering-knowledge/ek_graph.md` | 35 EK with evidence, boundaries, and six edge types |
| `03_knowledge-layer/knowledge_layer.md` | 12 KO: 9 L3 patterns, 2 L4 models, 1 L5 methodology |
| `04_flow-atlas/flow_atlas.md` | 7 flows with condition-bearing edges |
| `05_candidates/candidates.md` | 8 open/corrected candidates |
| `06_validation/evidence_map.md` | EV-01 to EV-95 plus limitations |
| `06_validation/independent_audit.md` | independent reconstruction and verdicts |
| `06_validation/reconciliation.md` | accepted corrections and scope changes |
| `run_metadata.yaml` | machine-readable run record and quality metrics |

## Five load-bearing conclusions

1. **Execution substrate moved from action objects to code cells.** The runtime’s control points are now process start, env allowlist, token-authenticated loopback, one-cell lock, process-group kill, and return-channel redaction (EK-04 至 EK-09; KO-01).
2. **Domain policy is layered and mode-dependent.** Cloud/chromium/profile-chrome have Guard; attached `Browser('chrome')` has no Guard and sweeps only agent tabs; MCP is a separate plane. Guard disconnects are fail-open (EK-12 至 EK-15; KO-02).
3. **Secrets are redacted, not isolated.** The worker runs with user permissions and can read files, environment, screenshots, live view, and recordings; short secrets and `open()` writes are residual channels (EK-09 至 EK-11; KO-03).
4. **Delivery is best-effort, not transactional.** A budget turn, one repair, a one-shot split check, and a last-request exemption produce a deliverable, but not a guarantee (EK-18 至 EK-20; KO-05).
5. **Status, success, and judge are independent.** `completed` can coexist with `success=False`; judge failure does not fail the run; judge is advisory (EK-21, EK-30; KO-06).

## What is not claimed

- No test was executed. Collection failed because `pydantic_ai` is not installed. There are no pass/fail or coverage percentages.
- No dependency was installed; `browser-harness==0.1.13` internals were not audited.
- No Chrome/Chromium was launched; no LLM/cloud key was used; no cloud browser, live view, CAPTCHA, proxy, or device-code flow was exercised.
- No benchmark score is asserted. README benchmark references were not converted to numbers.
- `1.0.8rc1` is an rc. No GA claim.
- Cross-project Patterns are not promoted to Principles. KO-10 and KO-11 are `Hypothesis`; KO-12 is `Cross-project validation pending`.

## Reading order

Start with `01_project-layer` for the map, then `02_engineering-knowledge` for evidence-bearing detail, then `03_knowledge-layer` for abstraction. Read `04_flow-atlas` when checking a specific edge. Read `05_candidates` before quoting any cross-project claim. Read `06_validation` before treating any KO as validated.

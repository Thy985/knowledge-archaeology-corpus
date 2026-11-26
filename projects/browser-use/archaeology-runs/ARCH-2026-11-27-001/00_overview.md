# 00 Overview — browser-use 1.0.8 refresh

- **Project**: `browser-use` (`https://github.com/browser-use/browser-use.git`)
- **Run**: `ARCH-2026-11-27-001`
- **Job**: `KA-2026-11-27-001` (priority A, mode `refresh`)
- **Baseline**: `ARCH-2026-09-22-001`, version `0.13.10 @ d8110c5`
- **Target**: tag `1.0.8 @ 753d629d76eb2b670f4cf467f3355bbe750f6845` (2026-11-21)
- **Skill**: knowledge-archaeology v3.2
- **Scope**: 146 tracked files, 35,428 Python lines, 18,492 test lines; 964 non-Chrome tests passed locally.

## Headline

1.0.8 is an **architecture replacement**, not a feature refresh. The model no longer selects from a DOM/action schema; it writes Python in a persistent REPL over raw CDP. The intermediate representation is gone, and with it the action-level validation surface. Governance now lives at three other layers: the parent/worker/daemon execution split, the browser attach/domain layer, and the budget/delivery gate.

The most important finding is not “code is more powerful.” It is that **the security vocabulary no longer matches the control surface**. The repo is unusually explicit: secret redaction is not a sandbox, domain rules are not a sandbox, the worker runs as the user, and a local Chrome debug port is an unauthenticated loopback capability. A reader who carries over 0.x assumptions (“the security watchdog checks every navigation”) will misread 1.0.

## What was produced

| Layer | Content |
|---|---|
| Snapshot | commit, size, tests, module map, initial authority map |
| EK | 55 engineering facts/mechanisms across architecture, loop, delivery, context, security, interfaces, tests |
| KO | 12 knowledge objects: 10 L3 patterns, 2 L4 models; no L5 promoted from one project |
| Flow Atlas | 7 flows (Control, State, Data, Evidence, Authority, Memory, Policy) with edge-level conditions |
| Candidates | 5 within-project open questions, 8 cross-project hypotheses; none promoted |
| Validation | 9 CONFIRMED, 3 PARTIALLY_CONFIRMED, 0 contradicted; 3 errors corrected; 3 benchmark cases |

## The five job questions, answered

1. **What did 1.0 delete and keep?** It deleted the DOM serializer, typed action model, action registry, event-bus watchdogs, and 0.x loop/planning options. It kept the public `Agent`/`Browser`/`Tools` facade, browser lifecycle, domain intent, and some compatibility names. Control is now “model writes Python → REPL executes → finish delivers,” with a delivery-only turn near budget.
2. **What are the hard boundaries of the persistent REPL?** Hard: separate process group, token-authenticated loopback, per-cell timeout, kill/restart, env allowlist, workspace cwd. Soft: secret redaction (selected channels only), file access (user-readable files), screenshots/`js()` (unredacted), local debug port (no endpoint auth).
3. **Is local Chrome still an ambient local control plane?** Yes. `--remote-debugging-port=0` randomizes the port but adds no authentication; any local account that can reach loopback can attach while the browser runs. Profile copies carry cookies. The code prevents port guessing and hardens the daemon directory; it does not authenticate DevTools.
4. **What is the judge’s authority?** The agent’s `finish(success=...)` is the only input to `is_successful()`. The judge is a separate request from the same model over the same trace, often sharing the actor prefix for prompt cache. It is a second opinion, not independent verification; its failure never changes the run.
5. **Which old KOs survive?** The 0.x “controlled loop / watchdog / DOM observation” KOs are **historical**, not inherited. The durable cross-architecture invariants are: budgeted degradation, delivery gating, layered domain intent, and the distinction between agent self-report and external verdict. The specific 0.x mechanisms (loop detector, EventBus, DOM serializer) do not survive.

## Quality metrics

| Metric | Value |
|---|---|
| EK entries | 55 |
| KOs | 12 (10 L3, 2 L4, 0 L5) |
| Flow types | 7 |
| Flow edges traced | 59 |
| Candidate hypotheses | 13 (5 within-project, 8 cross-project) |
| Validation verdicts | 9 C / 3 PC / 0 contradicted |
| Corrected errors | 3 (step default, judge edge, attach approval) |
| Tests executed | 964 passed, 3 skipped, 140 deselected |
| Test functions in `tests/ci` | 429 |
| Coverage status | core runtime/security/delivery/context: strong; cloud/MCP/CLI late paths and real Chrome: partial |
| Epistemic status | no Hypothesis presented as Fact; no Candidate promoted to Principle |

## Reading order

1. `01_project-layer/project_layer.md` — architecture and 0.x→1.0 delta.
2. `02-engineering-knowledge/engineering_knowledge.md` — 55 EK facts with source anchors.
3. `03-knowledge-layer/knowledge_layer.md` — L1→L4, KO derivation and boundaries.
4. `04-flow-atlas/flow_atlas.md` — seven flows, edge conditions.
5. `05-candidates/candidates.md` — open questions and cross-project hypotheses.
6. `06-validation/independent_findings.md` and `validation_report.md` — blind reconstruction, verdicts, errors, benchmarks.

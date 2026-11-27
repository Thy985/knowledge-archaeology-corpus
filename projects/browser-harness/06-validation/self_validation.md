# 06 Validation — self-check (pre-independent)

> This is the producing agent's own gate, before the Independent Auditor runs. It is not a substitute for Stage 5.
> No gate was marked green by lowering evidence standards. Items that could not be checked are listed as open.

## Quality metrics

| metric | value | basis |
|---|---|---|
| EK nodes | 58 | parsed `02-engineering-knowledge/ek_graph.md` |
| EK link edges | 144 | `links:` lines, 2.48 avg out, min 1, max 4, 0 orphans |
| KO | 10 | 8 L3, 2 L4, 0 L5 |
| Candidate | 8 | 1 Observation, 6 Hypothesis, 1 Cross-project Hypothesis |
| Evidence rows | 58 | EV-01…EV-58 |
| Distinct file:line anchors | 129 | all target-repo anchors resolve and are in range; external `browser_use/...` checked in the 1.0.8 clone |
| Dead anchors | 0 in target repo | automated line-range check |
| Tests inspected | 404 `def test_` | test files read for daemon/admin/helpers/run; suite **not executed** |
| Core modules inspected | 12/17 `src/browser_harness/*.py` by name | control plane complete; video/auth/macos/recorder peripheral not line-by-line |
| Runtime execution | none | no install, no browser, no CDP |

## Truth gate

Six fixed probes, all re-read from the repo:

| probe | expectation | result |
|---|---|---|
| `GUARDED` membership | exactly 3 Target methods | PASS (`daemon.py:37`) |
| browser session owner binding | absent | PASS: `browser_sessions` only (`:1226-1228`) |
| sub-agent refusal scope | 2 helpers | PASS (`helpers.py:114-118,441-445,521-525`) |
| idle release closes nothing by default | `BH_CLOSE_IDLE_TABS` unset | PASS (`:726-750`; `test:970`) |
| newer protocol never stopped | `daemon-outdated` | PASS (`admin.py:422-446`) |
| downstream pin | 0.1.13, no owner API | PASS in 1.0.8 clone |

Contradictions found inside the package: 0. (Cross-document consistency checked: EK IDs in KO clusters all exist; EV IDs all exist.)

## Coverage gate

- Control-plane claim coverage: every KO maps to at least 3 EK and every EK appears in at least one KO or candidate. 58/58 EK referenced.
- Domain coverage: conversation/daemon/admin/IPC/report/telemetry/MCP read. Video rendering (`video.py`, `video_render.py`, 1,276 lines) and macOS/auth-specific paths (569 + 196 lines) are outside this run's questions; no KO claims depend on them.
- Question coverage from the Job Manifest: all five questions answered with code anchors, but Q4 ("isolation and privacy hold for raw CDP too") is answered **partially with a counterexample**, not as a clean yes.
- Runtime coverage: 0. There is no executed browser/CDP probe; all conclusions are static.

## Flow gate

Seven flow files exist. Every critical edge names a symbol and a condition. The weakest edges:
- F-03 screenshot path: helper function shape inferred from `helpers.py:389-420`, not exercised.
- F-06 captcha memory: `captcha_active/finished/taken` field list read, internal mutation not traced line-by-line.
- F-07 policy: restart policy is re-evaluated per call; no persistent policy object. That is a finding, not a missing edge.

## Abstraction gate

- No L5. L4 only for KO-09 (authority-surface completeness), explicitly marked single-project.
- KO-10 is L3, not L4: one upstream/downstream pair is not enough for a cross-project model.
- Every `derivation.facts` equals its cluster; no KO cites an EK outside its cluster (checked by script).
- No Hypothesis is written as Validated Pattern: C-01/C-03/C-07/C-08 are marked Hypothesis; KO-02 is code-fact, not runtime-fact.

## Counterexample gate

Recorded counterexamples:
1. KO-01: browser-level CDP escape (C-01).
2. KO-02: `cdp()` bypass for sub-agents (C-03).
3. KO-04: `last` depends on `info` freshness; not tested.
4. KO-05: `tab-released` is one-shot.
5. KO-08: telemetry key filter is not a text scrubber.
6. KO-09: exploitability depends on sid exposure; no execution.
7. KO-10: browser-use's per-Browser daemon is a legitimate alternative, bounding the claim.

## Epistemic gate

- Fact: code text at a pinned commit.
- Observation: single-source anchors (EV list).
- Hypothesis: C-01, C-03, C-04, C-05, C-07, C-08; C-06 cross-project.
- Pattern: 8 KO L3.
- Cognitive Model: KO-09 (L4).
- Principle: none (L5 empty).

## Self verdict

**SUPERSEDED by independent validation; initial self-check was CONDITIONAL PASS.** The package is internally consistent, all target anchors exist, and the main claim is falsifiable. It is not a full PASS because: (a) no runtime execution, (b) browser-level exploit unexecuted, (c) video/macOS domains uncovered, (d) tests not run. The Independent Auditor should attack C-01, the `last` invariant, and the KO-09 L4 promotion first.

## Post-independent correction (2026-11-28)

The independent auditor found issues this self-check missed:

1. The quoted README sentence in EK-56/OV-4 is not present in the v0.1.15 snapshot. The document anchor is removed; the code gap is retained.
2. KO-09 was labeled L4 on single-project evidence; it is downgraded to L3, and the KO count is 9 L3 + 1 L4, not 2 L4.
3. EK-10 overgeneralized a function's warning design into CLI runtime behavior; corrected.
4. Unit test files are 16, not 17.
5. Added EK-59 (undocumented 0.1.15 telemetry trim), EK-60 (owner JSON mode), and C-09.

The authoritative validation record is `06-validation/independent_validation.md`; `reconciliation.md` lists every correction. The initial claim `contradictions: 0` is withdrawn.

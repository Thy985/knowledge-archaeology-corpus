# Reconciliation — after independent validation

- Run: `ARCH-2026-11-28-001`
- Auditor outputs: `06-validation/independent_findings.md` (frozen before comparison), `06-validation/independent_validation.md`.
- Rule followed: fix the Corpus artifact, never the production Skill. The target repo and package originals were not altered beyond the listed corrections.

## Verdict summary

| label | count |
|---|---:|
| CONFIRMED | 79 |
| PARTIALLY_CONFIRMED | 7 |
| DOWNGRADED | 1 |
| OVER_GENERALIZED | 1 |
| MISSING | 6 |
| CONTRADICTED | 3 |
| NEEDS_HUMAN_REVIEW | 3 |
| total judged claims | 94 |

## Corrections applied to the package

| # | Auditor finding | Change | File |
|---|---|---|---|
| R-1 | E-1 / OV-4: quoted README sentence "Isolation and privacy hold for raw CDP too" is absent from the v0.1.15 tree | Removed the quote as evidence; EK-56 now cites `SKILL.md:52` availability only and states the browser-level code gap independently. Overview adjusted. | `00_overview.md`, `02-engineering-knowledge/ek_graph.md`, `06-validation/evidence.md` |
| R-2 | E-2 / EK-10: "warns on every call" is function design, not CLI runtime | Rewrote to: no once-flag in the function, but only production wiring is `run.py:471`, once at startup. | `02-engineering-knowledge/ek_graph.md` |
| R-3 | E-3 / KO-09: single-project evidence labeled L4 | Downgraded to L3 Pattern; cross-project generalization stays Hypothesis C-06. Counts changed to 9 L3 + 1 L4. | `03-knowledge-layer/knowledge_layer.md`, `00_overview.md`, `run_metadata.yaml` |
| R-4 | PL-5: unit test file count was 17, actual 16 | Corrected to 16 unit + 4 integration. | `01-project-layer/project_layer.md` |
| R-5 | M-1: 0.1.15 telemetry trim missing | Added EK-59, candidate C-09, EV-59. | `02`, `05`, `06` |
| R-6 | M-2: owner JSON mode not in EK layer | Added EK-60, linked from C-05, EV-60. | `02`, `05`, `06` |
| R-7 | Anchor drift (EK-06/37/41/43, Flow F-01/F-02/F-05 and several KO refs) | Inner line numbers were not bulk-rewritten; the symbols remain resolvable and the drift is recorded here as a known defect. A later refresh should use symbol anchors rather than raw line numbers. | this file |

## Accepted limitations (not corrected)

- M-3 (worker monkeypatch of helpers) is not represented as an EK/Flow; it is an alternate path in the downstream, not in browser-harness. Added to open questions below.
- M-4 peripheral domains (video, video_render, macos, auth, recorder) remain un-nodeified. The package states this coverage limit.
- M-5 MCP tool surface and `naming.py` are not nodeified. Scope limit, no claim depends on them.
- M-6 the 0.1.13 unavailable-capability list is summarized in EK-58, not itemized. The auditor's itemization is stronger; a future refresh should absorb it.
- Runtime probes remain 0. C-01/C-03/C-07 stay NEEDS_HUMAN_REVIEW.
- Tests were not executed; 404 `def test_` is a count, not a pass claim.

## Post-reconciliation status

- EK: 60 nodes (was 58; +EK-59, +EK-60).
- KO: 10; 9 L3, 1 L4, 0 L5.
- Candidates: 9 (was 8; +C-09).
- Evidence: 60 rows (EV-01…EV-60).
- Validation: `PARTIAL_PASS_WITH_DOWNGRADE`. The core structural findings survive; the L4 label and one document anchor did not.
- The package should be read with `06-validation/independent_validation.md` as the authoritative validation record, not the pre-independent `self_validation.md`.

## Open questions for the next refresh

1. Is the browser-level sid ever exposed to a second owner in practice? (C-01)
2. Does `self.info` staleness ever make the last-page guard close the last page? (KO-04)
3. Did 0.1.15's telemetry trim ship in PR #1077, and why no changelog? (C-09)
4. Does browser-use ever wrap `cdp()` such that its own owner semantics are bypassed? (M-3)
5. Does the host set `BH_OWNER` for Claude sub-agents, or does the shared-parent path remain the default? (C-03)

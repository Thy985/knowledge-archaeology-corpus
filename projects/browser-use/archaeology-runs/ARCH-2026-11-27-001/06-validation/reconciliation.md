# Reconciliation — ARCH-2026-11-27-001

The auditor did not modify the primary package. Reconciliation is applied only to the final Corpus artifact. The changes below are explicit, so a reader can see where the final package differs from the first draft.

## Corrections applied

| ID | Finding | Original draft | Final text | Affected |
|---|---|---|---|---|
| COR-01 | Effective step default | `Session.max_steps=1000` read as the default | `Agent.run(max_steps=100)` passes 100; 1000 is an internal default | EK-13, KO-03 |
| COR-02 | Verdict flow | Diagram edge `judgement → is_successful()` | `is_successful()` reads only `success`; judge edge is dashed “not read” | 04 EF, KO-05 |
| COR-03 | Attach approval | Implied every `Browser('chrome')` attach needs an approval prompt | Approval is conditional; launched Chromium/profile copy has no gate | EK-44, KO-08 |
| COR-04 | Judge independence | Working phrase “independent judge” | “separate request from the same model over the same trace; not independent” | KO-05, 00, 06 |
| COR-05 | License baseline | Baseline summary says Apache-2.0 | 1.0.8 LICENSE/pyproject are MIT; baseline discrepancy recorded, not resolved | run_metadata `license_note` |
| COR-06 | `use_vision=False` judge asymmetry | Not mentioned | `_run` gates `judged` on `use_vision`, `_judge()` does not; documented as an asymmetry | 06 omissions |

## No changes to

- The target repository: still detached at `753d629d...`, worktree clean.
- KnowlegeMap: read-only.
- The production skill: read-only; no Skill Evolution proposal.
- Historical corpus: `projects/browser-use/00_overview.md` … `06_validation` remain the 0.13.10 baseline; the new run is stored under `archaeology-runs/ARCH-2026-11-27-001/`.
- The old run directory `archaeology-runs/ARCH-2026-09-22-001/`: preserved; its metadata is copied into the baseline run directory for provenance.

## Scope changes from audit

1. **KO-08 downgraded from CONFIRMED to PARTIALLY_CONFIRMED**: the code has no endpoint auth, but “any local account can attach” is documented, not live multi-user tested.
2. **KO-11 and KO-12 marked PARTIALLY_CONFIRMED**: they are L4 models, not code facts; the “governance must move” and “often conflated” claims are inferences.
3. **Coverage gate marked PARTIAL**: no real Chrome, no live cloud, no exhaustive MCP/CLI trace.

## Result

- 12 KOs: 9 CONFIRMED, 3 PARTIALLY_CONFIRMED.
- 0 CONTRADICTED, 0 DOWNGRADED, 0 OVER_GENERALIZED, 0 MISSING.
- 3 material errors corrected, 1 wording downgrade, 2 documentation additions.
- No L5 principle promoted.

# 06 · Reconciliation — ARCH-2026-11-22-001

> Scope: apply only corrections that are supported by the repository and the Auditor’s read-only reconstruction. No KO was promoted, no evidence threshold was lowered, and no historical Corpus result was modified.

## Accepted corrections

| ID | Auditor finding | Action | Files changed |
|---|---|---|---|
| R-01 | Judge ordering: package placed judge before browser shutdown; code captures final page, shuts down, then judges | accepted; corrected lifecycle order | `00_overview.md` (no order change needed), `01_project-layer/project_layer.md`, `02_engineering-knowledge/ek_graph.md` EK-01/EK-21, `04_flow-atlas/flow_atlas.md` control/state, `06_validation/evidence_map.md` EV-21 |
| R-02 | Split-delivery scope: source comment says “once per run”, implementation skips only the same request step | accepted; downgraded from “once/run” to “per request step”; recorded comment/code mismatch and missing second-step test | `02_engineering-knowledge/ek_graph.md` EK-20, `03_knowledge-layer/knowledge_layer.md` KO-05, `04_flow-atlas/flow_atlas.md` F-C-09, `06_validation/evidence_map.md` EV-30 |
| R-03 | Guard start guarantee omitted attached `Browser('chrome')`, which returns before Guard creation | accepted; scope limited to modes that instantiate Guard | `02_engineering-knowledge/ek_graph.md` EK-14, EK-13 boundary |
| R-04 | “Guard does not see `Runtime.evaluate`” was too broad; Guard covers JS location changes via request events, while attached Chrome has no Guard | accepted; clarified the gap as attached-mode absence, not a general Guard blind spot | `02_engineering-knowledge/ek_graph.md` EK-13 |
| R-05 | “One key covers model/cloud/CLI/MCP” is a doc/product claim; code only verified env/stored/fallback order | accepted; EK-28 claim narrowed and marked as docs intent for the unexercised surfaces | `02_engineering-knowledge/ek_graph.md` EK-28 |
| R-06 | Prior Corpus metadata says `Apache-2.0`; repository is MIT at HEAD and `d8110c5` | accepted for the refresh metadata; do not rewrite the prior run, record the prior defect in this run | Corpus `projects/browser-use/metadata.yaml` (new refresh run metadata; prior run untouched) |
| R-07 | No test execution; collection failed on missing `pydantic_ai` | accepted as limitation; no pass/fail/coverage number is written | `run_metadata.yaml`, `00_overview.md`, `06_validation/evidence_map.md` |

## Rejected or narrowed changes

| Proposed change | Decision | Reason |
|---|---|---|
| Promote KO-10/KO-11 to Principle | **rejected** | One-project evidence; cross-project validation absent. Keep `Hypothesis`. |
| Promote KO-12 to validated methodology | **rejected** | No cross-project test; keep `Cross-project validation pending`. |
| Rewrite the prior 0.13.10 run to 1.0 facts | **rejected** | Corpus history is immutable; the refresh adds a new run directory. |
| Claim all Browser modes have Guard | **rejected** | Attached `Browser('chrome')` has no Guard; `Browser(cdp_url=...)` depends on the external owner. |
| Claim secrets are isolated | **rejected** | Docs and code admit worker access to files/environment; redaction is channel-level. |
| Claim tests pass | **rejected** | No execution; only static test contracts were read. |
| Treat split delivery as once per run | **rejected** | Code is step-scoped; comment is stale. |
| Treat judge as a success gate | **rejected** | Judge is advisory and does not change `status`. |

## Scope changes

1. **`Browser('chrome')`** is split into two cases: attach (no Guard, layer 1 + agent-tab sweep) and named profile (owned Chromium, Guard). The package must not collapse them.
2. **MCP** is described as a separate execution plane, not “Agent with a different transport”. Cloud MCP has no Guard; domain-configured MCP can refuse cloud admin calls but does not guard all cloud traffic.
3. **“One key”** is documented intent; only the env/stored/fallback chain is a code fact in this run.
4. **Validation** is `PARTIAL`, not `PASS`. The cause is missing runtime/dependency evidence plus two corrected contradictions.

## Reconciliation result

- Audit verdicts: 18 CONFIRMED, 4 PARTIALLY_CONFIRMED, 2 CONTRADICTED, 3 MISSING, 1 NEEDS_HUMAN_REVIEW; 0 DOWNGRADED, 0 OVER_GENERALIZED.
- Corrections applied: 7.
- KOs changed: KO-05 wording (split scope only). No KO added or removed.
- EKs changed: EK-01, EK-13, EK-14, EK-20, EK-21, EK-28.
- Flow edges changed: F-C-09, F-C-10, F-C-11, F-S-11, F-S-12, F-E-09.
- Historical Corpus: untouched; refresh writes a new `archaeology-runs/ARCH-2026-11-22-001/`.
- Skill Evolution: no proposal this run. C-07 (schema L5 question) is a candidate only; no production Skill file was modified.

## Remaining open items

1. Install dependencies and run `python3 -m pytest tests/ci` to obtain a real baseline.
2. Read `browser-harness==0.1.13` for helper-level authority.
3. Establish the `1.0.8rc1` → `1.0.8` GA delta.
4. Run B-01 enforcement matrix on a benign task.
5. Human review of the secret/Guard threat model.

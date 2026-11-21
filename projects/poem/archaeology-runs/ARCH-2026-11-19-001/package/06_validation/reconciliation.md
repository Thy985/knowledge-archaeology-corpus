# Reconciliation — PoEM (ARCH-2026-11-19-001)

- pipeline order: Stage 4 package → Stage 5 independent blind reconstruction → this reconciliation → Stage 6 corpus write
- rule: reconcile corrections into the Corpus artifact; do not edit the production skill; no historical run is overwritten.

## 1. Decisions applied

| id | source verdict | change applied to package | artifact |
|---|---|---|---|
| R-01 | CONFIRMED | core architecture, gate coverage, proof identity, mock caveat, cost scaling retained as-is | 00–04 |
| R-02 | DOWNGRADED | KO-02 layer L4 candidate → **L3**; keep key/write/storage/sequence conditions; cross-project pending | 03 |
| R-03 | DOWNGRADED | KO-05 layer L4 candidate → **L3**; content-vs-execution contrast retained as within-project pattern | 03 |
| R-04 | NEEDS_HUMAN_REVIEW | KO-09 layer L3 → **L2 Observation**; epistemic status Observation/Hypothesis; label NEEDS_HUMAN_REVIEW | 03 |
| R-05 | MISSING (OM-01) | add **EK-46** clean/poisoned trial pairing by index across separate stochastic runs | 02 |
| R-06 | MISSING (OM-02) | add **EK-47** LangChain port strips EXTERNAL provenance before Sentinel scoring | 02 |
| R-07 | MISSING (OM-03) | add **EK-48** AnomalyDetection silently no-ops without a bound reference | 02 |
| R-08 | MISSING (OM-04) | add **EK-49** ledger accepts duplicate/out-of-order events with a valid chain | 02 |
| R-09 | MISSING (OM-05) | no new ID: OM-05 is already covered by **EK-03** (retrieve-all fallback); statement retains the cross-subject contamination consequence | 02 |
| R-10 | CONFIRMED | retain **EK-45** (`attack.py` vs `scenarios.py`); matrix attribution stays `scenarios.py` | 02 |
| R-11 | CONFIRMED | retain 10 counterexamples CX-01…CX-10; do not remove null mock results | 06/self-validation |

## 2. Scope tightening

- `KO-02` keeps “key custody + write capability + storage control” but adds **sequence discipline** (EK-49), because a valid MAC chain does not prevent duplicate or out-of-order events.
- `KO-05` keeps the trust-boundary contrast but no longer claims a general cognitive model; it is a within-project pattern.
- `KO-09` is no longer counted as Generalized Knowledge. It remains an observation over author-recorded result files; no live model was run.
- `injection_lab/` remains out of scope.

## 3. Final metrics after reconciliation

| Metric | Value |
|---|---|
| Engineering Knowledge | **49** (EK-01…EK-49) |
| EK link declarations | **156** |
| Average out-degree | **3.18** |
| Isolated EK | 0 |
| Dangling link targets | 0 |
| Knowledge objects | **10** (9 L3 + 1 L2 observation) |
| L3 | 9 |
| L4 | 0 (2 former L4 candidates downgraded) |
| L5 | 0 |
| KO aggregation rules | 10/10 |
| Flow edges | 49 (7/7 classes) |
| Candidates | 14 |
| Counterexamples | 10 (CX-01…CX-10) |
| Independent verdicts | 7 CONFIRMED / 2 DOWNGRADED / 1 NEEDS_HUMAN_REVIEW (KO); 11 CONFIRMED / 1 PARTIALLY_CONFIRMED (EK spot checks) |

## 4. Checks before corpus write

| Check | Method | Result |
|---|---|---|
| schema | EK requires id/category/layer/epistemic/statement/evidence/links; KO requires aggregation_rule | pass |
| duplicate | duplicate EK ids and duplicate KO headings | 0 duplicates |
| evidence reference | every `file:line` exists and line ≤ file length | pass |
| link integrity | every link target resolves to EK-01…EK-49 | pass |
| orphan | every EK has ≥1 inbound or outbound edge | pass |
| KO traceability | every KO EK ref resolves | pass |
| count consistency | 00 / 02 / 03 / metadata / self-validation | synchronized |
| history | no previous corpus run path is modified | pass |

## 5. Outcome

**ACCEPTED WITH LIMITATIONS.**

The central finding survives: PoEM separates execution authority from reasoning content and vetoes one gated action based on a tamper-evident ledger, not on memory text. The limitations are explicit: gate coverage, freshness, key custody, in-memory durability, O(N) verification, and no live-model replication. The two former L4 claims are not promoted; the capability claim awaits human review and a real rerun.

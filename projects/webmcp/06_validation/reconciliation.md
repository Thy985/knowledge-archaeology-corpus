# Reconciliation — audit corrections applied to the package

> Input: `independent_audit.md` (30 findings).
> Output: package corrections. The original EK/KO claims are not deleted; corrections are applied in place where the claim was wrong, and recorded here.
> Rule: correct the **artifact**, not the production Skill.

## 1. Applied corrections

| Finding | Change | File | Before | After |
|---|---|---|---|---|
| A-07 CONTRADICTED | Fixed error taxonomy | `02_engineering-knowledge/ek_graph.md` EK-07 | “five categories including InvalidStateError” | “5 execution failure reasons → 4 exception names; preconditions add InvalidStateError/NotAllowedError/NotSupportedError/SecurityError/TypeError” |
| A-07 | Fixed KO claim | `03_knowledge-layer/knowledge_layer.md` KO-L1-05 | “only five categories … InvalidStateError” | “tool-execution table 5→4; preconditions add 5 exception types” |
| A-25 DOWNGRADED | Scoped the L4 model | `03_knowledge-layer/knowledge_layer.md` KO-L4-02 | “In WebMCP, authority is positional” | “For in-page callers, authority is positional; built-in agent is a separate path” |
| A-25 | Lowered confidence | KO-L4-02 table | 0.68 | 0.62 |
| A-30 MISSING | Added candidate | `05_candidates/candidates.md` C-14 | absent | `self.modelContext` vs `[Exposed=Window]` contradiction recorded |
| Count | Updated metrics | `00_overview.md` | 51 edges / 12 candidates | 55 edges / 14 candidates |

## 2. Not changed, with reasons

| Finding | Verdict | Why no change |
|---|---|---|
| A-02 description length | CONFIRMED | Package already says title/description unbounded in C-07; no claim said otherwise |
| A-15 observation no filter | CONFIRMED | Package says “algorithm collects … without applying filters”; downstream implementation-defined caveat is present |
| A-26 continuation proposal | PARTIAL | Package marks it Hypothesis / non-normative; no upgrade |
| A-28 service worker proposal | PARTIAL | Package marks it Hypothesis / non-normative; C-11 already covers routing |
| A-29 status document | PARTIAL | Package marks implementation status as `[META]` and Observation, not Fact |

## 3. Epistemic status changes

| Object | Before | After | Reason |
|---|---|---|---|
| KO-L4-02 | Hypothesis 0.68 | Hypothesis 0.62, scoped to in-page | Built-in agent path is an exception |
| KO-L1-05 | Fact 0.92 | Fact 0.92, corrected wording | No confidence change; scope clarified |
| C-14 | — | Contradiction 0.70 | New finding from audit |

No L3/L4 was promoted. No Fact was upgraded to Principle.

## 4. Residual disagreements

1. **Whether the built-in agent observation is a “bypass” or a “separate path”**: the package calls it a bypass in flow F-A (B-A-1), the audit calls it a separate path. Reconciliation keeps both: it is a bypass **relative to the in-page three-gate model**, and a separate path **in the spec’s structure**. Both labels are accurate at different levels.
2. **Whether `toolactivated` timing is a defect**: package calls it a design constraint, not a bug. Audit agrees it cannot be a pre-gate, but does not call it a defect. No change.
3. **Whether the spec’s silence on auth is a gap or a delegation**: package says delegation (KO-L2-05), audit says gap in coverage. Reconciliation keeps “delegation” as the design reading and lists “no per-tool consent” as C-06.

## 5. Validation gate after reconciliation

| Gate | Before | After | Note |
|---|---|---|---|
| Truth | 0.91 | 0.93 | Error table corrected; one contradiction resolved |
| Coverage | 0.78 | 0.78 | No new runtime evidence |
| Flow | 0.84 | 0.84 | No flow change |
| Abstraction | 0.82 | 0.80 | L4-02 scoped down |
| Counterexample | 0.86 | 0.86 | No change |
| Epistemic | 0.90 | 0.91 | Clearer separation of built-in path |
| **Overall** | **PARTIAL** | **PARTIAL** | Runtime still absent |

## 6. Audit trail

- `06_validation/independent_audit.md` — 30 findings, stats, benchmark case.
- `06_validation/evidence_map.md` — 58 checks, 49 PASS / 9 PARTIAL.
- This file — applied corrections.
- No target repo / KnowlegeMap / production skill file was modified.

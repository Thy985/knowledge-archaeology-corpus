# 06 · Reconciliation

## Rule

The independent audit is not overwritten. Reconciliation only adjusts the **package**: pipeline order, omissions, scope tightening, and confidence labels. The original auditor file remains at `audit/independent_findings.md`.

## Reconciliation table

| Audit item | Original claim | Final package treatment | Change type |
|---|---|---|---|
| A-01…A-13, A-15…A-19, A-21…A-29 | confirmed facts | retained as L1/L2/EK with same evidence | none |
| A-14 | Docker "any MCP client" | preserved as **CONTRADICTED** in C-03 and Gate PARTIAL; README vs `http.ts` both quoted | scope/contradiction preserved |
| A-20 | extension token | EK-30 confidence = medium; C-04 open; text says "relay does not compare token", not "token auth" | downgrade |
| A-30 | no runtime evidence | Gate = PARTIAL; quality metric `runtime_validation=false`; no PASS claim | omission preserved |
| Attack: a11y → pattern | "a11y architecture" | L3-02 scoped to "a11y-first default with vision opt-in"; screenshot default and `--caps=vision` noted | scope tightening |
| Attack: L4 → L5 | "default-open control plane" | kept **L4-01 Hypothesis**, not L5 | downgrade |
| Attack: project → universal | "convenience guard" | L5-01 kept but scoped to browser/agent tooling; cross-project pending | scope tightening |
| Attack: no vision | "no vision" | C-08 corrected to "vision not required by default"; screenshot default + coordinate tools opt-in | overgeneralization corrected |
| Attack: extension token | "relay auth" | C-04; no server-side compare found; extension side unread | missing evidence |
| Attack: `vm` escape | "RCE" | C-02 NEEDS_HUMAN_REVIEW; package says RCE-equivalent by description/code, not demonstrated exploit | epistemic separation |
| Attack: benchmark | none | B-01 added as candidate benchmark, not validated rubric | addition |

## What changed in the package as a result

1. **Confidence labels**: EK-30 lowered to medium; L4 models explicitly marked Hypothesis; L5 scoped.
2. **Flow caveat**: F5 authority flow marks the extension token edge as unverified; no "authentication" label.
3. **Evidence map**: Docker contradiction and 30-finding statistics added; no smoothing.
4. **Candidates**: C-03/C-04/C-08/C-10/C-13 added or re-scoped.
5. **Gate**: PARTIAL, with explicit conditions for PASS.

## What did **not** change

- No claim was upgraded to Fact because it sounded coherent.
- No contradiction was deleted.
- No test result was invented.
- No target repository or Skill file was modified.

## Final verdict statistics

**30 independent findings:** 27 CONFIRMED, 1 PARTIALLY_CONFIRMED, 1 CONTRADICTED, 1 MISSING.

**Claim-level attacks:** 2 DOWNGRADED, 1 OVER_GENERALIZED, 1 NEEDS_HUMAN_REVIEW, 1 PARTIALLY_CONFIRMED, 1 MISSING (extension source).

**Gate:** **PARTIAL**.

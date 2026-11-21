# Self-Validation — Mem0 2.2.1 refresh

- run_id: `ARCH-2026-11-17-001`
- validator role: author-side self-check (not independent)
- scope: package completeness, evidence traceability, layer discipline, counts

## 1. Completeness checklist

| Artifact | Present | Note |
|---|---|---|
| 00 Overview | yes | repository facts, top findings, metrics |
| 01 Project Layer | yes | map, lifecycle, stores, auth, test system |
| 02 Engineering Knowledge | yes | 55 EK, 6 edge types |
| 03 Knowledge Layer | yes | 10 KO, R1–R4 |
| 04 Flow Atlas | yes | 83 edges, 7 classes |
| 05 Candidates | yes | 15 C, 5 XP, 5 HR |
| 06 Validation | yes | self + independent + reconciliation + gaps |
| run_metadata.yaml | yes | see package root |
| refresh-delta.md | yes | old KO 9/9, old EK 36/36 |

## 2. Count checks (script-derived, not hand-written)

| Item | Claimed | Script result | Status |
|---|---|---|---|
| EK count | 55 | 55 (`grep -c '^### EK-'`) | PASS |
| EK link declarations | 157 | 157 | PASS |
| EK average out-degree | 2.85 | 157/55 = 2.85 | PASS |
| KO count | 10 | 10 | PASS |
| Flow edges | 83 | 83 | PASS |
| Candidates | 15 | 15 | PASS |
| XP | 5 | 5 | PASS |
| HR | 5 | 5 | PASS |
| KO aggregation coverage | 100% | 10/10 have a rule | PASS |
| EK→KO reverse traces | 10/10 | each KO lists a cluster | PASS |

## 3. Evidence traceability spot-check

- Every EK has at least one `file:line` path or a named test.
- Every KO has `aggregation_rule`, EK cluster, counterexamples, reverse trace.
- Every Flow edge names a file and line range or a symbol.
- Candidates that are not facts are labeled `Hypothesis` / `Needs Runtime` / `Needs Owner`.
- The package does not claim S5 runtime validation for any behavior that was only mock-tested.

## 4. Layer discipline check

- L1/L2 material is kept in 02 and not forced upward: 55 of 55 EK are L1/L2 (27 L1 + 28 L2); no EK is labeled L3.
- L3 KOs are within-project patterns; L4 candidates are explicitly "cross-project validation pending".
- No L5 claim.
- No KO is based only on "same subsystem"; each has a mechanism/causal/invariant/theme reason.

## 5. Known limitations

1. No live vector DB, LLM, embedder, or Postgres. Behavior evidence is code + unit tests + mock HTTP only.
2. TS tests were not executed (`mem0-ts/node_modules` absent). TS statements rely on code + test source, not a green run.
3. `tests/test_server_auth.py` is not green in this environment; its failures are used as evidence of drift, not as proof that production is broken.
4. The retrieval layer is carried from 2.1.0; only `mem0/memory/main.py` changed (+65/−17), `mem0/utils/scoring.py` unchanged. It is not re-proven end-to-end.
5. The old package's 36 EKs are not all re-read; 23 are marked `NR` in `refresh-delta.md`.
6. Flow edges are static; bypass/override paths are mapped only where the code shows them.

## 6. Self-check verdict

**PASS WITH LIMITATIONS.**

The package is complete and internally traceable. It does not meet S5 runtime validation, and two API-family statements were found to be over-broad during independent review (see `06_validation/independent-validation.md`), so this verdict is not "all green".

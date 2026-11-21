# Self-Validation — PoEM (ARCH-2026-11-19-001)

- validation type: **author-side self-check**, before independent blind reconstruction
- rule: this document does not substitute for the Independent Auditor; it records what the archaeologist checked and where the package is weak.

## 1. Structural checks

| Check | Result | Evidence |
|---|---|---|
| EK count | 49 | `grep -c '^### EK-' 02_engineering_knowledge.md` |
| EK link declarations | 156 | parsed `(↔|→)EK-nn` on `links:` lines |
| Average out-degree | 3.18 | 156 / 49 |
| Dangling link targets | 0 | all targets in EK-01…EK-49 |
| Isolated EK | 0 | every ID has ≥1 inbound or outbound edge |
| KO count | 10 (9 L3 + 1 L2 observation) | `03_knowledge_layer.md` |
| KO aggregation rules | 10/10 declared | R1–R4 present |
| Flow edges | 49 | `04_flow_atlas.md` |
| Flow classes | 7/7 | CF/SF/DF/EF/AF/MF/PF |
| Candidates | 14 | `05_candidates.md` |
| KO→EK references | all resolve | no missing EK IDs |
| File/line references | all in range | checked `file:line` against `wc -l` |

> Note: counts in `00_overview.md` are updated to these final values after reconciliation (49 EK / 156 links / 3.18 / 10 KO [9 L3 + 1 L2 observation] / 49 flow / 14 candidates).

## 2. Truth audit

Sampled claims were re-opened against the cited lines. Representative results:

| Claim | Anchor | Verdict |
|---|---|---|
| `ExecutionLedger` default key is `b"trusted-runtime-key"` | `memory.py:89` | CONFIRMED |
| `record()` is public and unchecked | `memory.py:97-101` | CONFIRMED |
| `contains_event` verifies chain then matches action+subject | `memory.py:103-108` | CONFIRMED |
| `verify_chain` fails closed with `compare_digest` | `memory.py:110-119` | CONFIRMED |
| `clear()` is public on both stores | `memory.py:72-74, 121-122` | CONFIRMED |
| safe step writes ledger without success check | `agent.py:42-44` | CONFIRMED |
| run order filter→prompt→decide→authorize→execute | `agent.py:46-71` | CONFIRMED |
| PoEM only vetoes `unsafe_action` | `defenses.py:83-89` | CONFIRMED |
| NaivePoEM ignores subject and chain | `defenses.py:91-106` | CONFIRMED |
| Sentinel threshold 0.4 / five signals | `defenses.py:25-74` | CONFIRMED |
| Anomaly threshold 0.111, calibrated on genuine notes | `defenses.py:142`; `calibrate_anomaly.py` | CONFIRMED |
| ASR = clean safe AND poisoned unsafe | `experiment.py:70-105` | CONFIRMED |
| `attack.py` is not imported by `experiment.py` | `experiment.py:74`; grep | CONFIRMED |
| paper lists coverage/freshness/baseline/overhead/isolation | `body.tex:620-639` | CONFIRMED |
| `_execute_safety_step` has no real-action check | `agent.py:42-44` | CONFIRMED (limitation) |
| ledger has no task/nonce/time lookup | `memory.py:97-108` | CONFIRMED |

Claims **not** independently verified by execution:

- `results_gpt4o.json` / `results_llama8b.json` / `results_gpt4omini.json` values (S3, author-produced; no live model).
- `langchain_results.json` and `auto_attacker_*.json` (not produced in this run).
- Paper’s reported ASR/microsecond claims as scientific effect sizes (only cited as design claims).

## 3. Coverage audit

Read in full: `memory.py`, `agent.py`, `defenses.py`, `attack.py`, `scenarios.py`, `llm.py`, `experiment.py`, `fp_experiment.py`, `attack_poem.py`, `policy_test.py`, `overhead_test.py`, `langchain_port.py`, `auto_attacker.py`, `calibrate_anomaly.py`, `run_sweep.py`, `README.md`; relevant sections of `poem-paper/body.tex`.

Not read in full: `poem_arxiv.tex`, `poem.bib`, all figure scripts, all top-level analysis Markdown, `injection_lab/` (out of scope).

Coverage blind spots:
- No production integration code: cannot prove how a real deployment constructs `ExecutionLedger` or where it stores `events`.
- No live model: cannot confirm the capability paradox or real-model ASR.
- No LangChain execution: port read only; no OpenAI key.
- No concurrency/crash test: durability claims are code-level, not runtime.
- No downstream call-graph: `record()` reachability by untrusted modules is a code-structure concern, not proven in a deployment.

## 4. Flow audit

- 49 edges, each with a symbol/file anchor; no edge derived solely from paper prose.
- Control flow follows `agent.py:46-71`; ledger flow follows `memory.py:97-119`.
- Policy edges PF-02/PF-04 are marked “code + design intent” where the paper, not code, supplies the rule.
- No bypass path was omitted from the code: `authorize_decision` has one condition; `run_task` has no alternate execution branch.

## 5. Abstraction audit

| KO | Layer | Is the layer justified? | Risk |
|---|---|---|---|
| KO-01 | L3 | yes: cluster spans two stores + gate | low |
| KO-02 | L4 candidate (pre-audit; now L3) | conditionally: general claim about audit-log trust; cross-project pending | medium |
| KO-03 | L3 | yes: gate-set coverage is a within-project structural fact | low |
| KO-04 | L3 | yes: proof tuple is observable in code | low |
| KO-05 | L4 candidate (pre-audit; now L3) | conditionally: general trust-boundary claim; only one project | medium |
| KO-06 | L3 | yes: FP/utility trade measured | low |
| KO-07 | L3 | yes: coercion metric implemented | low |
| KO-08 | L3 | yes: mock limitation explicit | low |
| KO-09 | L3 | **weakest**: only three models, author-recorded, no live rerun | high → keep as Observation/Hypothesis |
| KO-10 | L3 | yes: measured linear cost + code mechanism | low |

No L5 claim is made. After reconciliation both L4 candidates are downgraded to L3, explicitly marked “cross-project validation pending,” and are not written as Principles.

## 6. Counterexample and contradiction register

| id | Finding | Impact on package |
|---|---|---|
| CX-01 | PoEM 0% in mock is by construction (no hint phrase for adaptive notes); `README` says mock is plumbing | Prevented citing mock 0% as efficacy. KO-08 records it. |
| CX-02 | Sentinel 100% adaptive ASR in mock is also by construction (adaptive notes contain no mock hints) | Prevented “Sentinel fails” as a scientific claim; only wiring evidence. |
| CX-03 | `policy_test.py` Part A mock reports NoDefense 0%, contradicting paper’s 100% NoDefense (gpt-4o-mini) | Recorded as backend-dependent null result; KO-08. |
| CX-04 | Paper says microsecond overhead; measured 173.8 ms at N=100,000 | KO-10 qualifies the claim; not called a contradiction in the paper because paper compares to an LLM call and code notes O(N). |
| CX-05 | `attack.py` variants differ from `scenarios.py` variants and are unused by `experiment.py` | Added EK-45; published matrix must be attributed to `scenarios.py`. |
| CX-06 | Default key allows a valid chain to be minted | KO-02 records deployment invariant; not called a paper-novel attack because the paper already assumes key isolation. |
| CX-07 | Same-subject old event authorizes a later skip | KO-04 records it as a freshness limitation, matching paper limitation (2). |
| CX-08 | `X01` vs `x01` false negative | Recorded as availability/FP issue, not bypass. |
| CX-09 | `ReasoningMemory.retrieve` returns all entries on a miss | Recorded as context contamination risk (EK-03), not a PoEM bypass. |
| CX-10 | `injection_lab` results could be confused with PoEM | Scope boundary set in 00/01; excluded. |

## 7. Epistemic audit

| Status | Count / usage |
|---|---|
| Fact | EK-01…EK-49 (implementation statements) |
| Observation | KO-01, KO-03, KO-04, KO-06, KO-07, KO-08, KO-10 |
| Hypothesis | KO-09; all 14 Candidates |
| Pattern | KO-01…KO-10 (within-project) |
| L3 Pattern | 8 KOs |
| L4 candidate (pre-audit, downgraded in reconciliation) | KO-02, KO-05 |
| L5 / Principle / Law | **0** |
| S5 runtime validated | overhead timings, ledger boundary behaviors (local) |
| S4 test validated | deterministic mock scripts (logic only) |
| S3 implemented / recorded | all code facts; author result files |
| S4/S5 for real-model ASR | **none** |

## 8. Self-verdict

**SELF_VALIDATED — ACCEPTED FOR INDEPENDENT REVIEW, WITH LIMITATIONS.**

Strong: code-level separation of stores, gate semantics, ledger field/lookup behavior, coercion metric, cost scaling.
Weak: no live-model replication; no production key/storage integration; capability-paradox KO rests on author-recorded artifacts; `attack.py`/`scenarios.py` duplication is a code-hygiene finding, not resolved by the authors’ README.

Independent Auditor must attempt to falsify at least:
1. KO-01’s “execution fact is independent” claim by finding a path where memory can write the ledger.
2. KO-02’s key-custody claim by checking whether any call site requires a non-default key.
3. KO-03’s coverage claim by enumerating all actions and gate conditions.
4. KO-05’s trust-boundary claim by finding a content-based defense that is invariant to rewording.
5. KO-09’s capability claim by checking whether coercion/task failure explains Llama’s lower ASR.

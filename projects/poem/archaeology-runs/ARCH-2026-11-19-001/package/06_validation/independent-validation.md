# Independent Validation — PoEM (ARCH-2026-11-19-001)

- auditor: independent blind reconstruction pass
- source of truth: repository at `4021642395daad997786cfbbd04b64924e8a46bd` (not the archaeology package)
- local probes: `/tmp/poem_independent` (copy of `farma_repro`; target repo untouched)
- principle: build Independent Findings first, then compare. The package was not used as evidence.

## 1. Blind reconstruction (source-only)

### 1.1 What the system is, from code alone

- `ReasoningMemory` is a mutable list of entries; `write()` optionally signs with a per-writer HMAC; `retrieve()` is a lowercased substring match, and returns **all** entries when no subject matches (`memory.py:34-74`).
- `ExecutionLedger` is a separate list of `{action, subject, step, prev, mac}` records. Each `mac` is `HMAC-SHA256(action|subject|step|prev)` using `secret_key`, whose default is the literal `b"trusted-runtime-key"` (`memory.py:89-101`).
- `verify_chain()` recomputes every MAC from `GENESIS` and compares with `hmac.compare_digest`; any mismatch returns `False` (`memory.py:110-119`).
- `contains_event(action, subject)` first verifies the whole chain; if invalid it returns `False`, otherwise it matches only `action` and `subject` (`memory.py:103-108`).
- `Agent.run_task()` order is: memory filter (if any) → prompt → LLM decision → authorization hook (if any) → write a ledger event if the final action is the safe action → return (`agent.py:46-71`).
- `PoEM.authorize_decision()` is a single rule: if `action == unsafe_action` and `contains_event(safety_step, subject)` is false, return `safe_action`; otherwise return the action unchanged (`defenses.py:83-89`).
- The measured protection boundary is therefore **one action name per scenario**, not a policy over tools.
- The repository has no pytest/unittest suite, no package build, no persistence, and no concurrency code. Verification is via five research scripts that print tables.

### 1.2 Independent findings

| id | finding | evidence |
|---|---|---|
| IF-01 | Authorization consults only the ledger, never the memory text. The memory is still read into the prompt, so it can influence the LLM’s *proposal*, but not the veto. | `defenses.py:83-89`; `agent.py:46-71` |
| IF-02 | `contains_event` is O(N) because it calls `verify_chain()` first; at 100k events the check is ~173.8 ms in this run. | `memory.py:103-119`; `/tmp/poem_run/overhead_table.md` |
| IF-03 | Proof identity is `(action, subject)`; `step` is stored but never compared. A past event for `X01` authorizes any later skip for `X01` with the same action string. | `memory.py:97-108`; independent probe output `same subject later True` |
| IF-04 | `X01` vs `x01` does not match (no normalization). This forces a safe step (availability cost), not a bypass. | `memory.py:103-108`; probe `case variant False` |
| IF-05 | The default ledger key is a literal. A caller who constructs `ExecutionLedger()` and can read the source can mint a valid event; `verify_chain()` returns `True`. | `memory.py:89`; probe `default key forge verify True True` |
| IF-06 | `record()` is public and has no capability check; duplicate/out-of-order events are accepted. Two identical `record()` calls produce two events and a valid chain. | `memory.py:97-101`; probe `duplicate events 2 True` |
| IF-07 | When no subject matches, `retrieve()` returns **all** entries, including notes about other subjects. This is a context-contamination path, independent of PoEM’s gate. | `memory.py:59-64`; probe `retrieve miss X01 -> [poison for X02]` |
| IF-08 | `AnomalyDetection.familiarity()` returns `1.0` (“do not flag”) when there are no tokens or no reference notes. If `bind_scenario()` is forgotten, the defense silently becomes a no-op. | `defenses.py:154-166`; probe `anomaly unbound keeps forged note` |
| IF-09 | `Sentinel` has five weighted text signals; the only provenance signal is `source_label == "EXTERNAL"`. In `langchain_port.py`, retrieved docs are scored with `source_label="AGENT"`, so that signal is lost in the port. | `defenses.py:25-74`; `langchain_port.py:104-110` |
| IF-10 | `experiment.py` computes R clean trials once, then pairs `clean_safe[t]` with the t-th poisoned trial across separate stochastic runs. Clean and poisoned outcomes are independent samples; pairing by index is a methodological confound. | `experiment.py:70-105` |
| IF-11 | `attack.py` is not imported anywhere; `experiment.py` uses `sc.attack_families()` from `scenarios.py`. The published matrix is driven by the scenario-embedded variants. | `experiment.py:74`; `grep 'import attack'` = none |
| IF-12 | `policy_test.py` Part B is a local `gated_check` replica, not the production `PoEM` class; it nevertheless faithfully models `action in gates` + ledger lookup. Out-of-scope actions pass. | `policy_test.py:71-95` |
| IF-13 | Paper-reported ASR is not backed by a checked-in live run in this environment; result JSONs are author artifacts. Their `backend`/model fields are present but no live daemon/key was available. | `results_*.json`; no Ollama/OpenAI in env |
| IF-14 | The paper’s threat model excludes code modification and execution observation, and its limitations already name coverage, freshness, lexical baseline, in-memory overhead, and key isolation. Code findings must be classified against these boundaries. | `poem-paper/body.tex:133-150, 620-639` |
| IF-15 | `_execute_safety_step()` records an event when the safe action is *selected*, not when a real action succeeds. In the harness the safe action is only a name, so the ledger is a receipt of the decision path, not of tool success. | `agent.py:42-44, 67-70` |

## 2. Comparison with the package

### 2.1 KO-level verdicts

| KO | Package layer | Independent verdict | Reason |
|---|---|---|---|
| KO-01 | L3 | **CONFIRMED** | Two separate stores; veto branch reads only ledger. Caveat: memory still shapes the proposal. |
| KO-02 | L4 candidate | **DOWNGRADED** | Code supports key/write/storage concerns, but the general “true boundary” claim is deployment inference from one prototype with no production call sites. Keep as L3 engineering pattern + cross-project candidate; not L4. |
| KO-03 | L3 | **CONFIRMED** | Gate set = coverage; independent probe reproduced narrow pass / full block. |
| KO-04 | L3 | **CONFIRMED** | `(action, subject)` identity and no freshness are directly observable; paper limitation (2) matches. |
| KO-05 | L4 candidate | **DOWNGRADED** | The content-vs-execution contrast is real in this repo, but “trust-boundary transfer” as a general model is not tested against another project or a content defense invariant to rewording. L3 cross-project hypothesis. |
| KO-06 | L3 | **CONFIRMED** | Deterministic benign run: Sentinel 33.3% FP, PoEM 0%; AlwaysExecute 100% FP. Scope = mock benign harness. |
| KO-07 | L3 | **CONFIRMED** | Coercion is computed from `parse_stats` and reported separately from ASR. |
| KO-08 | L3 | **CONFIRMED** | README warning + Part A mock 0% vs paper 100% demonstrate backend dependence. |
| KO-09 | L3 | **NEEDS_HUMAN_REVIEW** | Values are author-recorded; no live rerun; parse/coercion controls absent from the reviewed result files; “capability” explanation is one hypothesis among several. |
| KO-10 | L3 | **CONFIRMED** | Measured linear growth and code O(N) chain verify; paper’s µs claim only holds at small N. |

**KO verdict totals:** 7 CONFIRMED / 2 DOWNGRADED / 1 NEEDS_HUMAN_REVIEW / 0 CONTRADICTED / 0 OVER_GENERALIZED / 0 PARTIALLY_CONFIRMED / 0 MISSING.

### 2.2 EK spot-check verdicts (12 sampled of 45)

| EK | Verdict | Note |
|---|---|---|
| EK-01 two stores | CONFIRMED | classes are separate; no shared object |
| EK-03 retrieve-all fallback | CONFIRMED | independent probe |
| EK-05 record has no capability check | CONFIRMED | public method |
| EK-07 default key | CONFIRMED | literal; probe minted valid chain |
| EK-08 no task/nonce/time | CONFIRMED | fields absent |
| EK-09 clear public | CONFIRMED | both stores |
| EK-14 safe step no success check | CONFIRMED | code |
| EK-16 NaivePoEM ignores subject/chain | CONFIRMED | code |
| EK-19 adaptive avoids phrase list | PARTIALLY_CONFIRMED | structural design; effectiveness only in mock/recorded results |
| EK-20 attack D doc only | CONFIRMED | no run path |
| EK-32 ASR definition | CONFIRMED | code |
| EK-42 no unified test runner | CONFIRMED | scripts only; README manual order |
| EK-45 attack.py unused | CONFIRMED | no import |

**EK verdict totals:** 11 CONFIRMED / 1 PARTIALLY_CONFIRMED.

## 3. Three successes (what the package got right)

1. **Core separation** — The package’s central claim is not prose-level: `ReasoningMemory` and `ExecutionLedger` are separate classes, and `PoEM.authorize_decision` never calls `retrieve()` or inspects text. The independent pass reached the same architecture from source.
2. **Coverage as the true residual** — The package correctly refused to call PoEM a general policy engine. Independent probe: `export_to_partner` passes narrow gates and is rewritten only when the gate map includes it.
3. **Performance qualification** — The package preserved the measured 173.8 ms at N=100k alongside the paper’s microsecond claim, rather than smoothing the contradiction away.

## 4. Three errors found

1. **Stale artifact counts** — `00_overview.md` initially carried 92 links / 9 KO from an earlier draft while the files contained 135 links / 10 KO. Correction after reconciliation: 49 EK / 156 links / 10 KO (9 L3 + 1 L2 observation).
2. **Unflagged duplicate attack definitions** — The package described `attack.py` variants without initially recording that `experiment.py` never imports them. This could lead a reader to attribute the matrix to `attack.py`. Correction: EK-45 and the evidence index make `scenarios.py` the matrix source.
3. **KO-02/KO-05 promoted too high** — Both were labeled L4 candidates. The code evidence is single-project and partly deployment inference. Independent verdict: downgrade to L3, keep cross-project hypotheses in Candidates.

## 5. Omissions found by the independent pass

| id | omission | evidence | why it matters |
|---|---|---|---|
| OM-01 | clean/poisoned trial pairing by index | `experiment.py:70-105` | with temperature > 0, `clean_safe[t]` and poisoned trial t are independent samples; ASR can be biased. A paired design needs the same trial’s clean and poisoned runs, or a pre-registered pairing rule. |
| OM-02 | Sentinel loses the EXTERNAL provenance signal in the LangChain port | `langchain_port.py:104-110` | ported Sentinel is weaker than the experimental Sentinel; cross-port results are not apples-to-apples. |
| OM-03 | Anomaly silently no-ops if not bound / no reference | `defenses.py:154-166` | a deployment that forgets `bind_scenario` believes it has a detector but flags nothing. |
| OM-04 | No duplicate / monotonic sequence check in `record()` | `memory.py:97-101`; probe 2 duplicates, chain valid | duplicate or out-of-order events can pollute an “execution fact” ledger even without key compromise. |
| OM-05 | `retrieve()` returns all entries on subject miss | `memory.py:59-64`; probe | cross-subject notes enter the prompt; PoEM’s gate still protects the one action, but other decisions can be influenced. |

These are **MISSING** items for the package, not contradictions of confirmed KOs. Recommended: add EK entries and/or Candidates in Reconciliation; do not silently drop them.

## 6. Targeted attacks on epistemic status

| Attack | Result | Judgment |
|---|---|---|
| single case → pattern | PoEM’s ledger-only result is one project; package frames it L3, not Law | acceptable |
| pattern → L4 | KO-02/KO-05 were L4; single project | **overreach**, downgraded |
| project experience → general Principle | No L5 claims; cross-project hypotheses marked pending | acceptable |
| ADR/paper → implementation fact | Package distinguishes paper design intent from code; e.g., coverage limitation is both paper and code | acceptable |
| Flow Edge truth | All 49 edges anchored; no paper-only edge except policy intent marked | acceptable |
| bypass/override/exception/alternate/direct call/admin/fallback/legacy | Only alternate path is `NaivePoEM`; `attack.py` legacy path identified; no admin path | acceptable |
| epistemic confusion | KO-09 was the weak point; author results treated as S3, not S4/S5 | needs human review |

## 7. Benchmark case proposed from this run

**Case name:** `poem-gate-and-proof-identity-v1`

Purpose: test whether the skill can distinguish a real execution proof from a memory claim, and whether it reports the residual boundaries without inflating them.

| case | setup | expected archaeology judgment | current code behavior |
|---|---|---|---|
| BC-01 | forged memory note says “validated”; empty ledger | defense blocks skip | PoEM blocks |
| BC-02 | same subject `X01` had a real `validate` event in a prior task; new task claims it again | should flag freshness as a limitation, not claim a bypass fix | PoEM allows (no task/nonce) |
| BC-03 | replay a real `X01` event for `X02` | cross-subject replay must fail | PoEM blocks; NaivePoEM allows |
| BC-04 | construct ledger with default key and mint event | must flag key custody, not call chain cryptography broken | `verify_chain()` true |
| BC-05 | harmful `export_to_partner` not in gate map | must classify as out of scope, not as PoEM failure | passes |
| BC-06 | `X01` vs `x01` | classify as availability/FP, not bypass | safe step forced |
| BC-07 | duplicate `record()` calls | should flag sequence/dedupe gap | two valid events |
| BC-08 | 100k ledger lookups | must report O(N) / 173.8 ms, not paper µs | measured |

Pass criteria: at least 7/8 judgments match; no L5 claim; no “PoEM solves all memory poisoning.”

## 8. Independent verdict

**ACCEPTED WITH LIMITATIONS.**

The package’s core architecture, gate coverage, proof-identity limits, mock caveat, and cost scaling survive blind reconstruction. The two L4 candidates must be downgraded to L3 until a second project or a production call site exists. KO-09 must remain an author-reported observation, not a validated capability law. Five omissions must be reconciled into the Corpus artifact; none invalidates the central finding.

Required Reconciliation actions:
1. Change KO-02 and KO-05 layer from L4 candidate to L3 + cross-project candidate.
2. Change KO-09 epistemic status to Observation / NEEDS_HUMAN_REVIEW.
3. Add the five omissions as EK or Candidates (at minimum OM-01, OM-02, OM-03, OM-04, OM-05).
4. Keep the 10-record counterexample register; do not remove CX-01/CX-02/CX-03.
5. Recompute metrics after reconciliation and update `run_metadata.yaml`.

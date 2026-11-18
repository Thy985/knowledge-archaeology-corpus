# 03 Knowledge Layer — PoEM

- run_id: `ARCH-2026-11-19-001`
- rule: each KO is a cluster on the EK graph; `aggregation_rule` is mandatory (R1–R4).
- epistemic discipline: L3 = within-project pattern; L4 = model candidate, cross-project validation pending; no L5 claimed.
- reconciliation: Independent Validation downgraded KO-02 and KO-05 from L4 candidate to L3; KO-09 remains an L2 Observation (NEEDS_HUMAN_REVIEW), not Generalized Knowledge. Final tally: **9 L3 KOs + 1 L2 observation**, 0 L4, 0 L5.

## KO-01 Execution fact is a separate authority from reasoning content
- layer: **L3**
- aggregation_rule: **R3 不变量簇**
- EK cluster: EK-01, EK-06, EK-09, EK-10, EK-14, EK-21
- edge types: `subsystem` (EK-01↔EK-06), `causal` (EK-06→EK-10→EK-09→EK-21; EK-14→EK-06), `constraint` (EK-09→EK-21)
- epistemic status: **Observation / Pattern**
- statement: The system treats “what was said about execution” and “what executed” as two different authorities. The decision gate consults only the HMAC-chained execution record, and a memory claim cannot substitute for a missing event. The separation is structural (two objects, two writers, two integrity rules), not a prompt instruction.
- why this cluster holds: six EKs converge on one invariant — a safety-step claim is valid only if an independently written event exists. It spans storage (`memory.py`), control flow (`agent.py`), and policy (`defenses.py`), and the chain check is the mechanism that makes the record independent of attacker text.
- L1 → L2 → L3:
  - L1: `ReasoningMemory` and `ExecutionLedger` are separate lists; `PoEM.authorize_decision` calls `contains_event`, not `memory.retrieve`.
  - L2: because the attacker controls the memory channel; any defense that reads it can be satisfied by the attacker. An event written by a different layer has different provenance.
  - L3: “execution facts should be a separate authority from claims” is a reusable pattern for agents with side-effecting tools.
- counterexamples: if the attacker can invoke the genuine action path, the event becomes attacker-influenced (paper limitation 5). If the action layer writes the event before the action succeeds, the fact is a receipt of intent, not execution (see C-05).
- evidence strength: S3 (code) + S2 (paper); no live model rerun for this structural claim.
- reverse trace: KO-01 → EK-01/EK-06/EK-09/EK-10/EK-14/EK-21 → `farma_repro/memory.py:34-122`; `farma_repro/agent.py:42-71`; `farma_repro/defenses.py:76-89`.

## KO-02 Tamper-evidence is key custody and write authority, not the hash chain alone
- layer: **L3** (downgraded from L4 candidate by Independent Validation: single-project deployment inference)
- aggregation_rule: **R3 不变量簇**
- EK cluster: EK-05, EK-07, EK-08, EK-11, EK-12, EK-44, EK-49
- edge types: `constraint` (EK-07→EK-06, EK-08→EK-12), `contrast` (EK-07↔EK-05), `subsystem` (EK-11↔EK-06)
- epistemic status: **Pattern (within-project); cross-project validation pending**
- statement: An HMAC chain proves “these bytes were produced by someone holding the key, in this order.” It does not prove “only the trusted runtime could produce them.” That second property depends on (a) secret key custody, (b) write capability isolation, (c) durable storage the attacker cannot rewrite, and (d) the action layer only appending after real execution. PoEM’s code implements the chain but leaves (a)–(d) to deployment: a public default key, a public `record()`, an in-memory list, and a public `clear()`.
- why this is L3 after reconciliation: the four conditions are a strong engineering pattern, but there is no second project or production call site proving it is a general cognitive model. `record()` also accepts duplicate/out-of-order events (EK-49), so the “write authority” condition is broader than key custody.
- L1 → L2 → L3:
  - L1: `secret_key=b"trusted-runtime-key"`; `record()` public; `events` list; `clear()` public.
  - L2: the paper assumes the attacker cannot modify code or observe execution, and states the key must remain in the trusted runtime. The implementation exposes a default that makes key knowledge trivial if source is readable; it does not require injection.
  - L3: “tamper-evident log” is a claim about a trust boundary, not a property of a hash function; the boundary is bounded by the weakest of key custody, writer capability, storage control, and sequence discipline.
- explanation scope: predicts that any deployment embedding `ExecutionLedger()` without an explicit key, or persisting `events` to an attacker-writable store, loses the guarantee even though `verify_chain()` still returns true.
- counterexamples: a deployment that injects a secret from a KMS, runs the ledger in a separate process, persists append-only storage, and exposes no write API restores the claim. The code would not need to change; the deployment would.
- evidence strength: S3 (code) + S1/S2 (threat model and paper limitation); this run additionally demonstrated a valid chain can be minted with the default key (`/tmp` experiment, see 06).
- reverse trace: KO-02 → EK-05/EK-07/EK-08/EK-11/EK-12/EK-44 → `farma_repro/memory.py:19-25, 89-122`; `poem-paper/body.tex:637-639`.

## KO-03 A defense that gates a decision protects exactly the gate set
- layer: **L3**
- aggregation_rule: **R2 因果链簇**
- EK cluster: EK-18, EK-21, EK-38, EK-44
- edge types: `causal` (EK-21→EK-18; EK-38→EK-21), `constraint` (EK-44→EK-21)
- epistemic status: **Observation / Pattern**
- statement: `PoEM.authorize_decision` only intervenes when the proposed action equals the scenario’s `unsafe_action`. Any other action — including a harmful action not in the gate set, or an unknown action — passes through unchanged. `policy_test.py` Part B demonstrates the exact residual: `export_to_partner` succeeds with narrow gates and is blocked only when added to the gate map. The guarantee is therefore a property of the deployment’s gate coverage, not of PoEM as a library.
- why this cluster holds: four EKs converge on the same boundary: the gate is an action-specific veto, and the policy’s protected set is whatever the caller wires. This is not a bug; it is the documented limitation.
- L1 → L2 → L3:
  - L1: the `if action == unsafe_action` condition; the `gated_check` demo; paper limitation (1).
  - L2: PoEM cannot know the complete set of harmful actions; that is domain policy. A library cannot enumerate a caller’s action space.
  - L3: “decision-level defenses have a coverage boundary equal to the set of wired decisions” is a reusable review checklist for agent guardrails.
- counterexamples: a deployment with a default-deny action whitelist would block unknown actions and shrink the residual; PoEM itself does not provide one. A wrapper that gates every tool call would cover more but still misses actions outside the wrapper.
- evidence strength: S3 (code) + S4 (deterministic Part B).
- reverse trace: KO-03 → EK-18/EK-21/EK-38/EK-44 → `farma_repro/defenses.py:76-89`; `farma_repro/policy_test.py:71-95`; `poem-paper/body.tex:622-624`.

## KO-04 The proof identity is subject + action, not task + step + freshness
- layer: **L3**
- aggregation_rule: **R3 不变量簇**
- EK cluster: EK-09, EK-15, EK-21, EK-44
- edge types: `constraint` (EK-15→EK-09; EK-09→EK-21), `contrast` (EK-03↔EK-16)
- epistemic status: **Observation / Pattern**
- statement: `record()` stores `step`, but `contains_event()` matches only `action` and `subject`. There is no task id, run id, nonce, timestamp, or expiry. Consequently (a) a genuine `validate` for `X01` authorizes every later skip for `X01`, including a different task with changed data; (b) `step` is decorative for lookup; (c) subject casing is exact (`X01` ≠ `x01`), which fail-safes rather than bypasses. The paper calls freshness a limitation; the code confirms it structurally.
- why this cluster holds: four EKs converge on “what makes a proof specific enough to authorize this skip.” The current tuple answers “which action for which subject,” not “for which task, at what time, with what data.”
- L1 → L2 → L3:
  - L1: `record(action, subject, step)`; `contains_event(action, subject)`; no other fields.
  - L2: the benchmark’s subjects are stable identifiers and the scenario is one-shot; freshness was out of scope for the prototype.
  - L3: “an authorization token must bind the action, the object, and the validity window” is a reusable security pattern.
- counterexamples: if the underlying subject is immutable (e.g., a fixed test fixture), indefinite validity is acceptable. If every task uses a fresh subject id, replay across tasks is reduced but not eliminated for same-subject retries.
- evidence strength: S3 (code) + S4 (deterministic `contains_event` behavior) + S2 (paper limitation).
- reverse trace: KO-04 → EK-09/EK-15/EK-21/EK-44 → `farma_repro/memory.py:97-108`; `farma_repro/agent.py:42-44`; `poem-paper/body.tex:626-628`.

## KO-05 Content classification and execution accounting shift the trust boundary
- layer: **L3** (downgraded from L4 candidate by Independent Validation: single-project conceptual comparison)
- aggregation_rule: **R1 机制簇**
- EK cluster: EK-01, EK-04, EK-21, EK-23, EK-25, EK-26, EK-27, EK-29, EK-30, EK-47
- edge types: `contrast` (EK-21↔EK-25; EK-25↔EK-27; EK-23↔EK-04), `mechanism` (EK-25↔EK-26), `causal` (EK-29/EK-30→EK-25), `subsystem` (EK-01↔EK-21), `dependency` (EK-47→EK-19)
- epistemic status: **Pattern (within-project); cross-project validation pending**
- statement: Sentinel, AnomalyDetection and SignedMemory all decide on properties of the attacker-controlled note (wording, lexical familiarity, writer signature). Rewording changes the first two; a compromised authorized writer defeats the third. PoEM decides on a property of a different object (an execution event) that the attacker does not control under the threat model. The trade is not “better classifier vs worse classifier”; it is “inspect the claim vs inspect the world.”
- why this is L3 after reconciliation: the contrast is directly implemented and measured in this repo, but it is one project plus a conceptual comparison. A second project or a content defense invariant to rewording is required before a cognitive-model claim.
- L1 → L2 → L3:
  - L1: Sentinel score signals; Anomaly Jaccard; SignedMemory writer HMAC; PoEM ledger lookup; LangChain port loses EXTERNAL provenance (EK-47).
  - L2: attacker controls the content channel, so content-based scores are an arms race; the execution channel is outside that control only if the trust boundary holds (KO-02).
  - L3: “don’t classify the claim; verify the fact” is a recognizable security pattern; robustness is bounded by whether the decision input is attacker-controlled.
- explanation scope: predicts that rewording attacks will keep working against phrase/familiarity/signature filters, and that execution-accounting defenses will be insensitive to wording but sensitive to key custody and gate coverage.
- counterexamples: a content classifier trained on the true distribution could in principle outperform a weak execution check; a compromised action layer defeats PoEM. The claim is about the input’s trust status, not about classifier quality in general.
- evidence strength: S3 for code; S3 for recorded results; no live-model comparison in this run.
- reverse trace: KO-05 → EK-01/EK-04/EK-21/EK-23/EK-25/EK-26/EK-27/EK-29/EK-30/EK-47 → `farma_repro/defenses.py:25-204`; `farma_repro/attack.py:28-98`; `farma_repro/langchain_port.py:100-112`.

## KO-06 False positives are a security-relevant utility cost
- layer: **L3**
- aggregation_rule: **R1 机制簇**
- EK cluster: EK-24, EK-25, EK-26, EK-27, EK-28, EK-36
- edge types: `mechanism` (EK-25↔EK-26; EK-27↔EK-28), `contrast` (EK-24↔EK-25), `causal` (EK-26/EK-27→EK-36)
- epistemic status: **Observation / Pattern**
- statement: The benchmark treats legitimate skip rate as utility and FP rate as `NoDefense - defense`. Sentinel’s audit-log-style genuine note is dropped (33.3% FP in mock; 33–50% in recorded results), because it uses the same vocabulary as a forgery. AlwaysExecute reaches 0% ASR by destroying 100% of legitimate skips. PoEM preserves 100% of legitimate skips in mock because the genuine ledger event exists. Security evaluation that reports only ASR hides this cost.
- why this cluster holds: six EKs connect the detector’s decision rule to the benign trace and the measured utility loss. It spans three defenses and one experimental design.
- L1 → L2 → L3:
  - L1: Sentinel drop rule; genuine audit-log note; `fp_experiment` skip rate; AlwaysExecute.
  - L2: a detector tuned to forgeries will also fire on genuine logs written in the same register; a defense that always blocks has zero utility.
  - L3: “a security control’s false-positive cost is part of its security case, not a separate quality metric” is reusable.
- counterexamples: if re-validation is cheap relative to the risk, a high-FP defense may be acceptable; the benchmark’s scenarios assume re-validation has a cost. If the genuine log can be rewritten to avoid the detector’s signals, FP can be reduced without changing the detector.
- evidence strength: S4 for deterministic FP behavior (mock) + S3 for recorded real-model results.
- reverse trace: KO-06 → EK-24/EK-25/EK-26/EK-27/EK-28/EK-36 → `farma_repro/fp_experiment.py:45-105`; `farma_repro/defenses.py:25-176`.

## KO-07 Parser failure must be separated from defense success
- layer: **L3**
- aggregation_rule: **R2 因果链簇**
- EK cluster: EK-19, EK-32, EK-33, EK-35
- edge types: `causal` (EK-19→EK-35; EK-33→EK-32), `constraint` (EK-35→EK-32)
- epistemic status: **Observation / Pattern**
- statement: A model reply that cannot be parsed is coerced to the safe action. If the evaluator counts every safe action as “attack blocked,” a weak model’s formatting failure inflates the defense’s apparent robustness. `experiment.py` records `json_error` / `missing_action` / `bad_action` and prints a coercion rate next to ASR; this is the mechanism that keeps “resistance” distinct from “failure to answer.”
- why this cluster holds: four EKs connect parsing, coercion, ASR definition, and reporting.
- L1 → L2 → L3:
  - L1: `_coerce_action`; `parse_stats`; ASR definition; coercion print.
  - L2: an unparseable answer is not evidence that the defense changed the model’s intent; it is evidence that the harness could not read it.
  - L3: “report parser/abstention rate separately from attack success” is a reusable evaluation rule for LLM security.
- counterexamples: if coercion is treated as a secure default by the deployed system, one could argue it is a real defense outcome; but then the claim is about the wrapper, not the model’s resistance. The report should state which.
- evidence strength: S3 (code) + S4 (logic).
- reverse trace: KO-07 → EK-19/EK-32/EK-33/EK-35 → `farma_repro/llm.py:87-112`; `farma_repro/experiment.py` coercion section.

## KO-08 Mock harness validates plumbing, not efficacy
- layer: **L3**
- aggregation_rule: **R4 主题簇**
- EK cluster: EK-20, EK-30, EK-38, EK-41, EK-42
- edge types: `contrast` (EK-20↔EK-30), `constraint` (EK-20→EK-38), `mechanism` (EK-41↔EK-42)
- epistemic status: **Observation / Pattern**
- statement: The mock backend decides by phrase hints. It can show that the control path, ledger writes, and attack injection are wired correctly, but it cannot show that a real model would be persuaded by a reworded note. The repository says this explicitly. Two consequences appear in this run: (a) the main mock matrix reproduces PoEM 0% and NoDefense 100% by construction; (b) `policy_test.py` Part A reports 0% for NoDefense because the judgment notes contain no mock hints, contradicting the paper’s 100% NoDefense claim, which was measured with gpt-4o-mini. The evaluator must name the backend.
- why this cluster holds: five EKs connect the mock’s decision rule, adaptive variants, the default policy test, the auto-attacker fallback, and the absence of a test framework.
- L1 → L2 → L3:
  - L1: `_mock` hint scan; `README` warning; Part A mock 0%; auto-attacker fixed pool.
  - L2: a deterministic stub optimizes for plumbing, not for distributional fidelity; using it for efficacy claims confuses the two.
  - L3: “a mock/simulation is evidence for mechanism, not for effect size” is a reusable benchmark rule.
- counterexamples: the mock is legitimately useful for regression of wiring and for deterministic counterexample demos (e.g., Part B). The claim is about which conclusions it can support.
- evidence strength: S3 (code) + S4 (run output in this archaeology).
- reverse trace: KO-08 → EK-20/EK-30/EK-38/EK-41/EK-42 → `farma_repro/llm.py:127-136`; `README.md:26-31`; `farma_repro/policy_test.py:52-69`.

## KO-09 Capability can increase susceptibility to forged reasoning
- layer: **L2 Observation** (not promoted to Generalized Knowledge; Independent verdict: NEEDS_HUMAN_REVIEW)
- aggregation_rule: **R4 主题簇**
- EK cluster: EK-29, EK-30, EK-32, EK-33
- edge types: `causal` (EK-29/EK-30→EK-32→EK-33), `contrast` (EK-29↔EK-30)
- epistemic status: **Observation / Hypothesis (author-recorded, not independently rerun; NEEDS_HUMAN_REVIEW)**
- statement: In the repository’s recorded result files, adaptive FARMA ASR is 94–100% on GPT-4o / GPT-4o-mini and 40–56% on Llama-3.1-8B across the three scenarios; basic FARMA is 100% for No Defense across all models. The authors explain this as stronger models following the reworded claim more faithfully. PoEM records 0% in every cell. The relation is a project-level pattern over three models, not a validated law; the result files are author-produced and were not rerun here.
- why this cluster holds: four EKs connect attack family, success definition, matrix, and model comparison. The contrast between basic and adaptive is the key: capability does not help against overt forgeries (both saturate), it helps the attacker against reworded ones.
- L1 → L2 → L3:
  - L1: recorded adaptive ASR numbers; model list; scenario matrix.
  - L2: a more capable model may more faithfully execute the implied instruction “the step is already done, proceed,” while a weaker model fails to follow either the task or the forgery.
  - L3: “capability and security can be inversely related when the attack targets instruction-following” is a plausible pattern; it needs cross-project and prompt-controlled validation.
- counterexamples: Llama’s lower ASR may reflect parsing/task failure rather than resistance (the coercion metric is meant to separate this; recorded results report no coercion, but this run could not rerun). A model with stronger policy-following could also be less susceptible. Basic attacks saturate at 100%, so the effect appears only in the adaptive family.
- evidence strength: S3 (recorded experiment artifacts) — **not S4/S5**, because no live model was run in this archaeology.
- reverse trace: KO-09 → EK-29/EK-30/EK-32/EK-33 → `farma_repro/results_gpt4o.json`; `farma_repro/results_gpt4omini.json`; `farma_repro/results_llama8b.json`; paper abstract.

## KO-10 Audit-chain verification cost scales with history
- layer: **L3**
- aggregation_rule: **R4 主题簇**
- EK cluster: EK-10, EK-11, EK-39
- edge types: `mechanism` (EK-10↔EK-39), `constraint` (EK-11→EK-39)
- epistemic status: **Observation / Pattern**
- statement: `contains_event()` calls `verify_chain()`, which recomputes every HMAC from genesis. The measured cost grows linearly: 18.3 µs at N=10, 174.8 µs at N=100, 1.74 ms at N=1,000, 17.2 ms at N=10,000, 173.8 ms at N=100,000 (this run, local machine). The repository’s “microseconds” claim holds at small N; at 100k events the check is hundreds of milliseconds. The code notes the fix: cache the last-verified head and verify only new appends (O(1) amortized).
- why this cluster holds: three EKs connect chain verification, in-memory storage, and measured cost. It is a performance property with a clear architectural remedy.
- L1 → L2 → L3:
  - L1: full-chain loop in `verify_chain`; benchmark table.
  - L2: tamper-evidence requires checking the whole chain unless a trusted head is cached; the prototype prioritizes simplicity.
  - L3: “append-only audit verification is O(history) unless the verifier keeps a trusted checkpoint” is a reusable design rule.
- counterexamples: a ledger with a trusted checkpoint or Merkle root can verify in O(log N) or O(1) amortized; a ledger small enough (<1k events) makes the cost negligible next to an LLM call.
- evidence strength: S5 for the local timing (actually executed in this run), S3 for code.
- reverse trace: KO-10 → EK-10/EK-11/EK-39 → `farma_repro/memory.py:103-119`; `farma_repro/overhead_test.py:30-48`.

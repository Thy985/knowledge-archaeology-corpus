# 05 Candidates — PoEM

- run_id: `ARCH-2026-11-19-001`
- rule: these are **not** validated knowledge. Each carries an epistemic status, current evidence, missing evidence, and a validation path.
- Cross-project candidates must not be written as Principles.

## C-01 Cached verified head makes grounding-check cost amortized O(1)
- type: Design / Performance
- epistemic: **Hypothesis**
- statement: `contains_event()` currently re-verifies the entire chain on every lookup (O(N)). Caching the last verified head MAC and event count, then verifying only appended events, would make repeated lookups amortized O(1) while preserving tamper detection for the verified prefix.
- current evidence: `memory.py:103-119`; `overhead_test.py` source comment names this optimization; measured 173.8 ms at N=100,000.
- missing evidence: no implementation, no benchmark of cache invalidation, no analysis of what happens if the cached head is rolled back or the process restarts.
- validation path: implement a `TrustedCheckpoint` with head MAC + index in a separate branch; benchmark lookup at N=100k; test rollback/truncation and cross-process checkpoint tampering.
- layer potential: L3 engineering pattern (audit-chain checkpointing); not a KO yet.

## C-02 Bind proof to task/nonce and a validity window
- type: Design / Security
- epistemic: **Hypothesis**
- statement: Extending the event or lookup with `task_id`, `attempt_id`, `nonce`, and `issued_at`/`expires_at` would prevent a past `validate` from authorizing an unrelated later skip for the same subject, and would make the `step` field meaningful.
- current evidence: `record(action, subject, step)` stores only those fields; `contains_event(action, subject)` ignores `step` (`memory.py:97-108`); a same-subject old event returned `True` in this run.
- missing evidence: no freshness requirement in the benchmark; no task identity in the harness; no spec for expiry semantics (hard fail vs re-validate).
- validation path: add a task-scoped subject and a TTL test with a fake clock; run `attack_poem.py` B (cross-subject) and a new same-subject/different-task case; measure legitimate re-validation cost.
- layer potential: L3 pattern (authorization token binding); feeds KO-04 revision.

## C-03 The constructor should require an injected key, not default to a literal
- type: Design / Security
- epistemic: **Hypothesis**
- statement: Changing `ExecutionLedger.__init__` to require `secret_key` (no default) and having `Agent` accept a key provider would make the key-custody assumption explicit at every call site. Today the default literal means a deployment can accidentally run with a public key.
- current evidence: `memory.py:89-90`; `agent.py:37-39` constructs without a key; this run minted a valid chain using the default key and `verify_chain()` returned true.
- missing evidence: no production integration code in the repo; cannot show a real deployment uses the default.
- validation path: grep for `ExecutionLedger(` call sites in downstream code; add a failing test that constructs without a key; run a deployment-level key-custody review.
- layer potential: L2/L3; supports KO-02.

## C-04 Ledger writes should be a capability held only by the action layer
- type: Design / Security
- epistemic: **Hypothesis**
- statement: Passing the ledger object to arbitrary modules makes `record()` callable by anyone. A capability object (e.g., a write-only handle constructed inside the executor) or a separate ledger process would enforce “only the action layer writes.”
- current evidence: `record()` is public (`memory.py:97`); the docstring says it is the only legitimate writer but there is no check.
- missing evidence: no call-graph proof that untrusted code cannot reach the ledger; no process boundary in the prototype.
- validation path: instrument all `record()` callers; attempt an unauthorized append from a “plugin” module; if it succeeds, implement a capability handle and re-test.
- layer potential: L3 pattern (capability-separated audit writes); supports KO-02.

## C-05 A receipt must be written after the action succeeds, not when it is selected
- type: Design / Semantics
- epistemic: **Hypothesis**
- statement: `_execute_safety_step` writes the ledger event when the agent *selects* the safe action. If the real action can fail (tool error, timeout, partial write), the ledger contains a false execution fact. A two-phase record (`intent` then `commit`) or an executor callback would make the receipt correspond to success.
- current evidence: `agent.py:42-44, 67-70`; no success check; the harness’s safe action is just an action name.
- missing evidence: the benchmark has no failing tool; no production executor; no test of action failure.
- validation path: add a safe action that raises; assert no `commit` event is written; add an `intent` event and verify PoEM does not accept it as proof.
- layer potential: L3 pattern (execution receipt semantics); directly relevant to KO-01’s “execution fact” claim.

## C-06 Gate coverage should be a registry, not a single `unsafe_action`
- type: Design / Policy
- epistemic: **Hypothesis**
- statement: A default-deny registry of protected actions (or a wrapper around every tool call) would close the residual shown by `policy_test.py` Part B. Today only the scenario’s `unsafe_action` is gated; `export_to_partner` succeeds when not in the gate map.
- current evidence: `defenses.py:83-89`; `policy_test.py:71-95`; paper limitation (1).
- missing evidence: no inventory of harmful actions in the benchmark; no default-deny semantics; no cost analysis for false positives on new actions.
- validation path: enumerate all actions in three scenarios; run each through PoEM with narrow vs full registry; add an unknown-action test.
- layer potential: L3 pattern (coverage boundary); supports KO-03.

## C-07 Subject canonicalization would reduce false negatives, not bypasses
- type: Design / Usability
- epistemic: **Observation**
- statement: `contains_event` uses exact string equality for `subject`. `X01` and `x01` do not match, so a genuine event under one spelling does not authorize a skip under the other; the result is a forced safe step (availability/FP cost), not a security bypass.
- current evidence: this run’s boundary experiment; `memory.py:103-108` (no normalization).
- missing evidence: no evidence that real callers vary case/whitespace.
- validation path: add canonicalization at the ledger boundary and re-run benign skip rate; check collision risk across distinct subjects.
- layer potential: L2; not a KO.

## C-08 The ledger needs durability and a defined recovery model
- type: Design / Runtime
- epistemic: **Hypothesis**
- statement: An in-memory ledger loses all proofs on restart and cannot serve concurrent agents. A durable append-only store (with the same chain) plus a recovery check would make PoEM usable across sessions; it also introduces a new attack surface (storage write access).
- current evidence: `events` is a list; no persistence code (`memory.py:89-92`).
- missing evidence: no deployment requirement; no crash/restart test; no concurrency test.
- validation path: persist to an append-only file/DB; kill the process mid-write; verify `verify_chain()` after restart; test two writers.
- layer potential: L3 pattern (durable audit log); supports KO-02/KO-10.

## C-09 Cross-project: execution-fact defenses complement provenance defenses
- type: Cross-project
- epistemic: **Hypothesis (Cross-project validation pending)**
- statement: Mem0’s scope/write-path controls and AMG’s poisoning detection govern *who may write* and *whether content looks malicious*; PoEM governs *whether an action actually happened*. These are three different trust questions. A mature agent memory stack likely needs all three, not one.
- current evidence: PoEM EK-01/EK-21; Corpus `mem0` scope invariants (PR #76) and `agent-memory-guard` validation; not yet combined in one system.
- missing evidence: no project implements all three; no interaction analysis (e.g., a scoped, signed memory that still lies about execution).
- validation path: build a comparison matrix across `mem0`, `agent-memory-guard`, `poem` in the Corpus; test one scenario where each layer fails alone and succeeds together.
- layer potential: L4 candidate if validated across ≥2 projects.

## C-10 Cross-project: “verifier knows what executed” is a general agent-security primitive
- type: Cross-project
- epistemic: **Hypothesis (Cross-project validation pending)**
- statement: Any agent that can skip a safety step based on its own memory needs an external execution record; the same pattern may apply to approval workflows, deployment gates, and tool-use traces. PoEM is one implementation, not the primitive.
- current evidence: PoEM KO-01; paper framing; no second implementation in Corpus.
- missing evidence: no cross-project code; no evidence that other projects’ “verification” is memory-based.
- validation path: search Corpus for agents with `skip`/`override`/`already done` logic; classify whether their proof source is memory or execution.
- layer potential: L4 candidate; do not write as Principle.

## C-11 Cross-project: mock/simulation evidence is often mislabeled as efficacy
- type: Cross-project / Evaluation
- epistemic: **Hypothesis (Cross-project validation pending)**
- statement: Research repos frequently ship deterministic mocks for plumbing, then report mock-derived tables alongside real-model claims. The failure mode is not unique to PoEM; it is a benchmark-governance issue.
- current evidence: PoEM `README` warning; `policy_test.py` Part A mock 0% vs paper 100% NoDefense; auto-attacker mock pool.
- missing evidence: no survey of other repos; no standard “backend” field in result schemas.
- validation path: scan Corpus projects for result files lacking a backend/model field; classify claims by evidence strength S3 vs S4/S5.
- layer potential: L3/L4 evaluation pattern.

## C-12 Capability paradox needs a prompt-controlled replication
- type: Evaluation
- epistemic: **Hypothesis**
- statement: The recorded 94–100% (GPT-4o) vs 40–56% (Llama-3.1-8B) adaptive ASR may reflect stronger instruction-following, or may reflect differences in JSON compliance/task competence. A replication that holds prompt format, decoding temperature, and parse success constant would separate the two.
- current evidence: `results_gpt4o.json`, `results_gpt4omini.json`, `results_llama8b.json`; no coercion counts in the recorded artifacts reviewed; no live rerun here.
- missing evidence: live model access; per-cell parse stats; same prompt template across models; multiple seeds.
- validation path: run 20 trials × 3 scenarios on at least two models with `parse_stats` reported; stratify ASR by `ok` parse only; test a policy-trained model.
- layer potential: L3 if replicated; KO-09 currently single-project.

## C-13 A LangChain callback/middleware could gate tool calls automatically
- type: Integration
- epistemic: **Hypothesis**
- statement: `langchain_port.py` wires PoEM manually in `LangChainAgent.run()`. A `BaseCallbackHandler` or tool wrapper could call `contains_event()` before every gated tool invocation, making coverage systematic instead of per-agent.
- current evidence: `langchain_port.py:85-128`; no callback registration; no tool wrapper.
- missing evidence: no run of the port (no OpenAI key); no analysis of callback coverage for async/streaming tools.
- validation path: implement a callback handler, count gated vs ungated tool calls, and run a small LangChain agent with two tools.
- layer potential: L2/L3 integration pattern.

## C-14 A real optimizing attacker may find bypasses the fixed variants miss
- type: Evaluation / Attack
- epistemic: **Hypothesis**
- statement: `auto_attacker.py` is designed to iteratively paraphrase around Sentinel’s phrase list. With the mock backend it returns a fixed pool; with a real LLM it could discover phrasings absent from both `attack.py` and `scenarios.py`. PoEM should remain invariant to wording, but the gate could still be attacked through scope (out-of-scope action) or ledger/key assumptions.
- current evidence: `auto_attacker.py:34-130`; `llm.py` mock `complete_json()`; attack D not executed.
- missing evidence: no real attacker run; no success criterion for “new” bypass vs known variant; no human review of generated notes.
- validation path: run with a real backend, human-audit generated notes, classify by attack type (wording/scope/ledger), and test PoEM against each class.
- layer potential: L2 evaluation method; not a KO.

## Candidate summary

| id | type | layer potential | cross-project | status |
|---|---|---|---|---|
| C-01 | performance | L3 | no | Hypothesis |
| C-02 | security design | L3 | no | Hypothesis |
| C-03 | key custody | L2/L3 | no | Hypothesis |
| C-04 | capability | L3 | no | Hypothesis |
| C-05 | execution semantics | L3 | no | Hypothesis |
| C-06 | gate coverage | L3 | no | Hypothesis |
| C-07 | usability | L2 | no | Observation |
| C-08 | durability | L3 | no | Hypothesis |
| C-09 | cross-project | L4 candidate | yes | Hypothesis |
| C-10 | cross-project | L4 candidate | yes | Hypothesis |
| C-11 | cross-project | L3/L4 | yes | Hypothesis |
| C-12 | evaluation | L3 | no | Hypothesis |
| C-13 | integration | L2/L3 | no | Hypothesis |
| C-14 | attack evaluation | L2 | no | Hypothesis |

# 03 — Knowledge Layer (L1 → L5)

**Rules applied** (`knowledge-archaeology/contracts/knowledge-schema.md`, v3.2):

- Every KO is derived from an EK cluster, not a free list.
- `derivation.facts` equals the cluster exactly: no extra EK, no missing EK.
- L1/L2 may remain engineering facts/patterns. L3+ must explain more than any member EK.
- `flow_traceability` is non-empty.
- A cross-project claim without cross-project evidence is marked **hypothesis**, not validated principle.

Layer counts: **L1=4, L2=4, L3=3, L4=1, L5=1**.

---

## L1 — Ground facts

### KO-01 — Platform capability is contracted in core and realized by exactly one compiled adapter

- **layer**: L1 · **epistemic**: fact
- **claim**: The core exposes a four-trait capability contract; each platform crate implements it; the binary selects one adapter at compile time; the FFI wraps the same contract; unsupported targets fail the build.
- **aggregation_rule**: `R3 Invariant Cluster` — EK-01 (contract), EK-02 (selection), EK-34 (capability matrix), EK-29 (platform-specific release path), EK-07 (FFI as a second consumer), EK-39 (C ABI wrapper) all enforce one invariant: *a capability exists only if the compiled platform implements the contract, and the contract is the only cross-platform vocabulary.* The cluster crosses data/contract, build, runtime matrix, failure path, and two process boundaries.
- **derivation.facts**: `EK-01, EK-02, EK-07, EK-29, EK-34, EK-39`
- **why it is more than a list**: It names the invariant that makes "cross-platform" a contract rather than three separate implementations. The FFI and the release guard are not exceptions; they are consequences of the same boundary. EK-07 adds the in-process consumer; EK-39 adds the C ABI consumer; both must obey the same capability vocabulary.
- **counterexample**: A platform method that returns a default success instead of `not_supported` would break the invariant. No such default success was found in the adapter traits.
- **flow_traceability**: CF-03, AF-05, PF-02.
- **boundary**: Does not prove platform implementations are correct, only that capability claims are explicit.

### KO-02 — One command authority is projected into CLI, batch, MCP, and embedded skills

- **layer**: L1 · **epistemic**: fact
- **claim**: CLI, batch, and MCP all resolve to the same `Commands` enum / clap tree; skills are compiled from the same repository. There is no independent batch grammar or hand-written MCP tool registry.
- **aggregation_rule**: `R1 Mechanism Cluster` — EK-03 (direct dispatch), EK-04 (batch same enum), EK-06 (MCP clap projection), EK-31 (embedded skills) are four instances of one mechanism: *derive every surface from one compiled authority.* They occur in four independent files/subsystems, which is the R1 cross-instance requirement.
- **derivation.facts**: `EK-03, EK-04, EK-06, EK-31`
- **why it is more than a list**: It explains why command drift is structurally hard: a new command cannot appear in only one surface without failing compilation or a drift test.
- **counterexample**: `sandbox exec` is special-cased in MCP as passthrough and is not a normal JSON-envelope command; it is a deliberate exception to the projection, not a second registry.
- **flow_traceability**: CF-01, CF-02, CF-04, CF-05, PF-06, MF-06.
- **boundary**: Derived surfaces reduce drift but concentrate change cost. It does not make the surfaces semantically identical.

### KO-03 — Authority is preflighted per command and per batch entry, with explicit exceptions

- **layer**: L1 · **epistemic**: fact
- **claim**: Permission need, multi-agent identity, and screenshot scope are checked before dispatch; batch validates every entry before the first action. Clipboard is explicitly outside the accessibility permission map.
- **aggregation_rule**: `R4 Thematic Cluster` — EK-08 (permission map), EK-09 (agent-id gate), EK-36 (screenshot scope), EK-05 (batch all-entry preflight), EK-30 (clipboard exception) cover complementary dimensions of one theme: *authority is a static preflight over the command surface, and exceptions must be named.*
- **derivation.facts**: `EK-05, EK-08, EK-09, EK-30, EK-36`
- **why it is more than a list**: It separates "authority coverage" from "authority enforcement": the map declares, `preflight` enforces, batch pre-enforces the whole plan, and clipboard is a visible hole rather than an implicit one.
- **counterexample**: Batch-level `policy_for == None` does not skip entry-level preflight; the cluster would be false if batch bypassed per-entry checks. `prepare` does run them.
- **flow_traceability**: AF-01, AF-02, AF-04, PF-04, PF-07, CF-04.
- **boundary**: Static preflight cannot predict runtime staleness or mid-batch UI change.

### KO-04 — Action outcome is a constrained delivery lattice, and the envelope carries the settled state

- **layer**: L1 · **epistemic**: fact
- **claim**: Delivery has five states; retry is derived; unknown mutations stay unknown; pre-dispatch/unsent errors settle to `not_delivered`; the JSON envelope serializes the disposition; batch wait-event baselines preserve causal order.
- **aggregation_rule**: `R2 Causal Chain Cluster` — EK-14 (lattice) → EK-15 (settlement) → EK-17 (post-action wait wrap) → EK-38 (envelope), with EK-35 providing the pre-action baseline ordering that makes the chain causally valid inside batch. The chain covers decision → execution → disposition → serialization.
- **derivation.facts**: `EK-14, EK-15, EK-17, EK-35, EK-38`
- **why it is more than a list**: It explains why `ok:true` is not the same as "safe": the envelope's `disposition` carries the safety judgment, and the wait wrapper can change the top-level `data` shape without changing that judgment.
- **counterexample**: A mutating command that returns `ok:false` with `delivery_uncertain` is not retry-safe; a read-only command with the same error may be `not_delivered`. The distinction is command class, not error text.
- **flow_traceability**: CF-04, CF-06, EF-02, DF-04, MF-03.
- **boundary**: The lattice is a model of what the code can prove; it does not prove the OS or app actually applied the effect.

---

## L2 — Engineering patterns

### KO-05 — Actionability is an ordered evidence battery with a semantic-first policy

- **layer**: L2 · **epistemic**: pattern
- **claim**: Before an action, the runtime selects a policy (headless/focus-fallback/headed), evaluates a per-action gate battery, lets the platform downgrade a headed request to semantic evaluation, chooses semantic vs physical pointer delivery, and adds a five-point hit-test only for physical delivery.
- **aggregation_rule**: `R2 Causal Chain Cluster` — EK-10 (policy) → EK-37 (per-action capability/delivery) → EK-12 (battery) → EK-13 (hit test), with EK-11 as the platform-specific rewrite of the request. The chain covers policy decision → requirement selection → gate evaluation → physical proof.
- **derivation.facts**: `EK-10, EK-11, EK-12, EK-13, EK-37`
- **why it is more than a list**: It explains the *ordering*: policy is chosen before requirements; requirements decide whether hit testing exists; hit testing is the last gate. Reordering would change which failures are reported.
- **counterexample**: `receives_events` unknown is non-blocking, but `visible`/`enabled` unknown is blocking. If the report treated all unknown equally, the model would be false.
- **flow_traceability**: CF-06, PF-01, PF-02, EF-04.
- **boundary**: The battery proves permission to attempt, not effect. It also depends on advertised capabilities, which may be stale.

### KO-06 — Snapshot completeness and tokenless identity are proven, not assumed

- **layer**: L2 · **epistemic**: pattern
- **claim**: A snapshot distinguishes presentation completeness (`complete`), truncation (`truncated`/`nodes_observed`), and tokenless identity proof (`identity_unproven`). Tokenless refs are published only after a whole-window uniqueness census; shallow captures may spend most of their deadline on that proof.
- **aggregation_rule**: `R2 Causal Chain Cluster` — EK-18 (tri-state result) → EK-19 (whole-window attempt) → EK-21 (capture proof). The chain covers presentation → proof attempt → publication decision.
- **derivation.facts**: `EK-18, EK-19, EK-21`
- **why it is more than a list**: It explains why a "successful" shallow snapshot can still contain unusable refs, and why `complete:true` does not imply identity proof.
- **counterexample**: A platform that retains element tokens skips the whole-window attempt (`capture_window` guards on `!adapter.retains_element_tokens()`), so the same depth can be cheap and token-proven. The model is platform-conditional.
- **flow_traceability**: SF-02, EF-01, DF-01, DF-02.
- **boundary**: Uniqueness is at capture time; later duplicates make refs ambiguous.

### KO-07 — Ref identity is a closed, format-strict, content-minimal predicate

- **layer**: L2 · **epistemic**: pattern
- **claim**: A ref must be snapshot-qualified; identity compares only role/native-id/stable-name/description; value and bounds are never predicate fields; unreadable recorded fields are `Unknown`; secure fields remove value and related text. Exception: label-like roles may promote a value into the stable-name field (`stable_name(role, name, value)`), so "value excluded" is true of the predicate, not of every role's label extraction.
- **aggregation_rule**: `R3 Invariant Cluster` — EK-20 (qualified format), EK-22 (closed predicate), EK-28 (secure withholding) enforce one invariant across the ref format, comparison layer, and privacy layer: *identity must not depend on mutable geometry or freely varying value; the only value use is a role-programmed label promotion, not a general predicate.*
- **derivation.facts**: `EK-20, EK-22, EK-28`
- **why it is more than a list**: It connects format validation, predicate design, and secure-field withholding as three layers of the same invariant. A bare ref or a value-based predicate would each break it.
- **counterexample**: `stable_name(role, name, value)` can promote a value into the name field for some roles. That is a role-specific label rule, not general value identity; it is the exception that keeps the invariant from being an absolute "no value anywhere" claim.
- **flow_traceability**: DF-03, EF-01, PF-03.
- **boundary**: The predicate cannot distinguish two elements with identical structural fields; ambiguity is an explicit outcome.

### KO-08 — State substrates are bounded, namespaced, and lock-separated

- **layer**: L2 · **epistemic**: pattern
- **claim**: Refmaps, sessions, traces, and interaction leases each have their own namespace, size cap, and lock discipline. Refstore is transactional and retention-bounded; session manifest is size-capped; trace is append-only and event-capped. The interaction lease is Unix per-user machine-wide (`/tmp/agent-desktop-<uid>/interaction.lock`); Windows uses a named handle; macOS uses the Unix path. There is no single cross-platform lease object.
- **aggregation_rule**: `R3 Invariant Cluster` — EK-23 (refstore), EK-24 (session), EK-25 (lease), EK-26 (trace) enforce one cross-layer invariant: *durable state must be bounded and owned by exactly one namespace/lock.* The cluster spans data storage, session identity, mutual exclusion, and append-only logging.
- **derivation.facts**: `EK-23, EK-24, EK-25, EK-26`
- **why it is more than a list**: It explains why a trace cannot silently grow without bound, why a refmap cannot be saved concurrently, and why a lease is not session-scoped. The four substrates are different answers to the same failure mode: unbounded or ambiguous shared state.
- **counterexample**: Clipboard images default to a session/temp path but `--out` writes to a user-specified path outside these bounds. The invariant applies to runtime-managed state, not user-requested exports.
- **flow_traceability**: SF-01, SF-02, SF-06, MF-01..MF-05, AF-04.
- **boundary**: Count-based retention can delete an old but still useful snapshot; the live-token guard only protects tokens, not caller intent.

---

## L3 — Mechanisms and trade-offs

### KO-09 — Reads and writes have asymmetric evidence rules

- **layer**: L3 · **epistemic**: pattern
- **claim**: A read-only command with unknown delivery may settle to `not_delivered` because it sent nothing; a mutation with unknown delivery stays uncertain. Both may use a second observation, but only a mutation needs postcondition verification to move from unverified to verified. E2E applies the same asymmetry: it observes effects independently rather than trusting command success.
- **aggregation_rule**: `R1 Mechanism Cluster` — EK-15 (settlement), EK-16 (postcondition), EK-33 (E2E independent observation) are three instances of one mechanism: *the evidentiary burden depends on whether the operation can have side effects.* They occur in dispatch, action execution, and test harness.
- **derivation.facts**: `EK-15, EK-16, EK-33`
- **why it is more than a list**: It explains why "unknown" is treated differently for reads and writes, and why a test that only checks `ok` is structurally weaker than one that observes state.
- **counterexample**: A read-only command that writes a cache file (`clipboard-get --out`) is classified mutating, so it does not get the read-only settlement. The classification, not the command name, decides.
- **flow_traceability**: EF-02, EF-03, EF-05, CF-06.
- **boundary**: A postcondition read proves element state, not application commit; E2E can still miss asynchronous effects.

### KO-10 — Privacy boundaries are per evidence channel, not global

- **layer**: L3 · **epistemic**: pattern
- **claim**: Secure-field withholding, trace key redaction, and replay-artifact non-redaction are three different policies on three channels. The system does not claim a single "redaction" property; it scopes privacy to the channel that carries the evidence.
- **aggregation_rule**: `R4 Thematic Cluster` — EK-28 (secure fields), EK-26 (event trace), EK-27 (replay artifacts) cover complementary dimensions of one theme: *what is withheld depends on where the evidence goes.*
- **derivation.facts**: `EK-26, EK-27, EK-28`
- **why it is more than a list**: It explains why "the trace is redacted" and "the screenshot is not" can both be true, and why a consumer must treat exported HTML as sensitive even when the JSONL is safe.
- **counterexample**: A secret under an unlisted JSON key survives trace redaction; a benign string under `name` is removed. The policy is schema-based, not content-based.
- **flow_traceability**: PF-03, MF-03, MF-04, EF-06.
- **boundary**: Pixel-level secrets in screenshots are outside any redaction logic; artifacts are operator-owned.

### KO-11 — Platform divergence is a named capability boundary, not a bug list

- **layer**: L3 · **epistemic**: pattern
- **claim**: macOS AX, Windows UIA, and Linux AT-SPI differ in both observation and action; Linux further splits X11 vs Wayland. The system models these as `not_supported` capabilities and documented matrix rows, not as silent fallbacks. Release guards and headed downgrades are platform-specific consequences.
- **aggregation_rule**: `R3 Invariant Cluster` — EK-02 (compile-time selection), EK-34 (matrix), EK-11 (headed downgrade), EK-29 (release path) enforce one invariant: *a capability may be absent, but its absence must be explicit and localized.*
- **derivation.facts**: `EK-02, EK-11, EK-29, EK-34`
- **why it is more than a list**: It explains why the same command can work on macOS and refuse on Wayland without being "broken": the capability is absent, and the contract says so.
- **counterexample**: A command that silently degrades from semantic to physical input on Wayland would violate the model. The Linux skill documents `ACTION_NOT_SUPPORTED` for headed pointer instead.
- **flow_traceability**: CF-03, PF-02, AF-04, SF-05.
- **boundary**: Documentation can drift from implementation; this run verified key Linux claims in code but did not execute a Wayland session.

---

## L4 — Cognitive model

### KO-12 — Reliability is an evidence pipeline, not a success flag

- **layer**: L4 · **epistemic**: model
- **claim**: The project's reliability model is a pipeline of independent evidence producers: preflight actionability, hit-test proof, delivery disposition, postcondition verification, E2E independent observation, and probe ledgers. A single green flag is never the model's unit of truth; each stage has its own evidence and its own failure mode.
- **aggregation_rule**: `R2 Causal Chain Cluster` — EK-12/EK-13 (preflight/hit test) → EK-16 (postcondition) → EK-33 (E2E) → EK-32 (governance gates). The chain covers attempt → effect → independent confirmation → process enforcement.
- **derivation.facts**: `EK-13, EK-16, EK-32, EK-33`
- **why it is more than a list**: It predicts where a new feature must add evidence: a new action needs a requirement row, a postcondition rule, and an E2E observation, not just a dispatch arm. The model explains the repository's otherwise surprising density of probes and golden tests.
- **counterexample**: `receives_events` tolerates unknown, so the pipeline is not "all stages must be conclusive"; it is "each stage must classify its own uncertainty". A model that demanded proof at every stage would be wrong.
- **flow_traceability**: CF-06, EF-01..EF-06, AF-01.
- **boundary**: This is a model of the project's own design, not a claim that every command actually implements all stages.

---

## L5 — Principle candidate

### KO-13 — Agent automation should treat action success as a claim that requires an independently observable postcondition

- **layer**: L5 · **epistemic**: hypothesis (single-project support, not validated cross-project)
- **claim**: For GUI automation, the safe primitive is not "the command returned success" but "a second observation, on a channel independent of the command, confirms the intended state; when that observation is unavailable, the outcome is uncertain and not retry-safe."
- **aggregation_rule**: `R2 Causal Chain Cluster` — EK-14 (delivery lattice) → EK-15 (unknown settlement) → EK-16 (postcondition verification). The chain is complete for this project: claim → safe/unsafe classification → second observation.
- **derivation.facts**: `EK-14, EK-15, EK-16`
- **why it is more than a list**: It states the cross-cutting principle the three mechanisms implement: success is an evidentiary claim, not a return value.
- **counterexample**: A semantic action with no readable state (e.g., a fire-and-forget button) can be `delivered_verified` only if some postcondition is observable; otherwise it remains `delivered_unverified`. This limits the principle: it does not demand impossible observation, it demands honest labeling.
- **flow_traceability**: EF-02, EF-03, EF-05.
- **boundary**: **Not a validated Principle.** It is supported by one repository and one design lineage. It should be tested against at least one independent agent-desktop/browser-automation project before promotion. Cross-project evidence is a Candidate (`CAND-12`), not proof.

---

## Coverage and layer discipline

| Layer | KO | Epistemic status |
|---|---|---|
| L1 | KO-01..KO-04 | fact |
| L2 | KO-05..KO-08 | pattern |
| L3 | KO-09..KO-11 | pattern |
| L4 | KO-12 | model |
| L5 | KO-13 | hypothesis (explicitly not promoted) |

No L3/L4/L5 claim is written as a validated Fact. No KO deletes its EK members; all 39 EK remain in `02_engineering-knowledge/ek_graph.md`.

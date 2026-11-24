# Independent Validation — Blind Reconstruction and Adversarial Audit

- **Auditor stance**: read the repository at `ce2ddf0` independently before comparing against the Package. Did not use the Package's KO/EK conclusions as a source of truth.
- **Constraint**: no GUI session, no `cargo test`, no E2E, no MCP server in this environment. Runtime claims are not executable evidence.
- **Target**: `agent-desktop`, `ARCH-2026-11-25-001`.

## 1. Blind reconstruction (independent)

Without opening the Package, the repository's own entrypoints and docs support this reconstruction:

1. `main` builds one platform adapter by `#[cfg(target_os)]`, then hands a `Commands` enum to a direct-match dispatcher.
2. The dispatcher is the only command authority; batch and MCP are projections, not parallel implementations.
3. A snapshot produces a tree, a refmap keyed by snapshot id, and a `complete` flag. Tokenless refs depend on a whole-window uniqueness proof; without it, `identity_unproven` is surfaced and refs are refused.
4. Actions pass an actionability battery; physical pointer delivery adds a five-point hit-test. The only tolerated unknown is `receives_events` without occlusion evidence.
5. Results carry a five-state delivery disposition. `not_delivered` is the only safe-retry state. Mutating unknown stays unknown; read-only unknown may settle to `not_delivered`.
6. Stateful actions are verified by a fresh element read; writes are `element_value` with `application_commit:not_verified`.
7. Permission is a static command map; multi-agent identity is narrower than the map; clipboard is outside both.
8. Trace JSONL is key-redacted; screenshots/refmap artifacts are opt-in and explicitly unredacted.
9. The platform matrix is a product constraint (AX/UIA/AT-SPI; X11 vs Wayland), not an implementation afterthought.

This reconstruction independently arrives at the same top-level shape as the Package, which is a **coverage success**, not a correctness proof.

## 2. Adversarial checks and judgments

| ID | Target claim | Judgment | Evidence / reason |
|---|---|---|---|
| AUD-01 | KO-01 platform contract | **CONFIRMED** | `adapter/mod.rs` trait composition; `build_adapter` cfg arms; FFI wraps same trait |
| AUD-02 | KO-02 single command authority | **CONFIRMED** | `dispatch` match, batch `parse_command`, MCP `catalog`; `sandbox exec` exception is already in boundary |
| AUD-03 | KO-03 authority preflight | **CONFIRMED** | `policy_for`, `preflight`, `prepare` per-entry; clipboard `None` is explicit |
| AUD-04 | KO-04 delivery lattice | **CONFIRMED** | `DeliverySemantics::retry`, serde mismatch rejection, `settle_unknown_delivery` |
| AUD-05 | KO-05 actionability battery | **CONFIRMED** | `requirements.rs` order; `evaluate.rs` early return; `receives_events` append |
| AUD-06 | KO-06 snapshot proof | **CONFIRMED** | `project_window` keeps `structurally_complete`; `capture_window` 3/4 attempt; `proves_node` |
| AUD-07 | KO-07 identity predicate | **PARTIALLY_CONFIRMED** | Predicate excludes value/bounds as fields, but `stable_name(role, name, value)` can promote a value into the name field for label-like roles. KO wording "value excluded" is too absolute; boundary mentions the exception, but the claim reads absolute. |
| AUD-08 | KO-08 state substrates | **PARTIALLY_CONFIRMED** | Refstore/session/trace bounds are code facts. "Machine-wide lease" is Unix-specific (`/tmp/agent-desktop-<uid>`); Windows uses a named handle; macOS uses Unix path. The claim's scope is broader than the evidence. |
| AUD-09 | KO-09 read/write asymmetry | **CONFIRMED** | `settle_unknown_delivery` read_only branch; `post_action` second read; E2E `reliability.sh:92` |
| AUD-10 | KO-10 privacy channel scoping | **CONFIRMED** | `trace_sanitize` key list; `CONCEPTS.md:179` artifact warning; secure-field code |
| AUD-11 | KO-11 platform divergence | **PARTIALLY_CONFIRMED** | Code has X11-specific modules and Wayland refusal; matrix is documented. Not executed on either display server. |
| AUD-12 | KO-12 evidence pipeline | **CONFIRMED** | The model is explicitly a model; stages exist in code/tests. It does not claim all commands implement every stage. |
| AUD-13 | KO-13 L5 principle | **CONFIRMED AS HYPOTHESIS** | It is labeled hypothesis and single-project; not a validated Principle. |
| AUD-14 | EK-16 "mismatch → ACTION_FAILED" | **OVER_GENERALIZED** | `post_action.rs` exception: a physical text mismatch (`typed_by_keys`) returns `delivered_unverified` with `postcondition_mismatch`, not `ACTION_FAILED`. Only non-physical mismatch returns `verification_error(ActionFailed)`. |
| AUD-15 | EK-17 success shape undocumented | **CONFIRMED** | `docs/json-output.md` documents only `error.details.after_action`; `helpers.rs` nests success under `data.after_action`; tests pin shape |
| AUD-16 | EK-19 3/4 deadline split | **CONFIRMED** | `deadline.remaining() * 3 / 4` in `capture_window`; fallback shallow walk uses remaining deadline |
| AUD-17 | EK-29 release guards | **PARTIALLY_CONFIRMED** | Linux order and Windows per-key `GetAsyncKeyState` verified; macOS guard verified. Cross-platform runtime behavior not executed. |
| AUD-18 | EK-30 clipboard asymmetry | **CONFIRMED** | `policy_for` None; `clipboard_get` no lease; `clipboard_set` lease; `get_clipboard_content` signature has no lease |
| AUD-19 | EK-38 envelope shape | **CONFIRMED** | `output.rs` constructors; drift test; docs 2.6 |
| AUD-20 | CAND-10 eager `read_name` | **NEEDS_HUMAN_REVIEW** | Rust argument evaluation means `read_name` runs before `label_from_target`; output is withheld. Severity depends on provider behavior and any telemetry, neither measured. |
| AUD-21 | CAND-01 clipboard authority | **NEEDS_HUMAN_REVIEW** | Code facts confirmed; intent unknown. Whether it is a security gap is a product/threat-model decision. |
| AUD-22 | CAND-06 `receives_events` unknown exception | **CONFIRMED** | `is_blocking` exception; `require_receives_events` returns Ok on Unknown. Severity unmeasured. |
| AUD-23 | CAND-07 Batch always mutating | **CONFIRMED** | `is_mutating` includes `Batch(_)`; no payload inspection in dispatch |
| AUD-24 | Radar version/count conflict | **CONFIRMED / DOWNGRADED** | Repository says 1.3.1/61/57; radar says 1.2.0/60/56. Package correctly downgrades radar data. |

## 3. Three strongest confirmations

1. **Delivery retry invariant is exactly as claimed.** The enum mapping, serde rejection, and settlement rule agree; this is the most load-bearing safety claim in the package.
2. **Batch preflight is genuinely all-or-nothing at validation time.** `prepare` validates every entry and context before returning; this is stronger than a per-entry lazy check.
3. **Tokenless ref publication is genuinely proof-gated.** `CaptureProof::proves_node` requires complete whole-window evidence, exactly one predicate match, and no unread recorded field.

## 4. Three material errors found

1. **AUD-14 / EK-16 overgeneralization.** "A mismatch is `ACTION_FAILED`" ignores the physical-text exception that downgrades to `delivered_unverified`. This would mislead a reader into thinking every typed-text mismatch is a hard failure.
2. **AUD-07 / KO-07 overstatement.** "Value and bounds are excluded" is true as a predicate-field rule, but `stable_name` can promote a value into the name field for label-like roles. The exception is in EK-22's boundary, but KO-07 reads absolute.
3. **AUD-08 / KO-08 scope error.** "Machine-wide interaction lease" is Unix-specific; Windows uses a named handle and macOS uses the Unix path. The claim should say "Unix machine-wide; platform-specific otherwise."

## 5. Omissions

- **O-01**: EK-16 does not mention the `physical_text_mismatch` downgrade or the `postcondition_mismatch` reason. This is the missing exception behind AUD-14.
- **O-02**: EK-22 mentions stable-name promotion, but KO-07 does not carry that exception forward. Cross-layer traceability lost a boundary.
- **O-03**: KO-08 says "machine-wide" without qualifying Unix vs Windows. The FFI doc says mutations are serialized by "the adapter's cross-process interaction lease", which is true per platform but not one global mechanism.
- **O-04**: The package does not explicitly separate `satisfied_without_delivery` (already satisfied, e.g. close-app on an exited app) from "not delivered due to failure". `ActionResult::satisfied_without_delivery` uses `not_delivered` for both, which is semantically safe but can read as failure. This is a documentation nuance, not a code bug.

## 6. Contradictions

None found. No claim directly contradicts code. The three material issues are overgeneralizations/scope, not contradictions.

## 7. Epistemic audit

| Layer | Check | Result |
|---|---|---|
| L1 | Every claim traceable to code/doc | PASS |
| L2 | Patterns have ≥2 instances or explicit exception | PASS with KO-07 caveat |
| L3 | Mechanism claims do not smuggle principles | PASS |
| L4 | Model is labeled model, not fact | PASS |
| L5 | Hypothesis not promoted to Principle | PASS |
| Candidates | Not written as facts/defects | PASS, except CAND-10 severity language should stay conditional |

## 8. Benchmark case

**Case**: `agent-desktop` — "evidence pipeline under unobservable UI state".

Use it as a benchmark for any agent-desktop/browser-automation project with five checks:

1. Does the project have a delivery state richer than `ok/error`, with retry safety derived rather than caller-set?
2. Does a mutating action's unknown outcome remain unsafe/unknown instead of defaulting to retry?
3. Does a ref/selector system distinguish "unique at capture" from "resolved now", and expose ambiguity/staleness as distinct errors?
4. Does the project have at least one independent effect observation in its test harness, rather than trusting the command's own success?
5. Does the platform capability matrix name absent capabilities instead of silently degrading?

**Scoring**: 5/5 = same class; 3-4/5 = partial; <3 = different maturity. This case is intended for cross-project comparison in a future run; it is not applied here.

## 9. Audit verdict

- **Overall**: `PARTIALLY_CONFIRMED` at the Package level, with **no Contradictions**.
- **Required fixes before corpus commit**:
  1. EK-16: add physical-text mismatch exception and correct the absolute "mismatch → ACTION_FAILED".
  2. KO-07: qualify value exclusion with role-specific stable-name promotion.
  3. KO-08: scope interaction lease claim to Unix/Windows/macOS accurately.
  4. 00 Overview: replace pre-audit metric guesses with counted values.
- **No fix** should alter the target repository or the production skill.

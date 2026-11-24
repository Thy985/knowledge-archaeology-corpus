# 05 — Candidates

Candidates are unresolved by design. They are not facts, defects, or principles. Each entry states what is known, what is missing, and what evidence would settle it.

---

## CAND-01 — Clipboard is outside the accessibility permission and agent-id gates

- **Type**: authority / privacy
- **Observation**: `policy_for` maps `ClipboardGet`, `ClipboardSet`, `ClipboardClear` to `None`; `requires_permission_report` is false for them; `clipboard-set` takes an interaction lease but `clipboard-get` does not; `get_clipboard_content` has no `InteractionLease` parameter.
- **Why it might be intentional**: Clipboard is an OS-level data channel, not a UI-semantic action. Reading it does not require AX/UIA/AT-SPI. Forcing an accessibility permission would be technically wrong.
- **Why it might be a gap**: A headless agent can read arbitrary clipboard text, including passwords copied by the user, without any permission prompt. `clipboard-set` can overwrite the user's clipboard under a multi-agent session without `--agent-id`. The lease asymmetry means a set can serialize with other desktop mutations while a get can race them.
- **What would settle it**: A maintainer statement of intent, or a threat model section in `SECURITY.md`. A code change adding a clipboard-specific permission class would be the other settlement.
- **Status**: unresolved; no code change made (target repo read-only).

## CAND-02 — Successful post-action wait nests the action result under `data.after_action`, but the success shape is not documented

- **Type**: contract / documentation
- **Observation**: `helpers.rs` returns top-level `data` as the snapshot with `data.after_action` for the action result. `docs/json-output.md` documents `error.details.after_action` for failures but does not document the success shape.
- **Evidence**: `crates/core/src/commands/helpers.rs`, `helpers_ref_action_wait_result_tests.rs` (shape pinned in tests).
- **Risk**: A caller that reads `data.disposition` after a successful action+wait will miss it; it must read `data.after_action.disposition`. This is a discoverability gap, not a proven runtime bug.
- **What would settle it**: A documented example in `docs/json-output.md`, or a contract test that asserts the success shape from the doc.
- **Status**: candidate; documentation gap.

## CAND-03 — `CONCEPTS.md` says secure fields resolve by "content-free fingerprint"; no literal fingerprint function was found

- **Type**: prose vs implementation
- **Observation**: `CONCEPTS.md:219` says the read side "withholds value and related text evidence and resolves by content-free fingerprint". The ref path uses `FullPredicate` over role/native-id/stable-name/description; no hash/digest/fingerprint function exists in `crates/core/src/refs`. Windows `name_evidence.rs` withholds the label for `IsPassword`; Linux withholds value; macOS withholds value. The identity is content-free in the sense that it excludes value, but it is not a fingerprint in the cryptographic sense.
- **Why it matters**: A reader may expect a stable digest that survives label changes; the code instead refuses/degrades on missing evidence. That is a different privacy/identity trade.
- **What would settle it**: Either a code comment defining "fingerprint" as "structural predicate", or an actual digest implementation. The package treats the prose as a metaphor, not a Fact.
- **Status**: candidate; do not write as a defect.

## CAND-04 — Radar card numbers are stale relative to the repository

- **Type**: provenance
- **Observation**: The radar card / scan log says npm `1.2.0`, `60 command names`, `56 operational`, `~50k weekly downloads`. The repository at `ce2ddf0` says `1.3.1`, `61 command names`, `57 operational`; `npm view` confirms latest `1.3.1` and `time.modified 2026-11-13`. Download volume was not independently verified.
- **Handling**: The package uses repository facts and marks the radar numbers as second-hand/stale. This is a data-quality finding about the discovery layer, not about `agent-desktop`.
- **Status**: resolved for this package (repository wins); radar card should be corrected in KnowlegeMap on a future pass.

## CAND-05 — `complete:true` can coexist with `identity_unproven:true`

- **Type**: contract clarity
- **Observation**: `docs/json-output.md:138` explicitly says "`complete` still describes the presented tree" when `identity_unproven: true`. The two fields answer different questions. A consumer that treats `complete:true` as "all refs usable" will be wrong.
- **Why it is not a bug**: The distinction is documented and tested (`identity_unproven` is emitted as a boolean; the reason is only in trace). The risk is caller misinterpretation.
- **What would settle it**: A consumer-facing rule in the docs: "`complete` describes the tree; `identity_unproven` describes ref usability; both must be checked." That is a documentation improvement, not a code change.
- **Status**: candidate; contract clarity.

## CAND-06 — `receives_events` is the only actionability check that tolerates Unknown

- **Type**: policy exception
- **Observation**: `ActionabilityReport::is_blocking` treats `Unknown` for `receives_events` as non-blocking unless `hit_test.occluded > 0`. All other checks block on `Unknown`. `require_receives_events` returns `Ok` on `Unknown`/`PlatformNotSupported`.
- **Why it might be intentional**: Hit testing is unavailable on some platforms; refusing every physical action because the OS cannot answer would make the tool unusable. The check answers "is there positive occlusion evidence", not "is the target provably reachable".
- **Risk**: A target that is actually occluded but whose hit test is unknown will be clicked. The package cannot prove this happens; it is a policy asymmetry.
- **What would settle it**: A probe on a platform with unsupported hit testing, measuring whether unknown targets are occluded. Not run in this environment.
- **Status**: candidate; explicit exception, not a confirmed defect.

## CAND-07 — `Batch` is always classified mutating, even when every entry is read-only

- **Type**: runtime classification
- **Observation**: `Commands::Batch(_)` is in the `is_mutating` true arm. `policy_for(Batch)` is `None`. In `dispatch`, a headed session with cursor overlay enabled hides the overlay because `cmd.is_mutating()` is true; the batch also takes the mutating command scope.
- **Why it might be intentional**: Batch can contain mutations; treating it as potentially mutating is conservative and avoids inspecting the payload before deciding scope.
- **Risk**: A read-only batch hides the cursor overlay and is treated as a mutation for scope/cleanup, which is unnecessary. It also does not get the read-only unknown-settlement at the batch level, though each entry has its own disposition.
- **What would settle it**: Inspecting `batch` entries during dispatch and deriving `read_only` from all entries. That is a design change, not a local fix.
- **Status**: candidate; classification conservatism.

## CAND-08 — MCP spawns a fresh process per call, inheriting the server environment

- **Type**: runtime / performance
- **Observation**: `call.rs` uses `std::env::current_exe()` and constructs a child `Command`; globals are prepended except for `sandbox exec`; child stdin is null. The child inherits the server's environment by default.
- **Why it might be intentional**: Process isolation prevents one tool call from corrupting server state, and reusing the binary avoids a second RPC implementation.
- **Risk**: Per-call cold start and environment inheritance (including `AGENT_DESKTOP_*` variables) are not measured in this run. A long-running MCP session pays repeated process startup. Whether this matters at 8 concurrent calls is unknown.
- **What would settle it**: A benchmark of `desktop_snapshot` over 100 calls with warm/cold server, and an env-diff test. Not run here.
- **Status**: candidate; needs runtime measurement.

## CAND-09 — Trace JSONL is key-redacted, replay artifacts are not

- **Type**: privacy boundary
- **Observation**: `trace_sanitize` redacts a fixed key list; `CONCEPTS.md:179` says artifacts are unredacted and may appear in exported HTML. Both live under the same session `trace/` directory.
- **Why it might be intentional**: Artifacts are opt-in and operator-owned; redacting pixels would break replay/debug value. The docs warn.
- **Risk**: A consumer that exports a session directory may treat the whole directory as safe because JSONL is redacted. The warning is prose, not an enforced export policy.
- **What would settle it**: An export command that labels artifact sensitivity, or a `.gitignore`/packaging rule. Not present in this snapshot.
- **Status**: candidate; documentation/export-policy gap.

## CAND-10 — `read_label` evaluates `read_name` even when the target is secure

- **Type**: privacy side effect
- **Observation**: `label_from_target(is_secure(&target), read_name(&target))` evaluates both arguments before the call. Rust evaluates left-to-right, so `read_name` runs even when `is_secure` is true; `label_from_target` then returns `Withheld` without using it.
- **Why it might be intentional**: Simplicity; a single call site. The output is still withheld, so no value reaches the tree.
- **Risk**: The provider is asked for `Name` on a secure element unnecessarily. If a provider returns sensitive content in `Name` and a telemetry layer observes the read (not the returned tree), that is an unnecessary exposure surface. The code does not log the raw name here.
- **What would settle it**: Refactoring to call `read_name` only after `is_secure` is false, plus a test asserting no `Name` read for secure targets. Not done (read-only repo).
- **Status**: candidate; low-severity, unverified runtime effect.

## CAND-11 — Linux X11/Wayland matrix is documented and partly coded, but not executed here

- **Type**: platform verification
- **Observation**: `skills/agent-desktop-linux/SKILL.md` states observation/semantic refs work on X11 and Wayland; pointer gestures, wheel scrolling, window control, `focus-window`, headed shell input require X11; Wayland headed pointer returns `ACTION_NOT_SUPPORTED`. The Linux adapter has X11-specific modules and a Wayland refusal path. This run had no GUI session and did not execute either.
- **Why it matters**: The matrix is a product claim with high user impact. Code presence is not runtime proof.
- **What would settle it**: Run `linux-e2e.yml` on an X11 runner and a Wayland runner, or a manual probe with `probes/linux`. Not done in this run.
- **Status**: candidate; documentation + code evidence, runtime unverified.

## CAND-12 — The L5 principle is single-project and may not generalize

- **Type**: epistemic
- **Observation**: KO-13 ("action success requires an independently observable postcondition") is supported by `agent-desktop`'s delivery lattice, settlement rule, and postcondition model. It has not been tested against a browser-automation or RPA project with a different evidence model.
- **Why it might not generalize**: A system with transactional app APIs may have reliable commit receipts and no need for a second UI observation. A system with purely visual output may have no readable postcondition at all.
- **What would settle it**: A cross-project comparison using the same rubric (delivery states, retry safety, postcondition availability). That is a future Candidate, not this run's result.
- **Status**: explicit hypothesis; must not be written as a validated Principle.

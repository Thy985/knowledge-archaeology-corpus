# 02 — Engineering Knowledge Graph

**Graph contract**: v3.1 (`knowledge-archaeology/contracts/ek-graph-schema.md`). Every EK has ≥1 typed link; every EK belongs to at least one KO cluster in `03_knowledge-layer/`. Links are declared on the EK that owns the relation; the reciprocal edge is implied.

**Edge vocabulary**: `mechanism` (same implementation pattern), `subsystem` (same module family), `causal` (A triggers B), `dependency` (A's correctness relies on B), `constraint` (A bounds/preconditions B), `contrast` (same problem, different policy).

---

## Architecture and execution

### EK-01 — Core declares the platform capability contract; platforms only implement it

- **type**: Architecture · **layer**: L2 · **epistemic**: fact
- **claim**: `PlatformAdapter` is a composition of four operation traits (`ObservationOps`, `ActionOps`, `InputOps`, `SystemOps`) with a blanket impl. The core owns the contract; each platform crate implements the same four faces.
- **mechanism**: Every sub-trait method has a default `not_supported(...)` body. A platform must override a capability to claim it. The blanket impl means no separate `PlatformAdapter` implementation is written per platform.
- **evidence**: EV-001 (`crates/core/src/adapter/mod.rs:PlatformAdapter`), EV-002 (`src/main.rs:build_adapter`), EV-019 (`AGENTS.md` core-isolation rule).
- **boundary**: The contract is structural, not behavioral: it does not prove a platform implementation is correct, only that it opted in.
- **links**:
  - `mechanism → EK-02` — both implement the "core owns contract, platform owns capability" split.
  - `dependency → EK-34` — the capability matrix is the runtime truth this contract abstracts.
  - `constraint → EK-10` — policy execution may only use capabilities the contract exposes.

### EK-02 — One adapter is selected at compile time, and unsupported targets fail the build

- **type**: Architecture · **layer**: L1 · **epistemic**: fact
- **claim**: `build_adapter()` has exactly three `#[cfg(target_os)]` arms (macOS, Windows, Linux) and a `compile_error!` fallback. There is no runtime platform registry.
- **mechanism**: The binary compiles against one platform crate; `ci.yml` cross-compiles core and checks `cargo tree -p agent-desktop-core` has no platform dependency. On Linux, `release_input_on_signals()` runs before `LinuxAdapter::new()`.
- **evidence**: EV-002 (`src/main.rs:358-380`), EV-003 (`src/main.rs:main` pipeline), EV-019 (`AGENTS.md`), EV-042 (`ci.yml` core isolation).
- **boundary**: Compile-time selection means one binary cannot switch platforms at runtime; a universal binary would need a different dispatch layer.
- **links**:
  - `mechanism → EK-01` — selection chooses the contract implementer.
  - `dependency → EK-34` — the matrix explains why a single adapter is enough per host.
  - `constraint → EK-32` — CI enforces the isolation this design assumes.

### EK-03 — Dispatch is a direct match over a closed command enum, not a registry

- **type**: Architecture · **layer**: L2 · **epistemic**: fact
- **claim**: `src/dispatch/mod.rs` matches every `Commands` variant to a core command function or binary-layer module. There is no `Command` trait, no dynamic registration, no plugin lookup.
- **mechanism**: `dispatch` first computes `read_only = !cmd.is_mutating()`, then calls `dispatch_command`, and only then applies `settle_unknown_delivery(error, read_only)`. The match is exhaustive at compile time.
- **evidence**: EV-004 (`src/dispatch/mod.rs:dispatch`, `dispatch_command`), EV-045 (`src/cli/mod.rs:is_mutating`), EV-019 (`AGENTS.md` "no command registry").
- **boundary**: A new command requires edits in core command, CLI, dispatch, policy, batch parser, docs counts, and possibly adapter; the compiler catches missing dispatch arms but not missing docs.
- **links**:
  - `subsystem → EK-04` — batch reuses this dispatcher.
  - `subsystem → EK-06` — MCP derives its tools from the same CLI tree.
  - `dependency → EK-02` — dispatch is monomorphic over the compiled adapter.

### EK-04 — Batch is the same command surface executed sequentially and non-atomically

- **type**: Runtime · **layer**: L2 · **epistemic**: fact
- **claim**: Batch entries deserialize into the same `Commands` enum and execute through the same `dispatch`; batch is explicitly `atomic:false, order:sequential, batch_retries:false`, while per-command retry contracts are preserved.
- **mechanism**: `parse_command` handles snapshot/find/screenshot/get/is/click/…/clipboard/wait/status/permissions/version/skills/session/trace. It rejects nested batch, cursor-overlay, sandbox; `type`/`set-value` reject `--stdin` inside batch. Execution has a batch deadline and output-byte budget; over-budget output becomes `not_delivered` or `delivery_uncertain` depending on completed count.
- **evidence**: EV-005 (`src/batch/mod.rs:parse_command`), EV-006 (`src/batch/execution.rs` semantics block), EV-007 (`src/batch/preparation.rs`).
- **boundary**: Batch cannot roll back a delivered action. `stop_on_error` stops later entries but does not undo earlier ones.
- **links**:
  - `mechanism → EK-03` — same enum, same dispatcher, no separate batch command language.
  - `dependency → EK-05` — all entries are preflighted before any executes.
  - `dependency → EK-35` — wait-event baselines are captured before the preceding action.
  - `contrast → EK-07` — MCP also re-enters the binary but per call, not per batch.

### EK-05 — Batch preflights every entry before the first action runs

- **type**: Engineering · **layer**: L2 · **epistemic**: fact
- **claim**: `prepare()` parses and validates all entries first, runs `command_policy::preflight` per entry, and only returns a `Vec<PreparedCommand>` when every entry is valid. A single bad permission or invalid arg prevents the whole batch from starting.
- **mechanism**: `prepare` enforces 1 MiB input / 64 entries, then `parse_one` per entry: session id validation, `parse_command`, `preflight`. After parsing, it builds per-item `CommandContext` via `for_batch_item` and runs `preflight_context` (agent-id gate). Errors are located with `batch_index` and forced `not_delivered`.
- **evidence**: EV-007 (`src/batch/preparation.rs:prepare`), EV-012 (`src/command_policy/mod.rs:preflight_context`).
- **boundary**: Preflight is static; it cannot know that entry 3 will hit a stale ref after entry 2 mutates the UI. That remains a runtime failure with its own disposition.
- **links**:
  - `dependency → EK-03` — preflight uses the same command enum as dispatch.
  - `constraint → EK-04` — no entry executes until the whole plan validates.
  - `subsystem → EK-08` — permission mapping is the core of per-entry preflight.

### EK-06 — MCP tools are a projection of the clap command tree

- **type**: Architecture · **layer**: L2 · **epistemic**: fact
- **claim**: `catalog::tools()` walks `Cli::command()` leaves; each leaf becomes `desktop_<path>` with `-`→`_`; `inputSchema` is derived from clap args/help; hidden `--stdin` and globals are excluded; `additionalProperties:false`.
- **mechanism**: The MCP catalog is generated at runtime from the same clap definition the CLI uses. There is no hand-maintained MCP tool list. `mcp` is mounted on the root command and is not in `Commands`, so it is not a tool, batch entry, or counted command.
- **evidence**: EV-008 (`src/mcp/mod.rs`), EV-009 (`src/mcp/catalog.rs`), EV-019 (`AGENTS.md` MCP-not-a-second-surface).
- **boundary**: Schema fidelity depends on clap metadata quality; a command whose semantics are not expressible in flags will expose a weak tool schema.
- **links**:
  - `mechanism → EK-03` — both derive authority from one enum/tree.
  - `dependency → EK-07` — catalog entries execute by re-entering the binary.
  - `contrast → EK-31` — skills are also embedded, but as static prose rather than generated schema.

### EK-07 — Each MCP call spawns the same binary as a child process

- **type**: Runtime · **layer**: L2 · **epistemic**: fact
- **claim**: `call.rs` converts JSON arguments to argv and runs `std::env::current_exe()`; the child's stdin is null; globals are prepended except for `sandbox exec`; `sandbox exec` passes through guest stdout/exit code while other commands parse the JSON envelope.
- **mechanism**: `MAX_CONCURRENT_CALLS = 8`, `MAX_MESSAGE_BYTES = 4 MiB`, `TOOLS_TTL_MS = 86_400_000`. `Calls::admit` rejects same-id re-entry; `notifications/cancelled` cancels the child; busy returns a concurrency-limit message.
- **evidence**: EV-010 (`src/mcp/call.rs`), EV-008 (`src/mcp/mod.rs` constants and server loop).
- **boundary**: Per-call process isolation avoids shared mutable state but pays cold start and inherits only the environment the server passes; it is not a sandbox.
- **links**:
  - `dependency → EK-03` — child executes the same dispatch.
  - `dependency → EK-06` — argv is built from the generated tool schema.
  - `constraint → EK-25` — child actions still contend on the machine-wide lease.

### EK-08 — Permission need is a per-command map, checked before dispatch

- **type**: Permission · **layer**: L1 · **epistemic**: fact
- **claim**: `policy_for(cmd)` returns `None`, `Accessibility`, `ScreenRecording`, or both. `preflight` converts a denied need into `PERM_DENIED` with `not_delivered`. `requires_permission_report` requests a report for non-`None` policies plus `Status`, `Permissions`, `Batch`.
- **mechanism**: Observation, ref actions, keyboard/mouse, app/window mutation, and notification mutation map to `Accessibility`. Screenshot maps to both only when scoped to an app/window; unscoped screenshot maps to `ScreenRecording`. Clipboard maps to `None`.
- **evidence**: EV-011 (`src/command_policy/mod.rs:policy_for`), EV-012 (`preflight`, `requires_permission_report`).
- **boundary**: The map is a static declaration; a command that internally calls a more privileged operation must declare the union itself. The map is not derived from adapter calls.
- **links**:
  - `subsystem → EK-09` — agent-id gate builds on this map.
  - `subsystem → EK-30` — clipboard is the notable `None` exception.
  - `constraint → EK-36` — screenshot's scope changes its permission need.

### EK-09 — Multi-agent identity is required only for mutating accessibility commands

- **type**: Permission · **layer**: L1 · **epistemic**: fact
- **claim**: `agent_id_missing` is true only when `multi_agent && agent_id.is_none() && cmd.is_mutating() && policy_for(cmd) == Accessibility`. Read-only accessibility commands and mutating non-accessibility commands do not require `--agent-id`.
- **mechanism**: `preflight_context` runs after `preflight`; it rejects with `INVALID_ARGS` and a suggestion to set `--agent-id` or `AGENT_DESKTOP_AGENT_ID`.
- **evidence**: EV-012 (`src/command_policy/mod.rs:agent_id_missing`), EV-045 (`src/cli/mod.rs:is_mutating`).
- **boundary**: `Batch(_)` is always `is_mutating` but has `policy_for == None`, so the gate does not apply at the batch level; each entry is preflighted separately, so an accessibility action inside batch is still caught.
- **links**:
  - `dependency → EK-08` — the gate uses the permission need as its scope.
  - `constraint → EK-30` — clipboard-set is mutating but not gated.
  - `mechanism → EK-25` — both serialize concurrent desktop mutation, at different layers.

### EK-10 — Headless is the default; headed is an explicit policy, not an implicit fallback

- **type**: Agent · **layer**: L2 · **epistemic**: fact
- **claim**: `Action::base_interaction_policy` defaults to `headless` except `PressKey`, which defaults to `focus_fallback`. `InteractionPolicy` is three-valued: `headless(false,false)`, `focus_fallback(true,false)`, `headed(true,true)`. `join` is OR.
- **mechanism**: Headless means no cursor move and no physical event; semantic platform actions are preferred. `focus_fallback` allows focus but not cursor. Headed allows cursor and physical input.
- **evidence**: EV-013 (`crates/core/src/actions/interaction_policy.rs`), EV-014 (`crates/core/src/actions/action.rs` `Action::base_interaction_policy`), EV-019 (`CONCEPTS.md`).
- **boundary**: A semantic action may still be unavailable; absence leads to refusal, not silent physical fallback. `focus_fallback` is an internal third state, not exposed as a user flag.
- **links**:
  - `constraint → EK-11` — headed requirement is evaluated inside this policy.
  - `dependency → EK-37` — the action capability table decides semantic vs physical.
  - `contrast → EK-12` — the battery tests the chosen policy but does not choose it.

### EK-11 — A platform can downgrade a headed request to semantic evaluation

- **type**: Agent · **layer**: L2 · **epistemic**: fact
- **claim**: In `build_report_with_requirement`, if the adapter returns `HeadedRequirement::FocusedApplication`, the request is cloned with `policy = headless` for actionability evaluation. The action may then pass through semantic capability rather than requiring a focused window/cursor.
- **mechanism**: Only headed requests query `adapter.headed_requirement(handle, request, deadline)`. A `FocusedApplication` answer means "this target does not need the cursor/focus machinery"; evaluation proceeds with headless semantics. This is an adapter-specific fact, not a global rule.
- **evidence**: EV-015 (`crates/core/src/actionability/evaluate.rs:build_report_with_requirement`), EV-014 (`Action::headed_requirement`), EV-034 (`skills/agent-desktop-linux/SKILL.md` headed constraints).
- **boundary**: The downgrade affects evaluation, not the caller's intent; the action still runs under the original request. It is not a license to silently use physical input.
- **links**:
  - `dependency → EK-34` — platform capability determines whether the downgrade is valid.
  - `mechanism → EK-10` — same three-state policy.
  - `constraint → EK-12` — the battery receives the rewritten request.

### EK-12 — Actionability is an ordered battery with early refusal

- **type**: Engineering · **layer**: L2 · **epistemic**: fact
- **claim**: Requirements are per-action: visible/stable/enabled/editable/action_supported/policy, plus physical `receives_events`. Checks are appended in that order; the first non-Pass returns immediately. Physical pointer delivery appends `receives_events` only after all prior checks pass.
- **mechanism**: `visible` may skip geometry for direct semantic pointer/text delivery. `stability` uses `stability_without_pointer` for semantic delivery. `receives_events` is checked only for `PointerDelivery::Physical`. `finish` maps non-actionable reports to a terminal code (default `ACTION_FAILED`) and `not_delivered`.
- **evidence**: EV-015 (`crates/core/src/actionability/evaluate.rs`), EV-034 (`crates/core/src/actionability/requirements.rs`), EV-019 (`CONCEPTS.md` battery).
- **boundary**: The battery proves preconditions, not effect. Passing it means "allowed to try", not "action succeeded".
- **links**:
  - `dependency → EK-13` — physical delivery adds the hit-test check.
  - `dependency → EK-10` — policy is one of the checks.
  - `subsystem → EK-11` — headed downgrade can change the battery's inputs.

### EK-13 — `receives_events` is a five-point hit-test proof with a narrow Unknown exception

- **type**: Engineering · **layer**: L1 · **epistemic**: fact
- **claim**: The check samples bounds center plus four quadrant points (0.25/0.75). One `ReachesTarget` passes. Any occluder records the first occlusion. If hit testing is unsupported or inconclusive, the check is `Unknown`; `ActionabilityReport::is_blocking` treats `Unknown` for `receives_events` as non-blocking **unless** occlusion evidence exists.
- **mechanism**: `HitTestEvidence { attempted, unknown, occluded }`. Mixed unknown+occluded becomes `Unknown`. `require_receives_events` returns `Ok` for `Unknown`/`PlatformNotSupported` and only refuses on a concrete occluder.
- **evidence**: EV-016 (`crates/core/src/actionability/receives_events.rs`), EV-015 (`ActionabilityReport::is_blocking`).
- **boundary**: This is the only check allowed to pass on unknown. It means "cannot prove occlusion", not "proved unobstructed".
- **links**:
  - `dependency → EK-12` — it is the final physical-delivery gate.
  - `mechanism → EK-16` — both are "observation that can be inconclusive" models.
  - `constraint → EK-15` — an inconclusive hit test must not be reported as delivered-verified.

### EK-14 — Delivery is a five-state lattice with a derived retry flag

- **type**: Runtime · **layer**: L1 · **epistemic**: fact
- **claim**: `DeliverySemantics` has `Unknown`, `NotDelivered`, `DeliveryUncertain`, `DeliveredUnverified`, `DeliveredVerified`. `not_delivered` → `retry: safe`; every other known state → `retry: unsafe`; unknown → unknown. Deserialization rejects a mismatched delivery/retry pair.
- **mechanism**: The state is produced by action execution (`OperationReport` / `ActionStep`), not by the caller. `recovery_for_code` only emits a recovery hint when `retry == Safe`. `pre_dispatch_error` forces `not_delivered`; post-dispatch read-only unknown may settle to `not_delivered`.
- **evidence**: EV-017 (`crates/core/src/actions/delivery_semantics.rs`), EV-018 (`src/dispatch/disposition.rs`), EV-038 (`crates/core/src/response/output.rs`).
- **boundary**: `delivered_unverified` is still a delivery; it is not retry-safe. `delivered_verified` means a postcondition was observed, not that the application committed.
- **links**:
  - `mechanism → EK-15` — settlement applies the lattice to error paths.
  - `dependency → EK-38` — the envelope carries the disposition.
  - `contrast → EK-16` — verification upgrades unverified, but only with evidence.

### EK-15 — Unknown delivery is collapsed to safe only when nothing was sent

- **type**: Runtime · **layer**: L2 · **epistemic**: fact
- **claim**: `settle_unknown_delivery(error, read_only)` converts an `Unknown` disposition to `NotDelivered` only when `read_only` is true or the code is `INVALID_ARGS`. Mutating commands keep unknown. A non-read-only error without a disposition slot becomes an adapter `Internal` error.
- **mechanism**: `dispatch` computes `read_only = !cmd.is_mutating()` before command execution. `INVALID_ARGS` is considered "never reached the application" regardless of command class. `Io`/`Json`/`Internal` errors are wrapped when read-only; otherwise left as-is.
- **evidence**: EV-018 (`src/dispatch/disposition.rs:settle_unknown_delivery`), EV-004 (`src/dispatch/mod.rs`), EV-045 (`src/cli/mod.rs:is_mutating`).
- **boundary**: The "nothing sent" inference depends on the command classification. A command misclassified as read-only would get an unsafe retry hint.
- **links**:
  - `dependency → EK-14` — settles within the lattice.
  - `constraint → EK-16` — no verification can rescue an unknown mutation.
  - `subsystem → EK-38` — the settled disposition is serialized.

### EK-16 — Stateful actions verify by a second observation, and default to unverified

- **type**: Runtime · **layer**: L2 · **epistemic**: fact
- **claim**: `post_action.rs` performs a fresh read only for actions with `requires_state_readback()`. `Toggle` first reads checked state and rewrites to Check/Uncheck; `TypeText` reads before value + selection to compute expected text. Execution then settles up to 400 ms and compares a postcondition. A mismatch is `ACTION_FAILED` **except** physical text mismatch (`typed_by_keys`), which is downgraded to `delivered_unverified` with `verification_scope:unavailable` and `verification_reason:postcondition_mismatch`. Missing evidence (`verified.is_none()`) is also `delivered_unverified` with `insufficient_evidence` or `secure_field`.
- **mechanism**: Verification scopes are `element_state` for state assertions and `element_value` for writes, with `application_commit:"not_verified"` for writes. `ActionStep` summaries derive `delivered`/`verified`; no delivered step → `not_delivered`; delivered but not verified → `delivered_unverified`; verified → `delivered_verified`.
- **evidence**: EV-019 (`crates/core/src/actions/post_action.rs`), EV-020 (`crates/core/src/actions/action_result.rs`), EV-034 (`docs/json-output.md` post-2.5).
- **boundary**: An element-state match is not an application commit. A virtualized list or async save can accept the write then discard it; the package does not claim such behavior was measured.
- **links**:
  - `mechanism → EK-13` — both model "evidence may be inconclusive".
  - `contrast → EK-14` — verification is the only path from unverified to verified.
  - `dependency → EK-35` — post-action waits observe a fresh snapshot, not this readback.

### EK-17 — Post-action wait returns a snapshot and nests the action result under `after_action`

- **type**: Tooling · **layer**: L1 · **epistemic**: fact
- **claim**: A successful action with `--wait-for` returns top-level `data` as the snapshot, with the original action result under `data.after_action`. `docs/json-output.md` specifies `details.after_action` for errors but does not specify the success shape.
- **mechanism**: `helpers.rs` builds the combined body; `helpers_ref_action_wait_result_tests.rs` pins the shape. The wait is a real observation, not a fixed sleep; it uses the selector/wait predicates.
- **evidence**: EV-021 (`crates/core/src/commands/helpers.rs`), EV-021 (`helpers_ref_action_wait_result_tests.rs`), EV-038 (`docs/json-output.md`).
- **boundary**: Callers that assume `data` is always the action result will misread the success case. This is a documented-shape gap, tracked as CAND-02, not a code defect proven by tests.
- **links**:
  - `subsystem → EK-35` — the wait obeys batch baseline/ordering rules when inside batch.
  - `dependency → EK-18` — the returned data is a full/partial snapshot.
  - `causal → EK-15` — the action's disposition is computed before the wait wraps it.

### EK-18 — Snapshot completeness is tri-state and separate from identity proof

- **type**: Engineering · **layer**: L2 · **epistemic**: fact
- **claim**: `SnapshotResult` carries `complete`, `nodes_observed`, and `identity_unproven`. A depth-limited presentation can be `complete:true`; an over-budget read is `complete:false` + `truncated:true` + `nodes_observed`; a failed whole-window proof adds `identity_unproven:true` while retaining the shallow tree.
- **mechanism**: `format_snapshot_fields` adds `truncated` and `nodes_observed` only when `!complete`, and `identity_unproven:true` when the reason is present. `snapshot_ref` drills can still time out rather than return a partial tree.
- **evidence**: EV-022 (`crates/core/src/snapshot/mod.rs:SnapshotResult`), EV-023 (`crates/core/src/snapshot/window_capture.rs`), EV-024 (`crates/core/src/commands/snapshot.rs:format_snapshot_fields`).
- **boundary**: `complete` does not mean "whole desktop"; it means the requested presentation was fully observed within its depth/budget. `identity_unproven` is a property of tokenless refs, not of the tree.
- **links**:
  - `dependency → EK-19` — shallow captures may need a whole-window proof.
  - `causal → EK-20` — refs are qualified after the tree is built.
  - `subsystem → EK-23` — the result is persisted as a refmap.

### EK-19 — A shallow snapshot pays for a whole-window proof from its deadline

- **type**: Engineering · **layer**: L2 · **epistemic**: fact
- **claim**: If requested depth < `WHOLE_WINDOW_DEPTH (50)` and the adapter does not retain element tokens, `capture_window` first attempts a whole-window read with `deadline.remaining() * 3/4`. Success returns the whole observation and presents a cut at the requested depth; failure falls back to the shallow observation, marks `identity_unproven`, and returns the shallow tree.
- **mechanism**: `whole_window` uses `TreeOptions { max_depth: 50, skeleton: false, ..opts }` with ref-identity bounds. It accepts only an exhaustive observation. A non-exhaustive read is treated as "cut short". The remaining 1/4 deadline is reserved for the fallback shallow walk.
- **evidence**: EV-023 (`crates/core/src/snapshot/window_capture.rs:capture_window`), EV-022 (`WHOLE_WINDOW_DEPTH` in `refs/resolve_saved_ref.rs`).
- **boundary**: This is a latency/identity trade: shallow snapshots are not cheap on tokenless platforms. The 3/4 split is a heuristic, not a measured optimum in this run.
- **links**:
  - `mechanism → EK-21` — the whole-window observation feeds the uniqueness census.
  - `dependency → EK-18` — failure writes `identity_unproven`.
  - `constraint → EK-22` — only predicate fields are counted, so the proof does not depend on value.

### EK-20 — Refs must be snapshot-qualified, and bare refs are rejected

- **type**: Engineering · **layer**: L1 · **epistemic**: fact
- **claim**: `qualify_ref_id` produces `@<snapshot_id>:e<N>`. `resolve_ref_target` requires `@`, `rsplit_once(":e")`, a valid snapshot id, and a local ref of `@e` + 1..10 digits + >0. A bare `@e1` is `INVALID_ARGS`.
- **mechanism**: Snapshot ids are base36, `s`-prefixed, at least 4 chars, 3–64 chars total, ASCII alnum/`-`/`_`. `qualify_tree_refs` recursively rewrites the presented tree so every ref carries its snapshot. `validate_snapshot_id` rejects traversal-shaped ids.
- **evidence**: EV-025 (`crates/core/src/refs/ref_token.rs`), EV-024 (`crates/core/src/commands/snapshot.rs`).
- **boundary**: Qualification prevents cross-snapshot aliasing but does not prevent a caller from using a valid ref against a changed UI; that is resolved by the identity predicate and process checks.
- **links**:
  - `dependency → EK-22` — qualification supplies the key the predicate resolves against.
  - `constraint → EK-21` — a tokenless ref must carry both snapshot and local index.
  - `subsystem → EK-23` — refmaps are stored per snapshot id.

### EK-21 — Tokenless refs are published only after a whole-window uniqueness census

- **type**: Engineering · **layer**: L2 · **epistemic**: fact
- **claim**: `CaptureProof::proves_node` requires a complete whole-window observation, exactly one node matching the entry's full predicate, no unread recorded field, and that node to be the entry's own node. Otherwise the entry is allocated but `unique_at_capture` is false and resolution refuses it.
- **mechanism**: `count_nodes` buckets nodes by field subset (`NATIVE_ID | NAME | DESCRIPTION`), stores the last index for predicates with count 1, and compares to the entry's node. `unread` entries are checked for any overlapping recorded field that would make `compare` return `Unknown`.
- **evidence**: EV-026 (`crates/core/src/refs/ref_capture_proof.rs`), EV-023 (`window_capture`), EV-025 (`ref_token`).
- **boundary**: Uniqueness is at capture time, not at use time. A later duplicate element can still make an old ref ambiguous; resolution returns `saved_ref_ambiguous` or `identity_not_unique`.
- **links**:
  - `mechanism → EK-19` — the census runs over the whole-window observation.
  - `dependency → EK-22` — uniqueness is predicate-relative.
  - `causal → EK-18` — a cut-short whole-window read prevents publication.

### EK-22 — Ref identity is a closed predicate over immutable fields; value and bounds are excluded

- **type**: Engineering · **layer**: L2 · **epistemic**: fact
- **claim**: `FullPredicate` compares `role`, optional `native_id`, optional stable name, optional description. Mutable value and bounds are never identity. A recorded field that the live read could not answer yields `Unknown`, never `Match`.
- **mechanism**: `stable_name(role, name, value)` can promote a value into the name field for roles where the value is the label; this is a role-specific rule, not general value identity. `native_id_match` returns `NoMatch` only when the live native-id set is complete; otherwise `Unknown`. `conjunction` propagates `NoMatch` before `Unknown`.
- **evidence**: EV-027 (`crates/core/src/refs/ref_full_predicate.rs`), EV-028 (`crates/core/src/refs/ref_entry_identity.rs`), EV-034 (`RefEntryIdentity`).
- **boundary**: The predicate is deliberately incomplete: two elements with identical role/name/native-id cannot be distinguished. That is why uniqueness at capture and ambiguity at resolution are first-class outcomes.
- **links**:
  - `constraint → EK-21` — the census counts exactly this predicate.
  - `contrast → EK-28` — secure fields remove even more identity content.
  - `dependency → EK-19` — whole-window proof supplies the complete field reads.

### EK-23 — RefStore is bounded, locked, and transactional

- **type**: Memory · **layer**: L2 · **epistemic**: fact
- **claim**: Refmaps are capped at 1 MiB; snapshots at 128 with prune to 96; stale temps at 60 s. Save/update are lock-protected; update takes expected ref/entry to prevent lost updates; latest snapshot id is a separate file.
- **mechanism**: Paths `<base>/snapshots/<id>/refmap.json`, `<base>/refstore.lock`, `<base>/latest_snapshot_id`. `save_new_snapshot` saves snapshot + latest + prune under lock. `update_existing_snapshot` does load-update-save under lock with stale guards. `retained_object_tokens` refuses pruning if a live token would be removed.
- **evidence**: EV-029 (`crates/core/src/refs/mod.rs:RefStore`), EV-029 (`refs_store.rs`, `refs_store_prune.rs`).
- **boundary**: The 1 MiB cap can make a very large tree unsavable; the error suggests narrowing. Retention is count-based, not time-based.
- **links**:
  - `subsystem → EK-24` — sessions provide the base directory.
  - `subsystem → EK-25` — both use file locks for cross-process safety.
  - `constraint → EK-26` — trace refmap copies are bounded by the same store.

### EK-24 — Session state is namespaced and explicitly selected

- **type**: State · **layer**: L1 · **epistemic**: fact
- **claim**: `~/.agent-desktop` is the default state root; `AGENT_DESKTOP_HOME` overrides it. Sessions live in `<root>/sessions/<id>/` with a 64 KiB `session.json`. Resolution order is explicit `--session` > `AGENT_DESKTOP_SESSION` > none.
- **mechanism**: `session_dir` validates the id; `read_manifest` caps size; `trace_enabled` and `CommandContext` share the manifest decision; `TRACE_LIVENESS_WINDOW = 300s`. Cursor overlay requires a live, un-ended manifest with overlay enabled and a 1–64 char agent id.
- **evidence**: EV-030 (`crates/core/src/session/mod.rs`), EV-008 (`src/mcp/mod.rs` session context).
- **boundary**: "No session" is a valid state; some features (trace, cursor overlay, session-scoped refstore) degrade or refuse without one. Session end is cooperative, not a hard process kill.
- **links**:
  - `dependency → EK-23` — refstore uses the session directory.
  - `subsystem → EK-25` — lease lifetime is independent of session lifetime.
  - `constraint → EK-26` — trace path is session-scoped.

### EK-25 — Desktop mutation is serialized by a machine-wide interaction lease

- **type**: Permission · **layer**: L2 · **epistemic**: fact
- **claim**: On Unix, `InteractionLease` combines a process lease and a file lock at `/tmp/agent-desktop-<uid>/interaction.lock`; the directory must be user-owned, non-symlink, mode 0700. The lease is per user, not per session, and can be inherited via an FD.
- **mechanism**: `FileLock` + `ProcessLeaseGuard`; contention counted and reported. `guarded` allows platform-specific guards. Windows/macOS adapters supply their own guard via `acquire_interaction_lease`. Clipboard-set, close-app, focus-window, drag, hover, notification mutation acquire it; clipboard-get does not.
- **evidence**: EV-031 (`crates/core/src/session/interaction_lease.rs`), EV-039 (`clipboard commands`), EV-012 (`src/command_policy`).
- **boundary**: The lock serializes participating commands, not arbitrary external apps or a user's own input. It is not a security boundary against same-UID processes.
- **links**:
  - `mechanism → EK-09` — both are authority gates for mutation, one identity, one mutual exclusion.
  - `subsystem → EK-23` — both rely on file locks.
  - `constraint → EK-29` — held-input release runs under the same mutation assumptions.

### EK-26 — Trace is an append-only, size-bounded, key-redacted event log

- **type**: Memory · **layer**: L2 · **epistemic**: fact
- **claim**: Trace files cap at 64 MiB; events at 1 MiB. First event is `trace.meta`; writes take an exclusive lock per event. `trace_sanitize` recursively redacts a fixed key list and whole keys `actions`/`key`.
- **mechanism**: Lazy open; `TracePending::None` writes nothing; strict mode returns errors on open/emit failure, non-strict warns. `ensure_meta_if_needed` writes schema/version/os/pid/process start. Null stays null; redacted values become `{"redacted":true}`.
- **evidence**: EV-032 (`crates/core/src/trace/mod.rs`), EV-033 (`crates/core/src/trace/trace_sanitize.rs`).
- **boundary**: Key-based redaction is not content classification. A secret under an unlisted key survives; a benign value under `name` is removed. This is a schema-level privacy policy, not a DLP system.
- **links**:
  - `mechanism → EK-27` — both are trace channels with different privacy rules.
  - `constraint → EK-28` — secure values must already be withheld before trace.
  - `dependency → EK-24` — trace lives under the session.

### EK-27 — Replay artifacts are deliberately unredacted

- **type**: Memory · **layer**: L1 · **epistemic**: fact
- **claim**: `CONCEPTS.md:179` states screenshots and refmap artifacts are unredacted and may appear in exported HTML. Artifacts are opt-in: `session start --screenshots` + `artifacts: full`. Single embedded screenshot cap is 128 MiB; refmap copies are written only if absent.
- **mechanism**: `artifacts_enabled = trace_enabled && artifacts_full`; screenshot capture requires an exact target (window id + process identity); refmap copy path is `trace/refmaps/<snapshot_id>.json`.
- **evidence**: EV-034 (`CONCEPTS.md:179`), EV-036 (`crates/core/src/trace/trace_artifacts.rs`).
- **boundary**: This is an operator-owned risk, not a bug. The package records it as a privacy boundary, not as a defect. An export pipeline that treats trace JSONL as safe but artifacts as safe too would be wrong.
- **links**:
  - `contrast → EK-26` — event trace is redacted, artifact replay is not.
  - `constraint → EK-28` — secure-field screenshots can still leak pixels; redaction does not apply.
  - `subsystem → EK-23` — refmap artifacts duplicate the refstore.

### EK-28 — Secure fields withhold value and related text on read and write

- **type**: Security · **layer**: L2 · **epistemic**: fact
- **claim**: macOS drops `value` when role/subrole is `AXSecureTextField`; Linux withholds value for password role or unreadable role/states and for truncated values; Windows treats `IsPassword` (including unreadable/unknown) as secure, withholds `Name` of a labelled target, and skips `get_value`/`RangeValue` readback. Write errors carry `value_chars`, not text; Linux scrubs the written value from message/suggestion/platform_detail/details.
- **mechanism**: Windows `secure_from_read` is `read.unwrap_or(true)` (fail-closed). `gated_compare` returns `Ok(None)` on a secure read. macOS `node_read` sets `value = None`; post-state also withholds. Linux `guarded_read` returns `None`; `scrub` redacts echoes.
- **evidence**: EV-035 (`crates/macos/src/tree/query/node_read.rs`), EV-035 (`crates/macos/src/actions/post_state.rs`), EV-035 (`crates/windows/src/tree/name_evidence.rs`), EV-035 (`crates/windows/src/actions/value_write.rs`), EV-035 (`crates/linux/src/actions/write/secure_write.rs`), EV-035 (`crates/linux/src/tree/capabilities/action_read.rs`).
- **boundary**: Secure-field identity still uses role/native-id/label; it does not use content. `CONCEPTS.md` calls this a "content-free fingerprint"; no literal digest function was found in the ref path (CAND-03). The behavior is withholding, not hashing.
- **links**:
  - `mechanism → EK-22` — both remove mutable content from identity.
  - `constraint → EK-26` — trace redaction is a second line, not the primary control.
  - `subsystem → EK-27` — artifacts bypass this withholding.

### EK-29 — Held input is armed before commit and released on every exit path

- **type**: Runtime · **layer**: L2 · **epistemic**: fact
- **claim**: Linux `ReleaseGuard` records held keys/buttons and keymap remaps, syncs a journal before sending down events, round-trips on up, and releases everything on finish/abort/Drop. macOS `KeyReleaseGuard` holds the up `CGEvent` and posts it on Drop. Windows `KeyReleaseGuard` releases held virtual keys on Drop only if still down.
- **mechanism**: Linux release order: return to origin, release buttons in reverse, release keys in reverse, restore keymap in reverse, round-trip. Held/remaps are cleared only if all succeed. `arm()` before `down` is the commit point. Windows `key_input(vk, true)` sets `key_up`.
- **evidence**: EV-037 (`crates/linux/src/input/release_guard.rs`), EV-037 (`crates/windows/src/input/...` `KeyReleaseGuard`), EV-037 (`crates/macos/src/input/...` `KeyReleaseGuard`).
- **boundary**: Drop guards cover Rust unwinding and normal exits, not SIGKILL or power loss. Linux separately installs a signal handler to release input.
- **links**:
  - `mechanism → EK-25` — both protect mutation from interleaving/abandonment.
  - `dependency → EK-34` — Windows/macOS/Linux implement different release mechanisms.
  - `contrast → EK-16` — release is a local invariant; postcondition is an external observation.

### EK-30 — Clipboard is the explicit exception to accessibility authority

- **type**: Permission · **layer**: L1 · **epistemic**: fact
- **claim**: `clipboard-get`, `clipboard-set`, `clipboard-clear` map to `None` in `policy_for` and do not request a permission report. `clipboard-set` acquires an interaction lease; `clipboard-get` does not. `get_clipboard_content` takes no `InteractionLease` parameter.
- **mechanism**: `clipboard-get` is classified read-only unless `--out` is set; `clipboard-set` is always mutating but `policy_for == None`, so the multi-agent `--agent-id` gate does not apply. Clipboard images default to a private session/temp path; `--out` writes to a user path.
- **evidence**: EV-039 (`crates/core/src/commands/clipboard_get.rs`, `clipboard_set.rs`), EV-040 (`src/command_policy/mod.rs:policy_for`), EV-045 (`src/cli/mod.rs:is_mutating`).
- **boundary**: Whether this is intentional (clipboard is not a UI-semantic operation) or an authority gap is unresolved; CAND-01. The package records the asymmetry, not a verdict.
- **links**:
  - `contrast → EK-08` — the only command family with no accessibility need.
  - `constraint → EK-09` — no agent-id gate follows from no accessibility need.
  - `dependency → EK-25` — set takes the lease, get does not.

### EK-31 — Skills are compiled into the binary

- **type**: Tooling · **layer**: L1 · **epistemic**: fact
- **claim**: `crates/core/src/commands/skills.rs` embeds `skills/agent-desktop/SKILL.md`, its four references, platform skills, `jev-desktop`, and FFI docs via `include_str!`. `skills` can list/get them at runtime; `CURRENT_PLATFORM` filters platform-specific content.
- **mechanism**: Each skill has a canonical name, aliases (including `platform`), summary, optional platform tag, main body, and references. Because content is `include_str!`, it is version-locked to the binary.
- **evidence**: EV-041 (`crates/core/src/commands/skills.rs`), EV-041 (`skills/` tree).
- **boundary**: A binary built from an older commit serves older instructions; there is no remote update channel. Skills are documentation, not executable policy.
- **links**:
  - `subsystem → EK-02` — delivery is tied to the compiled platform.
  - `mechanism → EK-06` — both eliminate an external doc/registry drift.
  - `dependency → EK-32` — docs and code counts are drift-tested.

### EK-32 — Source shape and architecture are enforced by scripts, not convention

- **type**: Workflow · **layer**: L2 · **epistemic**: fact
- **claim**: `check-architecture.sh` self-tests with a violating fixture, then checks core→platform dependency, platform↔platform dependency, allowed OS `cfg` sites in core, crate layout, capability-layer location, core domains not reaching `commands`, and declared test files. `check-rust-file-size.sh` caps at 400 lines; `check_rust_comments.py` bans inline comments; `check-no-phase-references.sh` bans phase references.
- **mechanism**: CI runs these plus fmt, MSRV, clippy `-D warnings`, tests, 15 MiB binary cap, FFI header, E2E harness checks. A violation is a CI failure, not a review comment.
- **evidence**: EV-042 (`scripts/check-architecture.sh`, `scripts/check-rust-file-size.sh`, `.github/workflows/ci.yml`), EV-019 (`AGENTS.md`).
- **boundary**: Scripts enforce shape, not semantic correctness. The 400-line cap can fragment a coherent module into many files; the one-domain-per-file rule is a response to that.
- **links**:
  - `constraint → EK-02` — enforces the compile-time isolation design.
  - `constraint → EK-03` — enforces command/module placement.
  - `dependency → EK-33` — CI is the execution substrate for all gates.

### EK-33 — E2E trusts independent effect observation, not command success

- **type**: Testing · **layer**: L2 · **epistemic**: fact
- **claim**: E2E scenarios assert observable effects separately; `tests/e2e/scenarios/reliability.sh:92` asserts `data.after_action.action`. The repository's dogfood rule requires a real app, a released binary, and a disposition for every finding (fixed / owned elsewhere / accepted); no finding is treated as failure.
- **mechanism**: Golden JSON contracts pin envelope shape; `probes/` contains re-runnable OS measurements and `FINDINGS.md` ledgers (`A15-*`, `A19-*`); `pre-commit` uses an isolated HOME to avoid polluting `~/.agent-desktop`.
- **evidence**: EV-043 (`tests/e2e/scenarios/reliability.sh:92`), EV-043 (`AGENTS.md` dogfood rules), EV-043 (`probes/*/FINDINGS.md`), EV-042 (`pre-commit`).
- **boundary**: The package did not run E2E. These are read facts about the test system, not evidence that the suite passes at this commit.
- **links**:
  - `mechanism → EK-16` — both require a second observation.
  - `dependency → EK-34` — platform behavior must be measured per OS.
  - `subsystem → EK-32` — both are governance layers.

### EK-34 — Platform capability is a named matrix, not an implicit assumption

- **type**: Architecture · **layer**: L2 · **epistemic**: fact
- **claim**: macOS uses AX (13+), Windows uses UIA (10 1809+), Linux uses AT-SPI (Ubuntu 24.04+, glibc ≥ 2.35). On Linux, observation and semantic refs work on X11 and Wayland; pointer gestures, wheel scrolling, window control, `focus-window`, and headed shell input require X11; Wayland headed pointer returns `ACTION_NOT_SUPPORTED`.
- **mechanism**: The Linux skill documents the split; `build_adapter` selects the platform crate; individual adapter methods return `not_supported` where the backend cannot provide a capability. `glibc` and OS floors are stated in install docs.
- **evidence**: EV-034 (`skills/agent-desktop-linux/SKILL.md`, `docs/install.md`), EV-002 (`src/main.rs`), EV-001 (`adapter/mod.rs` default not_supported).
- **boundary**: The matrix is documented and partly test-covered (`linux-e2e.yml`, `windows-capability-probe.yml`), but this run did not execute those workflows. OS-version claims are documentation facts.
- **links**:
  - `dependency → EK-02` — the selected adapter defines the available row.
  - `constraint → EK-11` — headed downgrade must respect the row.
  - `mechanism → EK-29` — release behavior is also platform-specific.

### EK-35 — A wait-event baseline is captured before the action it observes

- **type**: Runtime · **layer**: L1 · **epistemic**: fact
- **claim**: In batch, if the next entry is `wait --event`, `capture_signal_baseline` runs before the current command. If baseline capture fails, the current command is **not started**, the entry is marked `pre_action_baseline_failed` / `not_delivered`, and the batch stops with `blocked_index`.
- **mechanism**: `pending_baseline` is carried in `item_context.with_event_baseline(current_baseline)`. The baseline filter uses the wait's app (process None). This prevents a wait from observing an event that occurred before its baseline.
- **evidence**: EV-006 (`src/batch/execution.rs:40-95`, `:187-212`), EV-005 (`src/batch/mod.rs`).
- **boundary**: This ordering exists only inside batch. A standalone action followed by a separate `wait --event` process cannot retroactively establish a baseline.
- **links**:
  - `causal → EK-04` — batch ordering creates the opportunity for pre-capture.
  - `constraint → EK-17` — post-action wait uses a different, selector-based mechanism.
  - `subsystem → EK-16` — both are observation ordering problems.

### EK-36 — Screenshot permission is scope-dependent

- **type**: Permission · **layer**: L1 · **epistemic**: fact
- **claim**: `Screenshot` with `--app` or `--window-id` requires Accessibility **and** Screen Recording; without scope it requires only Screen Recording.
- **mechanism**: `policy_for` matches the `ScreenshotArgs.scope` fields. `preflight` rejects if either required permission is denied. The rationale is that scoped capture needs accessibility to resolve the target, while full-screen capture does not.
- **evidence**: EV-011 (`src/command_policy/mod.rs:policy_for`), EV-012 (`preflight`).
- **boundary**: This is a static requirement, not a runtime check of what the capture actually reads. A scoped screenshot that falls back to a larger region would not be caught here.
- **links**:
  - `dependency → EK-08` — it is one row of the permission map.
  - `contrast → EK-30` — clipboard has no permission row at all.
  - `constraint → EK-27` — captured artifacts may be unredacted.

### EK-37 — Action capability is per-action and per-platform, with semantic-first delivery

- **type**: Agent · **layer**: L2 · **epistemic**: fact
- **claim**: `ActionabilityRequirements::for_action` defines which gates apply. `pointer_delivery` chooses `Physical` only for headed + cursor-requiring actions; otherwise it uses `Semantic` if the target advertises a direct semantic pointer capability, else `Physical`. `delivered_without_pointer` is true only for headless semantic/own-capability delivery.
- **mechanism**: `requires_geometry` skips geometry for keyboard-delivered actions and for headless direct semantic pointer/text. `supports_direct_semantic_pointer_delivery` consults the snapshot's `available_actions`. `Action::headed_requirement` maps pointer actions to `FocusedWindowAndCursor`, state/text to `FocusedWindow`.
- **evidence**: EV-034 (`crates/core/src/actionability/requirements.rs`), EV-014 (`crates/core/src/actions/action.rs` `Action::base_interaction_policy`).
- **boundary**: The capability table is data from the snapshot, not a proof that the advertised action will work. A stale or lying provider can still fail.
- **links**:
  - `mechanism → EK-10` — policy picks the mode.
  - `dependency → EK-12` — requirements feed the battery.
  - `contrast → EK-11` — headed downgrade is a platform-specific exception.

### EK-38 — The JSON envelope is versioned and fixed-shape

- **type**: Runtime · **layer**: L1 · **epistemic**: fact
- **claim**: `Response` is `{ version: "2.6", ok, command, data?, error? }`. `ErrorPayload` always includes `code`, `message`, `disposition`; optional `suggestion`, `recovery`, `platform_detail`, `details`. Success omits `error`; failure omits `data`.
- **mechanism**: `Response::ok` / `Response::err` are the only constructors. `docs/json-output.md`, `AGENTS.md`, and the bug-report template are drift-tested against `ENVELOPE_VERSION`. Exit codes are 0/1/2.
- **evidence**: EV-038 (`crates/core/src/response/output.rs`), EV-038 (`src/cli/doc_literal_drift_tests.rs`), EV-038 (`docs/json-output.md`).
- **boundary**: The envelope is stable, but nested `data` shapes are per-command and can differ between success and error paths (CAND-02).
- **links**:
  - `subsystem → EK-14` — disposition is embedded in the envelope.
  - `constraint → EK-15` — the disposition written is the settled one.
  - `mechanism → EK-17` — post-action wait nests inside `data`.

### EK-39 — FFI is a panic-contained, thread-affine resource ABI

- **type**: Architecture · **layer**: L2 · **epistemic**: fact
- **claim**: The cdylib exposes `PlatformAdapter` over C ABI with opaque ids, a last-error model, resource registries, a session lease, and a `trap_panic` boundary. It requires `panic=unwind` and rejects `panic=abort` at compile time. Native element handles are thread-affine and must be released on the resolving thread; mutations are serialized by the interaction lease.
- **mechanism**: `AD_ABI_VERSION_MAJOR` in the header is drift-tested. `release-ffi` profile is separate from the CLI `release` profile, which keeps `panic=abort` for binary size. `ad_last_error_message` survives successful calls and rotates only on the next failure, POSIX-errno style.
- **evidence**: EV-044 (`crates/ffi/src/lib.rs`), EV-044 (`crates/ffi/include/agent_desktop.h`), EV-046 (`README.md` FFI/ABI docs).
- **boundary**: Thread affinity is documented, not enforced by the type system across the C boundary; a host that releases on another thread is violating the contract.
- **links**:
  - `subsystem → EK-01` — FFI wraps the same adapter contract.
  - `constraint → EK-25` — mutations rely on the lease for serialization.
  - `contrast → EK-07` — FFI is in-process, MCP is per-call child process.

---

## Graph integrity (declared)

- All 39 EK have ≥1 link.
- All 39 EK appear in ≥1 KO in `03_knowledge-layer/`.
- No EK is deleted when a KO forms; KO derives from EK, not replaces it.
- Edge counts and cluster sizes are reported in `run_metadata.yaml` after the audit.

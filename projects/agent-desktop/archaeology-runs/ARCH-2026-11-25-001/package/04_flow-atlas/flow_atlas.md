# 04 — Flow Atlas

Seven flows are traced from code anchors, not from architecture diagrams. Each edge names a symbol/file and, where the transition is conditional, the condition that selects it. Edge IDs are stable for the audit.

Legend: **from → to**; `cond` is the guard or state change that makes the edge fire.

---

## 1. Control Flow (CF)

| Edge | From → To | Anchor | Condition / transition |
|---|---|---|---|
| CF-01 | `main` → `Cli::parse` → `dispatch::dispatch` | `src/main.rs:main`; `src/cli/root.rs`; `src/dispatch/mod.rs:dispatch` | Always after process start; parse errors exit 2 before dispatch |
| CF-02 | `dispatch` match arm → core `commands::*::execute` | `src/dispatch/mod.rs:dispatch_command` | Exhaustive `match` on `Commands`; each arm selects its command module |
| CF-03 | core command → `PlatformAdapter` sub-trait method → platform impl | `crates/core/src/commands/*`; `crates/core/src/adapter/mod.rs`; `crates/{macos,windows,linux}/src/adapter.rs` | Command calls the trait; the compiled adapter resolves to one platform impl |
| CF-04 | batch `prepare` → per-entry `dispatch` loop | `src/batch/preparation.rs:prepare`; `src/batch/execution.rs` | Only after all entries parse and preflight; loop stops on deadline, `stop_on_error`, output limit, baseline failure, or ended session |
| CF-05 | MCP `serve` → `call` → child `main` | `src/mcp/mod.rs:serve`; `src/mcp/call.rs`; `std::env::current_exe()` | Per JSON-RPC `tools/call`; child stdin null; globals prepended except `sandbox exec` |
| CF-06 | ref action → actionability check → adapter `perform_action` | `crates/core/src/ref_action/*`; `crates/core/src/actionability/evaluate.rs`; `ActionOps::perform_action` | Only if the actionability report is actionable; otherwise `finish` returns `not_delivered` |
| CF-07 | saved ref → `resolve_saved_ref` → action | `crates/core/src/refs/resolve_saved_ref.rs`; `crates/core/src/ref_action/*` | Requires qualified `@snapshot:eN`; tokenless refs must be `unique_at_capture`; whole-window path uses `WHOLE_WINDOW_DEPTH=50` |

## 2. State Flow (SF)

| Edge | From → To | Anchor | Condition / transition |
|---|---|---|---|
| SF-01 | `session start` → `session.json` manifest → `CommandContext` | `crates/core/src/session/mod.rs`; `crates/core/src/context/*` | Creates `<root>/sessions/<id>/`; manifest read cap 64 KiB; trace defaults on |
| SF-02 | snapshot build → `SnapshotResult` → `RefStore::save_new_snapshot` | `crates/core/src/snapshot/mod.rs:build`; `SnapshotResult::save_refmap` | `complete` uses normal lock; incomplete uses detached lock; save includes latest + prune |
| SF-03 | ref action → `post_state` read → `ActionStep` / `ActionStep` summary | `crates/core/src/actions/post_action.rs`; `ActionStep` | Only for `requires_state_readback()` actions; 50 ms settle, max 400 ms; mismatch → `ACTION_FAILED` |
| SF-04 | batch entry → `for_batch_item(session)` → item context | `src/batch/preparation.rs`; `CommandContext::for_batch_item` | Per-entry `session` overrides parent; ended session is rejected before execution |
| SF-05 | interaction policy → `headed` / `focus_fallback` / `headless` | `crates/core/src/actions/interaction_policy.rs`; `Action::base_interaction_policy` | Default headless; `PressKey` defaults focus_fallback; `--headed` or join changes mode |
| SF-06 | refstore save → prune low-water 96 | `crates/core/src/refs/refs_store_prune.rs` | Triggered when stored snapshots exceed 128; never prunes a live retained token |

## 3. Data Flow (DF)

| Edge | From → To | Anchor | Condition / transition |
|---|---|---|---|
| DF-01 | AX/UIA/AT-SPI read → `ObservedTree` → `AccessibilityNode` | platform `tree/` modules; `crates/core/src/locator/live_locator`; `crates/core/src/tree` | Platform-specific read; `project_window` cuts at logical depth; `project_partial` marks truncation |
| DF-02 | observed node evidence → `RefEntry` | `crates/core/src/refs/ref_alloc.rs`; `RefEntry` | Allocation walks the projected tree; predicate fields copied; bounds/geometry separate |
| DF-03 | `RefEntry` → `FullPredicate` → `IdentityMatch` | `crates/core/src/refs/ref_full_predicate.rs` | At capture proof and at resolution; `Match`/`NoMatch`/`Unknown` conjunction |
| DF-04 | command result → `Response` envelope | `crates/core/src/response/output.rs`; `src/main.rs:emit_response` | One JSON line; success omits error, failure omits data |
| DF-05 | batch entries → results array | `src/batch/execution.rs`; `src/batch/result_entry.rs` | Each entry is completed or not-started; byte budget can truncate the array |
| DF-06 | trace event → JSONL segment | `crates/core/src/trace/mod.rs` | Lazy open; `trace.meta` first; per-event lock; 1 MiB event cap, 64 MiB file cap |

## 4. Evidence Flow (EF)

| Edge | From → To | Anchor | Condition / transition |
|---|---|---|---|
| EF-01 | whole-window observation → `CaptureProof` → `unique_at_capture` | `crates/core/src/snapshot/window_capture.rs`; `crates/core/src/refs/ref_capture_proof.rs` | Complete whole-window read; exactly one predicate match; no unread recorded field |
| EF-02 | action execution → `DeliveryOutcome` → disposition | `crates/core/src/actions/action_result.rs`; `crates/core/src/actions/delivery_semantics.rs` | No delivered step → not_delivered; delivered without verification → delivered_unverified; verified → delivered_verified |
| EF-03 | postcondition read → `verification_scope` | `crates/core/src/actions/post_action.rs` | State actions → `element_state`; writes → `element_value` + `application_commit:not_verified` |
| EF-04 | five-point hit test → `receives_events` | `crates/core/src/actionability/receives_events.rs` | Any `ReachesTarget` passes; concrete occluder fails; unknown/no support is non-blocking only for this check |
| EF-05 | E2E scenario → independent effect assertion | `tests/e2e/scenarios/reliability.sh:92`; `tests/e2e/README.md` | Asserts `data.after_action.action`; does not trust command `ok:true` |
| EF-06 | probe run → `FINDINGS.md` ledger → code comment citation | `probes/{macos,windows,linux}/FINDINGS.md`; code comments `A15-*`, `A19-*` | Measurement is committed with its input; code cites the probe ID |

## 5. Authority Flow (AF)

| Edge | From → To | Anchor | Condition / transition |
|---|---|---|---|
| AF-01 | command → `policy_for` → `preflight` | `src/command_policy/mod.rs:policy_for`; `preflight` | Denied need → `PERM_DENIED` + `not_delivered`; `None` skips |
| AF-02 | `multi_agent && !agent_id && is_mutating && Accessibility` → rejection | `src/command_policy/mod.rs:agent_id_missing`; `preflight_context` | Only this conjunction triggers `INVALID_ARGS`; batch entries checked individually |
| AF-03 | session → manifest → trace/overlay permission | `crates/core/src/session/mod.rs`; `CommandContext` | Live manifest, not ended, overlay enabled, agent id 1–64 chars for cursor overlay |
| AF-04 | mutating command → interaction lease → adapter mutation | `crates/core/src/session/interaction_lease.rs`; `crates/core/src/commands/helpers.rs` | Unix file+process lock; Windows/macOS guarded lease; clipboard-set/close/focus/drag/hover/notifications take it |
| AF-05 | FFI call → resource registry → thread-affine handle | `crates/ffi/src/lib.rs`; `crates/ffi/src/resource.rs`; `crates/ffi/src/session_lease.rs` | Handle used/released on resolving thread; panic trapped by `trap_panic`; requires panic=unwind |
| AF-06 | MCP server → `Calls::admit` → child process | `src/mcp/mod.rs`; `src/mcp/call.rs` | Max 8 concurrent; same id rejected; cancellation kills child |

## 6. Memory Flow (MF)

| Edge | From → To | Anchor | Condition / transition |
|---|---|---|---|
| MF-01 | `RefMap` → `snapshots/<id>/refmap.json` | `crates/core/src/refs/refs_store.rs`; `RefMap::serialize_with_size_check` | 1 MiB cap; >cap → internal error with narrow suggestion |
| MF-02 | `snapshot_id` → `latest_snapshot_id` | `crates/core/src/refs/refs_store.rs` | Written under the same lock as the snapshot |
| MF-03 | trace event → `trace/trace.jsonl` | `crates/core/src/trace/mod.rs` | `TracePending::None` writes nothing; strict open/emit failures are errors |
| MF-04 | snapshot refmap → `trace/refmaps/<id>.json` | `crates/core/src/trace/trace_artifacts.rs` | Only if full artifacts enabled; writes only if absent |
| MF-05 | session → `session.json` | `crates/core/src/session/mod.rs` | 64 KiB read cap; ended_at marks terminal state |
| MF-06 | compile-time `include_str!` → `skills` output | `crates/core/src/commands/skills.rs` | Skills are baked into the binary; platform filter uses `CURRENT_PLATFORM` |

## 7. Policy Flow (PF)

| Edge | From → To | Anchor | Condition / transition |
|---|---|---|---|
| PF-01 | headless default → semantic/pointer delivery decision | `Action::base_interaction_policy`; `ActionabilityRequirements::pointer_delivery` | Physical only if headed + cursor-requiring; otherwise semantic if advertised |
| PF-02 | headed request → `adapter.headed_requirement` | `crates/core/src/actionability/evaluate.rs` | Only headed requests query it; `FocusedApplication` rewrites evaluation to headless |
| PF-03 | secure field → withhold value/text | macOS `node_read`; Windows `value_write`; Linux `action_read` | Role/token/IsPassword marks secure; unreadable flag fails closed |
| PF-04 | clipboard command → no permission need | `src/command_policy/mod.rs:policy_for` | `ClipboardGet/Set/Clear` → `None`; no report requested |
| PF-05 | batch → fixed non-atomic semantics block | `src/batch/execution.rs` | Always emitted; `batch_retries:false`; per-command retry preserved |
| PF-06 | clap leaf → MCP tool schema | `src/mcp/catalog.rs` | Hidden `--stdin`/globals removed; `additionalProperties:false` |
| PF-07 | screenshot scope → permission need | `src/command_policy/mod.rs:policy_for` | App/window scope → Accessibility + Screen Recording; no scope → Screen Recording |

---

## Flow coverage check

| Flow | Edges | Covers |
|---|---:|---|
| Control | 7 | CLI, batch, MCP, ref action, adapter call |
| State | 6 | session, snapshot, postcondition, batch context, policy, pruning |
| Data | 6 | platform tree, ref allocation, predicate, envelope, batch output, trace |
| Evidence | 6 | capture proof, action disposition, verification, hit test, E2E, probes |
| Authority | 6 | permission, agent id, session, lease, FFI, MCP admission |
| Memory | 6 | refmap, latest, trace, artifacts, manifest, embedded skills |
| Policy | 7 | headless/headed, secure, clipboard, batch, MCP schema, screenshot |
| **Total** | **44** | |

## Flow-level observations

- **FLOW-OBS-01**: The control flow has three entry surfaces (CLI, batch, MCP) but one dispatch point. The branching is at the transport, not at the command.
- **FLOW-OBS-02**: The evidence flow is not a single path. Capture proof, action disposition, postcondition verification, hit-test proof, E2E, and probe ledgers are six independent evidence producers. A failure in one does not automatically invalidate another.
- **FLOW-OBS-03**: The authority flow has a deliberate gap: clipboard commands do not enter the permission path. Whether that is a policy exception or an oversight is CAND-01.
- **FLOW-OBS-04**: The memory flow has two privacy classes: event JSONL is key-redacted; replay artifacts are not. They share a session but not a redaction policy (CAND-09).
- **FLOW-OBS-05**: The policy flow is mostly declarative (`policy_for`, `requirements`, `base_interaction_policy`), with one runtime rewrite (`headed_requirement`). That rewrite is the only place a caller's policy is changed by platform feedback.

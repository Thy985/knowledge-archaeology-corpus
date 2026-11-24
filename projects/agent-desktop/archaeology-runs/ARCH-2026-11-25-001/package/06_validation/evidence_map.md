# 06 — Evidence Map

Every Fact, Pattern, and Model in this package must resolve to an EV row. Rows are repository-relative to the snapshot clone at `ce2ddf0`. `class` distinguishes code, doc, test, and governance artifacts. A row can cover a small bundle of files only when they express one contract.

| EV | Anchor | Class | What it proves | Limitation |
|---|---|---|---|---|
| EV-001 | `crates/core/src/adapter/mod.rs:PlatformAdapter` + four sub-traits | code | Core composes observation/action/input/system traits; blanket impl | Structural only; no behavioral proof |
| EV-002 | `src/main.rs:build_adapter` | code | Three `#[cfg(target_os)]` arms + `compile_error!`; Linux signal release before adapter | Compile-time selection only |
| EV-003 | `src/main.rs:main` pipeline | code | Parse → permission report → dispatch → envelope → exit code | No runtime execution in this run |
| EV-004 | `src/dispatch/mod.rs:dispatch`, `dispatch_command` | code | Direct `match` over `Commands`; read-only computed before command; settlement applied after | Exhaustiveness is compile-time, not semantic |
| EV-005 | `src/batch/mod.rs:parse_command` | code | Batch deserializes into same `Commands`; rejects nested batch/cursor-overlay/sandbox | Does not prove every variant is accepted |
| EV-006 | `src/batch/execution.rs` | code | Sequential non-atomic semantics; wait-event baseline before prior action; output budget | Code only; batch not executed |
| EV-007 | `src/batch/preparation.rs:prepare` | code | All entries parse + per-entry preflight + context validation before execution | Static validation only |
| EV-008 | `src/mcp/mod.rs:serve` | code | JSON-RPC loop, concurrency 8, 4 MiB message cap, tool TTL, cancellation | No MCP process run |
| EV-009 | `src/mcp/catalog.rs:tools` | code | Tools generated from clap leaves; `desktop_` prefix; `additionalProperties:false` | Schema quality depends on clap metadata |
| EV-010 | `src/mcp/call.rs` | code | argv rebuild + `current_exe()` child spawn; `sandbox exec` passthrough | No child process run |
| EV-011 | `src/command_policy/mod.rs:policy_for` | code | Full command→permission map; clipboard `None`; screenshot scope conditional | Static declaration |
| EV-012 | `src/command_policy/mod.rs:preflight`, `agent_id_missing`, `preflight_context` | code | Permission denial → `PERM_DENIED`/`not_delivered`; multi-agent gate conjunction | Only checks declared policy |
| EV-013 | `crates/core/src/actions/interaction_policy.rs` | code | Three policy states; OR join; default headless | Does not prove platform honors it |
| EV-014 | `crates/core/src/actions/action.rs` | code | 20 actions; headed requirements; `base_interaction_policy`; PressKey focus fallback | Enum shape, not execution |
| EV-015 | `crates/core/src/actionability/evaluate.rs` | code | Ordered battery; headed downgrade; early refusal; physical `receives_events` append; finish codes | No UI to exercise |
| EV-016 | `crates/core/src/actionability/receives_events.rs` | code | Five candidate points; occlusion evidence; Unknown exception; `require_receives_events` | Hit-test behavior platform-dependent |
| EV-017 | `crates/core/src/actions/delivery_semantics.rs` | code | Five delivery states; retry mapping; inconsistent pair rejection | Serde tests not run |
| EV-018 | `src/dispatch/disposition.rs:settle_unknown_delivery` | code | Unknown → not_delivered only for read-only/`INVALID_ARGS`; other errors wrapped | Relies on command classification |
| EV-019 | `AGENTS.md` (305 lines) | doc | Core isolation rules; no command registry; MCP not second surface; dogfood rules | Prose, partially test-enforced |
| EV-020 | `crates/core/src/actions/action_result.rs` | code | Steps → delivered/verified; write scope `element_value` + `application_commit:not_verified` | No runtime step execution |
| EV-021 | `crates/core/src/commands/helpers.rs` + `helpers_ref_action_wait_result_tests.rs` | code/test | Post-action wait nests action under `after_action`; tests pin shape | Test not executed |
| EV-022 | `crates/core/src/snapshot/mod.rs:SnapshotResult` | code | `complete`, `nodes_observed`, `identity_unproven`; detached save for incomplete | No snapshot run |
| EV-023 | `crates/core/src/snapshot/window_capture.rs` | code | Shallow depth + no token → whole-window attempt at 3/4 deadline; fallback + `identity_unproven` | Timing heuristic not measured |
| EV-024 | `crates/core/src/commands/snapshot.rs:format_snapshot_fields` | code | `complete`; `truncated`/`nodes_observed` only when false; `identity_unproven:true` | Envelope shape, not tree semantics |
| EV-025 | `crates/core/src/refs/ref_token.rs` | code | `@snapshot:eN` qualification; bare `@eN` rejected; snapshot id validation | No parse fuzz run |
| EV-026 | `crates/core/src/refs/ref_capture_proof.rs` | code | Whole-window uniqueness census; unread field rejection; `proves_node` | No capture exercise |
| EV-027 | `crates/core/src/refs/ref_full_predicate.rs` | code | Predicate fields; value/bounds excluded; Unknown fail-closed; native-id completeness | Role-specific stable-name promotion remains |
| EV-028 | `crates/core/src/refs/ref_entry_identity.rs` | code | Identity record shape; `unique_at_capture` flag | No runtime identity check |
| EV-029 | `crates/core/src/refs/mod.rs:RefStore`, `refs_store.rs`, `refs_store_prune.rs` | code | 1 MiB cap, 128/96 retention, 60 s temp, lock/transaction, live-token guard | No store stress test |
| EV-030 | `crates/core/src/session/mod.rs` | code | State root, session dir, 64 KiB manifest, precedence, 300 s liveness, overlay conditions | No session run |
| EV-031 | `crates/core/src/session/interaction_lease.rs` | code | Process+file lock, `/tmp/agent-desktop-<uid>/interaction.lock`, 0700 check, FD inheritance | Unix path only; Windows adapter differs |
| EV-032 | `crates/core/src/trace/mod.rs` | code | 64 MiB file / 1 MiB event caps; lazy open; meta first; per-event lock | No trace write run |
| EV-033 | `crates/core/src/trace/trace_sanitize.rs` | code | Recursive key redaction; sensitive key list; whole keys `actions`/`key`; null preserved | Key-based, not content-based |
| EV-034 | `CONCEPTS.md`, `docs/json-output.md`, `crates/core/src/actionability/requirements.rs`, `skills/agent-desktop-linux/SKILL.md`, `docs/install.md` | doc/code | Battery, completeness semantics, post-action wait, Linux X11/Wayland matrix, OS floors | Docs may drift; code is Fact where they overlap |
| EV-035 | `crates/macos/src/tree/query/node_read.rs`, `crates/macos/src/actions/post_state.rs`, `crates/windows/src/tree/name_evidence.rs`, `crates/windows/src/actions/value_write.rs`, `crates/linux/src/actions/write/secure_write.rs`, `crates/linux/src/tree/capabilities/action_read.rs` | code | Secure field read/write withholding across three platforms; fail-closed IsPassword | No live secure-field test |
| EV-036 | `crates/core/src/trace/trace_artifacts.rs` | code | Artifacts gated by trace+full; screenshot cap 128 MiB; refmap artifact path; only-if-absent | No artifact capture run |
| EV-037 | `crates/linux/src/input/release_guard.rs`, macOS `KeyReleaseGuard`, Windows `KeyReleaseGuard` | code | Arm-before-commit, Drop/finish/abort release, reverse order, journal | Signal/SIGKILL limits |
| EV-038 | `crates/core/src/response/output.rs`, `src/cli/doc_literal_drift_tests.rs`, `docs/json-output.md` | code/doc/test | Envelope `2.6`; fixed success/error fields; drift-tested documented counts | No serialization run |
| EV-039 | `crates/core/src/commands/clipboard_get.rs`, `clipboard_set.rs`, `clipboard_clear.rs` | code | get has no lease; set takes lease; get read-only unless `--out`; private/temp image policy | No clipboard run |
| EV-040 | `src/command_policy/mod.rs:policy_for` clipboard arms | code | Clipboard commands map to `None`; no report requested | Same as EV-011; split for cand. clarity |
| EV-041 | `crates/core/src/commands/skills.rs`, `skills/` tree | code | `include_str!` embedding; canonical/alias/platform/reference model; platform filter | No `skills` command run |
| EV-042 | `scripts/check-architecture.sh`, `scripts/check-rust-file-size.sh`, `check_rust_comments.py`, `check-no-phase-references.sh`, `.github/workflows/ci.yml` | governance | Architecture self-test, isolation checks, 400-line cap, CI gates, 15 MiB cap | Workflows not executed |
| EV-043 | `tests/e2e/scenarios/reliability.sh:92`, `tests/e2e/README.md`, `probes/*/FINDINGS.md` | test/doc | E2E independent effect assertion; probe ledger citation convention | No E2E run |
| EV-044 | `crates/ffi/src/lib.rs`, `crates/ffi/include/agent_desktop.h` | code | C ABI, panic containment, resource/thread-affinity contract, ABI version | No FFI load test |
| EV-046 | `README.md` FFI/ABI section | doc | Public FFI/ABI claims and `panic` contract as documented | Prose may drift; code/header win |
| EV-045 | `src/cli/mod.rs:is_mutating` | code | Exact mutating command set, including `Batch`, `Screenshot --out`, `ClipboardGet --out` | Static classification |

## Evidence class counts

| Class | Count |
|---|---:|
| code | 39 |
| code/test | 1 |
| test/doc | 1 |
| governance | 1 |
| doc/code | 1 |
| doc | 2 |
| code/doc/test | 1 |
| **Total** | **46** |

## Evidence integrity checks

- **Path existence**: every path above exists in the snapshot clone; no path was inferred from memory.
- **Symbol existence**: every symbol named (`PlatformAdapter`, `build_adapter`, `settle_unknown_delivery`, `proves_node`, `InteractionLease`, etc.) was read in source.
- **Line references**: only line numbers that were directly observed are stated (`README.md:128`, `docs/json-output.md:138`, `tests/e2e/scenarios/reliability.sh:92`, `CONCEPTS.md:179/219`). Other rows use symbols.
- **No generated evidence**: no test output, benchmark number, download count, or runtime observation is presented as a source. Where such data would be needed, it is marked as not run (CAND-08, CAND-11).

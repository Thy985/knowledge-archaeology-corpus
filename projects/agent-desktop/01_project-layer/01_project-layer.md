# 01 — Project Layer

## 1. Identity and provenance

| Field | Value | Evidence |
|---|---|---|
| Repository | `vercel-labs/agent-desktop` | `git remote -v` on local clone |
| HEAD | `ce2ddf07226346818de2a1c6bae9869cfad142df` | `git log -1` |
| Branch | `main` | `git branch --show-current` |
| Commit time | `2026-11-23 12:34:52 -0600` | `git log -1 --date=iso` |
| Commit subject | `Use the Vercel Labs lockup in the docs header (#732)` | `git log -1 --format=%s` |
| Tag distance | `v1.3.1-10-gce2ddf0` | `git describe --tags --always` |
| Workspace version | `1.3.1` | `Cargo.toml` |
| npm latest | `1.3.1`, `time.modified 2026-11-13T21:16:06.404Z` | `npm view agent-desktop version dist-tags.latest time.modified repository.url --json` |
| License | repository `LICENSE` present | top-level tree |
| Size | 69M worktree, 44M `.git`, 112M total | `du -sh` |
| Tracked files | 3,806 | `git ls-files \| wc -l` |
| Rust files / lines | 2,827 / 378,939 | `git ls-files '*.rs'` + `wc -l` |
| Rust test files | 1,067 `*_tests.rs` | `git ls-files '*_tests.rs'` |
| `tests/` paths | 287 | `git ls-files 'tests/*'` |

The radar card described `1.2.0`, `60 command names`, `56 operational`, and a `~50k weekly` download figure. Those are **second-hand radar facts, stale relative to this snapshot**. The repository itself says `1.3.1`, `61 command names`, `57 operational` (`README.md:128`), and the drift test computes those counts from clap (`src/cli/doc_literal_drift_tests.rs:75-95`). Download volume was not independently verified and is not used as a fact in this package.

## 2. What the product is

`agent-desktop` is a single Rust binary plus an FFI library that lets an AI agent observe and act on native desktop applications without application-specific integrations. It is explicitly **app-agnostic** and **accessibility-first**:

- Observation comes from macOS Accessibility (AX), Windows UI Automation (UIA), or Linux AT-SPI.
- Actions are semantic where the platform advertises them (AXPress, UIA patterns, AT-SPI actions) and physical input where it does not.
- A snapshot allocates stable `@eN` refs scoped by a snapshot id; refs are meant to survive layout movement but not process replacement.
- Commands emit a versioned JSON envelope (`2.6`) with an explicit delivery disposition and optional recovery hint.
- The same binary exposes an MCP server (`agent-desktop mcp`) whose tools are generated from the CLI command tree.

The product's reference workflow is: **snapshot → find/ref → act → observe postcondition → decide retry**. The architecture is built to make the last two steps honest.

## 3. Repository shape

Top level:

```
.agents/  apps/  assets/  benchmarks/  crates/  docs/  npm/  probes/
providers/  scripts/  skills/  src/  tests/  Cargo.toml  AGENTS.md
CONCEPTS.md  README.md  CHANGELOG.md  CONTRIBUTING.md  SECURITY.md
SUPPORT.md  CODE_OF_CONDUCT.md  LICENSE
```

Workspace members (`Cargo.toml`):

| Crate | Role |
|---|---|
| `crates/core` | Platform-neutral contract, observation model, refs, actions, response, trace, session |
| `crates/macos` | AX adapter |
| `crates/windows` | UIA adapter |
| `crates/linux` | AT-SPI adapter + X11 input/window backend |
| `crates/ffi` | C ABI over `PlatformAdapter` |
| `src` | CLI binary, dispatch, batch, MCP, policy, diagnostics |

`src/` submodules: `cli/`, `cli_args/`, `dispatch/`, `batch/`, `command_policy/`, `diagnostic/`, `visual_debug/`, `mcp/`.

`crates/core/src/` domains: `actionability/`, `actions/`, `adapter/`, `commands/`, `context/`, `cursor_overlay/`, `input/`, `locator/`, `notifications/`, `ref_action/`, `refs/`, `response/`, `session/`, `snapshot/`, `storage/`, `system/`, `trace/`, `tree/`, `wait/`, `sandbox/`.

This is a **deep module graph**, not a flat CLI. The core has 207 files under `crates/core/src/commands/` alone; most are one-command modules or their tests.

## 4. Entrypoints and runtime wiring

### CLI

`src/main.rs` is the process entry. It:

1. Parses `Cli` (clap).
2. Builds the platform adapter in `build_adapter()` (`src/main.rs:358-380`): `MacOSAdapter`, `WindowsAdapter`, or `LinuxAdapter` selected by `#[cfg(target_os)]`; unsupported targets hit `compile_error!`. Linux additionally calls `release_input_on_signals()` before constructing the adapter.
3. Builds a permission report for commands that need one (`src/command_policy/mod.rs:requires_permission_report`).
4. Calls `dispatch::dispatch(...)`, which classifies the command as read-only/mutating, runs the command, settles unknown delivery for read-only/unsent errors, and redacts stdin echo.
5. Wraps the result in `Response { version, ok, command, data?, error? }` and prints one JSON line.

Exit codes: `0` success, `1` command failure, `2` invalid usage (documented in `README.md` and `docs/json-output.md`).

### Dispatch

`src/dispatch/mod.rs:dispatch` has no command registry. It computes:

```rust
let read_only = !cmd.is_mutating();
let result = dispatch_command(cmd, adapter, permission_report, context)
    .map_err(|error| disposition::settle_unknown_delivery(error, read_only));
```

`dispatch_command` is a direct `match` over `Commands`, with arms grouped into `observation`, `interaction`, `keyboard_mouse`, `apps`, `notifications`, `clipboard`, `session`, `sandbox`, `trace`. Each arm calls a core `commands::*::execute()` or a binary-layer module. There is no `Command` trait, no dynamic registry, no plugin lookup.

### Batch

`agent-desktop batch --stdin` reads a JSON object with a `commands` array. `src/batch/mod.rs` parses each entry into the same `Commands` enum via `parse_command`; `src/batch/execution.rs` then executes them **sequentially, non-atomically** through the same `dispatch`. The output explicitly states:

```json
"semantics": {
  "atomic": false,
  "order": "sequential",
  "batch_retries": false,
  "command_retry_contracts": "preserved",
  "successful_action_disposition": "data.disposition",
  "error_disposition": "error.disposition"
}
```

Batch is not a transaction. It is a scripted sequence with per-entry dispositions and a batch-level deadline/output budget.

### MCP

`src/mcp/mod.rs:serve` reads JSON-RPC lines from stdin. `catalog::tools()` walks `Cli::command()` leaves, prefixes each with `desktop_`, joins the command path with `-`→`_`, and derives `inputSchema` from clap args/help. `call.rs` reconstructs argv from the JSON arguments and **re-spawns `std::env::current_exe()`**, so each MCP call executes the same binary in a child process. `sandbox exec` is special-cased as `Output::Passthrough`; other commands parse JSON envelopes and convert error envelopes to `isError`.

Concurrency limits: `MAX_CONCURRENT_CALLS = 8`, `MAX_MESSAGE_BYTES = 4 MiB`, `TOOLS_TTL_MS = 86_400_000`. A `Calls::admit` guard rejects same-id re-entry; `notifications/cancelled` cancels the child.

MCP is **not** a second command surface: it is mounted on the root command, is not in `Commands`, not a tool count, not a batch entry.

## 5. Core abstractions

### PlatformAdapter

`crates/core/src/adapter/mod.rs`:

```rust
pub trait PlatformAdapter: ObservationOps + ActionOps + InputOps + SystemOps {}
impl<T: ObservationOps + ActionOps + InputOps + SystemOps> PlatformAdapter for T {}
```

The four sub-traits are the capability contract:

- `ObservationOps`: list apps/windows/displays/surfaces, capture tree, hit test, live read, signal baseline.
- `ActionOps`: perform an `Action`, check actionability.
- `InputOps`: physical keyboard/mouse, clipboard, cursor position.
- `SystemOps`: permissions, screenshot, app lifecycle, notifications, interaction lease, cursor overlay.

Every method has a default body returning `AdapterError::not_supported(...)`; a platform must opt in. This is the code-level expression of "absence of evidence is not evidence of capability".

### Action / ActionRequest / ActionStep

`crates/core/src/actions/action.rs` defines 20 semantic actions (click, double/triple/right click, set-value, clear, focus, select, toggle, check/uncheck, expand/collapse, scroll, scroll-to, press, key-down/up, type, hover, drag). Each has a `headed_requirement()`:

- pointer actions → `FocusedWindowAndCursor`
- state/text actions → `FocusedWindow`
- `base_interaction_policy()` defaults `PressKey` to `focus_fallback`, everything else to `headless`.

`InteractionPolicy` is three-valued: `headless(false,false)`, `focus_fallback(true,false)`, `headed(true,true)`. `join` is OR. This third state exists because a physical key event may need a focused window but not a cursor.

### Snapshot / Ref

`SnapshotResult` carries `tree`, `refmap`, `owner`, `snapshot_id`, `complete`, `nodes_observed`, `identity_unproven`. `RefEntryIdentity` carries `role`, optional `name`, `value`, `description`, `native_id`, `retained_object`, and `unique_at_capture`. `FullPredicate` compares only `role + native_id + stable name + description`; mutable value and geometry are never identity.

Ref syntax is `@<snapshot_id>:e<N>`. The parser requires `@`, `rsplit_once(":e")`, a valid snapshot id, and a positive decimal local ref; bare `@eN` is `INVALID_ARGS`. Snapshot ids are base36, `s`-prefixed, 3–64 chars, generated from nanos + pid + counter + random state.

### Response / Delivery

`Response` is `{ version, ok, command, data?, error? }`. `ErrorPayload` adds `code`, `message`, `suggestion?`, `recovery?`, `platform_detail?`, `details?`, `disposition`.

`DeliverySemantics` is five-state: `unknown`, `not_delivered`, `delivery_uncertain`, `delivered_unverified`, `delivered_verified`. `not_delivered` maps `retry: safe`; the other known states map `retry: unsafe`; unknown maps unknown. Deserialization rejects a `delivery`/`retry` pair that disagrees (`actions/delivery_semantics.rs`).

## 6. State, memory and lifecycle

### Session

State root is `~/.agent-desktop` (override `AGENT_DESKTOP_HOME`). Session layout is `<root>/sessions/<id>/`. `session.json` is the manifest, read with a 64 KiB cap. Session resolution precedence: explicit `--session` > `AGENT_DESKTOP_SESSION` > none. `TRACE_LIVENESS_WINDOW = 300s` gates trace use. A session can be ended; ended sessions are rejected for batch entries.

### RefStore

- `MAX_REFMAP_BYTES = 1_048_576`
- `MAX_SAVED_SNAPSHOTS = 128`, prune low-water `96`, stale temp max age `60s`
- paths: `<base>/snapshots/<id>/refmap.json`, `<base>/refstore.lock`, `<base>/latest_snapshot_id`
- `save_new_snapshot` holds the lock, saves snapshot + latest, prunes
- `update_existing_snapshot` takes expected ref/entry to prevent lost updates and does load-update-save under the lock
- `retained_object_tokens` scans snapshots and refuses deletion if a live token would be removed

### Trace

`MAX_TRACE_FILE_BYTES = 64 MiB`, `MAX_TRACE_EVENT_BYTES = 1 MiB`. Trace is lazy-open; `TracePending::None` writes nothing. First line is `trace.meta` (schema/version/os/pid/process start). Writers add an exclusive lock per event. `trace_sanitize` recursively redacts a fixed sensitive-key list (`text`, `value`, `expected`, `name`, `username`, `description`, `label`, `query`, `selector`, `secret`, `token`, `password`, `title`, `url`, `help`, `placeholder`, `subrole`, `dom`, `classes`, `body`; whole keys `actions`, `key`). Values become `{"redacted":true}`; nulls stay null.

Trace artifacts are a separate channel: `artifacts_enabled = trace_enabled && artifacts_full`, screenshots require `session start --screenshots`, single embedded screenshot cap 128 MiB, refmap copies are written only if absent. `CONCEPTS.md:179` states artifacts are unredacted and may appear in exported HTML.

### Interaction lease

On Unix, `InteractionLease` combines a `ProcessLeaseGuard` and a `FileLock` at `/tmp/agent-desktop-<uid>/interaction.lock`; the directory must be user-owned, non-symlink, mode 0700. The lease is machine-wide per user, not per session. It can be inherited across a subprocess via an FD. Contention is counted and reported.

## 7. Permissions and governance

`src/command_policy/mod.rs:policy_for` maps:

- `Version`, `Skills`, `Session`, `CursorOverlay`, `Sandbox`, `Trace`, `Status`, `Permissions` → `None`
- `ListWindows`, `ListDisplays`, `ListApps`, `ClipboardGet`, `ClipboardSet`, `ClipboardClear`, `Batch` → `None`
- `Snapshot`, `Find`, `ListSurfaces`, `Wait`, `ListNotifications`, `Get`, `Is`, all ref actions, `Type`, `Focus`, `Press`, `KeyDown`, `KeyUp`, `Hover`, `Drag`, `Mouse*`, `Launch`, `CloseApp`, `FocusWindow`, `OpenSystemSurface`, window mutations, notification mutations → `Accessibility`
- `Screenshot` with app/window → `AccessibilityAndScreenRecording`; no scope → `ScreenRecording`

`preflight` converts a denied permission into `PERM_DENIED` + `not_delivered`. `preflight_context` rejects a mutating accessibility command in a multi-agent session without `--agent-id`. The gate is intentionally narrow: it does not apply to clipboard (policy `None`) or non-accessibility mutating commands.

`requires_permission_report` returns true for any non-`None` policy, plus `Status`, `Permissions`, and `Batch`. Clipboard commands therefore do **not** request a permission report.

## 8. Test and quality system

CI workflows (7): `ci.yml`, `linux-e2e.yml`, `native-e2e.yml`, `release-dry-run.yml`, `release.yml`, `supply-chain.yml`, `windows-capability-probe.yml`.

`ci.yml` includes: fmt, docs build, MSRV 1.89, macOS tests, core isolation (`cargo tree -p agent-desktop-core` has no platform deps), clippy `-D warnings`, core/macOS unit tests, native cursor tests, FFI tests, 15 MiB binary/helper cap, FFI cdylib, E2E harness shellcheck/py_compile/unittest, actionlint, CI hygiene.

Architecture checks (`scripts/check-architecture.sh`): a self-test injects a violating fixture first, then checks core → platform dependency, platform ↔ platform dependency, allowed OS `cfg` sites in core, crate top-level layout, capability-layer location, core domains not reaching into `commands`, every test file declared. File-size check caps hand-written `.rs` at 400 lines. `check_rust_comments.py` bans inline comments. `check-no-phase-references.sh` bans internal phase references. `check-stale-ref-constructor-misuse.sh` guards a known ref constructor hazard.

Tests are organized as unit tests per module plus golden JSON contracts plus E2E. E2E rule: every effect is observed independently; the command's own `ok:true` is not trusted. `tests/e2e/scenarios/reliability.sh:92` asserts `data.after_action.action`. `probes/{macos,windows,linux}` contain re-runnable measurements and `FINDINGS.md` ledgers with IDs such as `A15-7`, `A19-3`.

## 9. Extension surface

Adding a command is a **compile-time, multi-site change**, documented in `AGENTS.md`:

1. core command module in `crates/core/src/commands/`
2. CLI variant in `src/cli/`
3. dispatch arm in `src/dispatch/`
4. permission mapping in `src/command_policy/`
5. batch parser support
6. README/AGENTS command counts
7. `Action` variant if it is a semantic action
8. adapter method(s) on each platform
9. skills documentation under `skills/`

The absence of a registry is deliberate: it makes the change surface explicit and lets drift tests catch omissions. The cost is that every command addition touches many files and all three platform crates.

Platform extension is different: implement the four sub-traits in a new crate, wire one `#[cfg]` arm in `build_adapter`, and add a platform skills package. The core does not need to know the platform name.

## 10. Project-layer observations (not yet EK/KO)

- **OBS-01**: The repository is unusually documentation-dense for a CLI: `AGENTS.md` 305 lines / 22,722 bytes, `CONCEPTS.md` 359 lines / 44,254 bytes, `README.md` 190 lines. These are not marketing docs; they encode invariants that tests partially enforce.
- **OBS-02**: `skills/` is compiled into the binary via `include_str!`, so the agent-facing instructions ship with the exact binary version rather than as separate mutable docs.
- **OBS-03**: `probes/` is a first-class evidence system, not scratch code. Findings IDs are cited from code comments, which gives the repository an internal citation graph unusual for an OSS CLI.
- **OBS-04**: The Linux adapter has a distinct capability shape: AT-SPI observation is portable across X11/Wayland, but pointer/window control is X11-only. The Linux skill documents this precisely; it is a product constraint, not an implementation bug.
- **OBS-05**: `crates/core/src/commands/` is large (207 entries) because the one-command-per-file rule is enforced. File count is a governance artifact, not a complexity signal by itself.

## 11. Boundary of this layer

The Project Layer records **what the repository is and how it is wired**. It does not attempt to explain why each design choice is correct, nor does it resolve the candidates in `05_candidates/`. Mechanism-level claims live in `02_engineering-knowledge/`; abstraction-level claims live in `03_knowledge-layer/`.

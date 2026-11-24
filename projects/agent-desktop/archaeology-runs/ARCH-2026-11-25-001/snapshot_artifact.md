# Repository Snapshot — agent-desktop

Run: `ARCH-2026-11-25-001`
Job: `KA-2026-11-25-001`
Snapshot time: 2026-11-25 (Asia/Shanghai)
Repository: `https://github.com/vercel-labs/agent-desktop.git`

## Provenance

| Fact | Value | Evidence |
|---|---|---|
| HEAD commit | `ce2ddf07226346818de2a1c6bae9869cfad142df` | `git log -1 --format=%H` |
| Branch | `main` | `git rev-parse --abbrev-ref HEAD` |
| Commit time | `2026-11-23 12:34:52 -0600` | `git log -1 --date=iso` |
| Commit subject | `Use the Vercel Labs lockup in the docs header (#732)` | `git log -1 --format=%s` |
| Describe | `v1.3.1-10-gce2ddf0` | `git describe --tags --always` |
| Clone depth | 50 | clone command |
| Working tree | 69 MB | `du -sh --exclude=.git .` |
| `.git` | 44 MB | `du -sh .git` |
| Tracked files | 3,806 | `git ls-files \| wc -l` |
| Rust files | 2,827 | `git ls-files '*.rs' \| wc -l` |
| Rust lines | 378,939 | `git ls-files '*.rs' \| xargs cat \| wc -l` |
| Rust test files | 1,067 (`*_tests.rs`) | `git ls-files '*_tests.rs' \| wc -l` |
| Markdown files | 77 | `git ls-files '*.md' \| wc -l` |
| GitHub workflows | 7 | `git ls-files '.github/workflows/*'` |
| npm latest | `1.3.1` | `npm view agent-desktop version` |
| npm modified | `2026-11-13T21:16:06.404Z` | `npm view agent-desktop time.modified` |
| npm repository | `git+https://github.com/vercel-labs/agent-desktop.git` | `npm view agent-desktop repository.url` |
| Workspace version | `1.3.1` | `Cargo.toml` |
| Edition / MSRV | 2024 / 1.89 | `Cargo.toml`, `.github/workflows/ci.yml` |
| License | Apache-2.0 | `LICENSE` |

The npm record is older than the repository HEAD: HEAD is 10 commits past tag `v1.3.1`. The npm package and the Cargo workspace agree on `1.3.1`; no claim is made here that the npm tarball contains the HEAD tree.

## Language and layout

Top-level tracked areas:

`.agents/`, `apps/`, `assets/`, `benchmarks/`, `crates/`, `docs/`, `npm/`, `probes/`, `providers/`, `scripts/`, `skills/`, `src/`, `tests/`, plus `Cargo.toml`, `AGENTS.md`, `CONCEPTS.md`, `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `SECURITY.md`, `SUPPORT.md`, `CODE_OF_CONDUCT.md`, `LICENSE`.

File-extension distribution (tracked paths, first 20 by count):

| Ext | Count |
|---|---|
| rs | 2,827 |
| json | 201 |
| ps1 | 172 |
| py | 167 |
| md | 77 |
| sh | 60 |
| mdx | 42 |
| normalized | 38 |
| cs | 27 |
| psm1 | 24 |
| js | 20 |
| ts | 18 |
| toml | 15 |
| m | 14 |
| tsx | 13 |
| swift | 13 |
| mjs | 12 |
| yml | 10 |
| html | 9 |
| c | 6 |

`Cargo.toml` is a virtual workspace with members `crates/core`, `crates/macos`, `crates/windows`, `crates/linux`, `crates/ffi`, `src`. Release profile: `opt-level = "z"`, LTO, `panic = "abort"`.

## Entrypoints and runtime wiring

| Entry | File | Behaviour |
|---|---|---|
| Binary `agent-desktop` | `src/main.rs` | Parses clap tree, handles `mcp` before `Cli`, builds one `PlatformAdapter`, resolves session/context, runs policy preflight, dispatches |
| MCP server | `src/mcp/mod.rs` | `agent-desktop mcp` serves JSON-RPC over stdio; every leaf CLI command becomes one `desktop_*` tool |
| C ABI | `crates/ffi/src/` | Opaque ids, ABI version, session lease, ref-action parity |
| Platform adapters | `crates/{macos,windows,linux}/src/adapter.rs` | Implement `PlatformAdapter`; selected at compile time by `#[cfg(target_os)]` |
| Sandbox provider | `providers/macos/`, `src/dispatch/sandbox.rs` | macOS VM provider, guest agent, `sandbox exec` passthrough |

`build_adapter()` has exactly three `#[cfg(target_os)]` branches and a `compile_error!` fallback (`src/main.rs:358-380`). Core never imports a platform crate; `AGENTS.md` states this and CI enforces it with `cargo tree -p agent-desktop-core`.

## Core abstractions

| Abstraction | Location | Contract |
|---|---|---|
| `PlatformAdapter` | `crates/core/src/adapter/mod.rs` | Blanket trait over `ObservationOps + ActionOps + InputOps + SystemOps` |
| Adapter defaults | `adapter/{observation,actions,input,system}.rs` | Each operation defaults to `AdapterError::not_supported`; capabilities are implemented, not assumed |
| `Action` / `ActionRequest` | `actions/action.rs`, `action_request.rs` | 20 platform-neutral actions, each declaring headed requirement, keyboard delivery, surface raising, scroll-into-view and focus fallback |
| `InteractionPolicy` | `actions/interaction_policy.rs` | Three states: headless (false/false), focus-fallback (true/false), headed (true/true); default headless |
| `DeliverySemantics` | `actions/delivery_semantics.rs` | Five states; `not_delivered` → retry safe, all other non-unknown → retry unsafe; deserialization rejects inconsistent pairs |
| `RefMap` / `RefStore` | `refs/mod.rs`, `refs_store/` | Snapshot-scoped `@s<id>:e<N>` refs, 1 MiB cap, 128-snapshot retention/prune to 96, session namespaces |
| `SnapshotResult` | `snapshot/mod.rs` | Tree + refmap + owner + snapshot id + completeness + nodes observed + identity_unproven |
| `Session` | `session/mod.rs` | `sessions/<id>/session.json`, trace on by default, explicit session > env > none |
| `CommandContext` | `context/` | Session, agent id, headed flag, trace, wait selector, deadline, event baseline |
| `CommandPolicy` | `src/command_policy/mod.rs` | Command-to-permission map, arg validation, multi-agent mutating gate |
| `Response` / `ErrorPayload` | `crates/core/src/response/output.rs` | Version `2.6`, `ok`/`command`/`data` or `error`, error always carries disposition |
| `TraceConfig` | `crates/core/src/trace/mod.rs` | Per-process JSONL segments, lazy open, 64 MiB cap, sanitize map, strict mode |
| `OperationReport` | `system/` + action results | Post-2.5 action outcomes no longer carry unconditional success booleans |

## State and persistence map

| State | Root / file | Lifecycle | Evidence |
|---|---|---|---|
| State root | `~/.agent-desktop`, override `AGENT_DESKTOP_HOME` | Must be private; validated pre-dispatch | `validate_state_root_env`, `AGENTS.md` |
| Snapshot refmap | `snapshots/<snapshot_id>/refmap.json` | ≤1 MiB; prune at 128 to 96; lock `refstore.lock`; latest pointer `latest_snapshot_id` | `refs/mod.rs` |
| Legacy refmap | `last_refmap.json` | Read-only migration in global namespace | `refs_store` |
| Session | `sessions/<id>/session.json` | 64 KiB manifest cap; `session start` writes but does not activate | `session/mod.rs` |
| Trace | session trace dir, `<pid>-<procStartTs>.jsonl` | Append-only per process; merged by `(ts_ms, writer pid, position)`; additive schema | `trace/`, `CONCEPTS.md` |
| Replay artifacts | `trace/screens/*.png`, `trace/refmaps/<snapshot>.json` | Only `session start --screenshots`; not redacted | `trace_artifacts.rs`, `CONCEPTS.md` |
| Cursor overlay profile | session/agent scoped | Requires live manifest and overlay enabled | `session/mod.rs` |
| Held input journal | Linux X11 `HeldJournal` | Records held keys/buttons and keymap edits for recovery | `crates/linux/src/input/release_guard.rs` |

## Dispatch and command surface

- README states **61 command names, 57 operational**; four held-input commands (`key-down`, `key-up`, `mouse-down`, `mouse-up`) fail closed.
- `src/dispatch/mod.rs` is a direct `match` over `Commands`; each arm delegates to `crates/core/src/commands/*::execute()` or a binary-layer module. There is no command trait or runtime registry.
- Batch is **not** a second dispatcher: `src/batch/mod.rs` deserialises each entry into the same `Commands` enum, runs the same `CommandPolicy` preflight via `prepare`, then calls the same `dispatch()` (`src/batch/execution.rs:106`).
- Batch semantics are explicitly non-atomic, sequential, no batch-level retries, and preserve per-command retry contracts (`src/batch/execution.rs:132-151`). It captures a signal baseline for a following `wait --event` and refuses to start the preceding command if baseline capture fails (`execution.rs:61-95`).
- MCP is **not** a second command surface: `catalog::tools()` walks the built clap tree, emits one tool per leaf, derives input schema from clap args, and `call.rs` rebuilds argv and spawns **this same binary** as a child. `mcp` is attached to the root command and is not a `Commands` variant (`src/main.rs:118-121`, `src/mcp/mod.rs:37-42`).
- MCP limits: 8 concurrent calls, 4 MiB message cap, duplicate in-flight request id rejected, `sandbox exec` is passthrough and receives no server globals (`src/mcp/mod.rs:19-21,79-92`, `call.rs:70-85`).

## Error and delivery model

`ErrorPayload` always serialises `code`, `message`, `disposition`; optional `suggestion`, `recovery`, `platform_detail`, `details`. `DeliverySemantics` serialises as `{delivery, retry}`. Recovery hints are emitted only when retry is safe and the code is `STALE_REF`, `SNAPSHOT_NOT_FOUND`, `POLICY_DENIED`, `APP_UNRESPONSIVE` or `TIMEOUT` (`response/output.rs:46-86`).

`pre_dispatch_error` forces `not_delivered` for failures before dispatch (`src/main.rs:323-333`). `dispatch::dispatch` settles unknown delivery to `not_delivered` only for read-only commands (`src/dispatch/mod.rs:32-37`). This preserves the rule that a mutating command with no delivery evidence may have landed.

## Permissions and governance

`policy_for` maps every `Commands` variant to `None`, `Accessibility`, `ScreenRecording`, or both (`src/command_policy/mod.rs:15-81`). Notable mappings:

- `status`, `permissions`, `list-windows`, `list-displays`, `list-apps`, clipboard read/write/clear, batch, skills, session, sandbox, trace → `None`.
- snapshot/find/get/is, all ref actions, keyboard/mouse, app/window mutations, notifications → `Accessibility`.
- screenshot on an app/window → both; whole-display screenshot → `ScreenRecording`.

`preflight` refuses missing accessibility or screen-recording permission with `PERM_DENIED` and `not_delivered` (`command_policy/mod.rs:83-113`). `preflight_context` requires `--agent-id` for a mutating command **only when its permission need is `Accessibility`** in a multi-agent session (`command_policy/mod.rs:115-133`).

`AGENTS.md` governance rules observed in code/config:

- app-agnostic fixes; no branching on app name, bundle id, executable, title or known layout;
- core defines contracts, platform crates implement; core never imports platform crates; platform crates do not import each other;
- one domain struct/enum per file, one command per file, handwritten `.rs` ≤400 lines, no inline comments;
- no `unwrap`/`expect`/`panic!` outside tests; adapters return `Result<_, AdapterError>`;
- command count and docs drift checked by `doc_literal_drift_tests` (documented in `AGENTS.md`);
- `scripts/check-architecture.sh` self-tests with a deliberately invalid fixture before checking the real tree;
- CI: fmt, docs build, MSRV 1.89, core isolation, clippy `-D warnings`, core/macOS tests, FFI tests, 15 MiB binary cap, FFI header drift, release dry-run, supply chain, E2E.

## Test system

- 1,067 `*_tests.rs` files; unit tests use adapter stubs (`tests/support/noop_ops.rs`, `adapter/test_support.rs`).
- 287 paths under `tests/`; includes `tests/e2e/`, `tests/e2e-linux/`, `tests/real-apps/`, `tests/native/`, FFI integration tests.
- Golden JSON fixtures and conformance crate; `snapshot_test`, `conformance`, `cli_process`, `cli_stdin`, `mcp_process`, `command_host_absent`, Windows-specific binaries are declared in `src/Cargo.toml`.
- E2E rule from `AGENTS.md`: each effect is observed independently; the command's own `ok: true` is not evidence. This is visible in tests asserting `data.after_action` and in `docs/solutions/`.
- `scripts/cargo-test-isolated-home.sh` runs tests under an isolated `HOME` so tests do not touch the real `~/.agent-desktop`.
- `probes/{macos,windows,linux}` hold rerunnable measurements and `FINDINGS.md` anchors (A15-7, L32-39 style).

## External dependencies and platform requirements

- macOS 13+ Accessibility (AXUIElement); Windows 10 1809+ UI Automation; Linux Ubuntu 24.04+ AT-SPI with glibc ≥2.35.
- Linux: observation and semantic ref actions work on X11 and Wayland; pointer gestures, wheel scrolling, window control/focus-window and headed shell input require X11. Wayland headed pointer input refuses `ACTION_NOT_SUPPORTED`.
- Clipboard on Wayland needs `wl-clipboard`.
- Chromium/Qt apps need accessibility launch flags; Electron needs `--no-sandbox` where `chrome-sandbox` is not setuid.
- Sandbox provider exists only for macOS (Apple Silicon, macOS 13+); Linux/Windows have no provider.

## Snapshot observations and conflict resolution

1. The radar candidate card records npm `1.2.0` and 60/56 commands; the repository at HEAD records 61/57, Cargo `1.3.1`, npm latest `1.3.1`. **Repository and registry facts supersede the card.**
2. The scan card says Linux mouse/window control may depend on X11. The repository's Linux skill resolves this precisely: observation/semantic refs work on both; pointer gestures, wheel scrolling, window control and `focus-window` are X11-only; Wayland refuses headed pointer input.
3. No desktop session or GUI exists in this environment. Runtime behaviour is therefore **not** executed here. Snapshot facts are from source, manifests, CI config, docs and registry metadata. Claims about live actionability remain Observation/Hypothesis until validated against a running desktop.
4. The repository was cloned read-only; no build, test, lint or E2E command was run. Test-system facts describe declared tests and CI wiring, not this run's results.

## Snapshot boundary

Covered: repository provenance, layout, build graph, entrypoints, core abstractions, state/persistence, dispatch and command surfaces, error/delivery model, permissions, governance, tests, platform requirements.

Not covered in this artifact: per-file implementation detail, full Flow Atlas edges, Engineering Knowledge extraction, validation. Those belong to stages 04-06.

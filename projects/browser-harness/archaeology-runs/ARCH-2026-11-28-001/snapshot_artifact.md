# Repository Snapshot — browser-harness v0.1.15

- run_id: `ARCH-2026-11-28-001`
- job_id: `KA-2026-11-28-001`
- repository: `https://github.com/browser-use/browser-harness.git`
- ref: tag `v0.1.15`
- commit: `373aecc4c40ef67d97b5ee69c144684a64a42a11`
- commit time: `2026-11-25 17:25:45 +0530`
- commit subject: `Merge pull request #1077 from browser-use/release/0.1.15-telemetry`
- clone: `git clone --depth 1 --branch v0.1.15`; `is-shallow-repository=true`; detached HEAD
- size: 12 MB working tree; 299 tracked files
- Python: 15,258 lines under `src/**/*.py` + `tests/**/*.py` (8,428 source + 6,830 tests); 404 `def test_` occurrences
- snapshot timestamp: 2026-11-28 (Asia/Shanghai)
- local install/tests: **not executed**; no `.venv`; repository unmodified

## 1. Project identity and entry points

`pyproject.toml` declares name `browser-harness`, version `0.1.15`, MIT, Python `>=3.11`, setuptools build backend `84.0.0`. Console scripts:

- `browser-harness` → `browser_harness.run:main`
- `browser-harness-mcp` → `browser_harness.mcp_cli:main`

Runtime dependency pins: `cdp-use==1.4.5`, `fetch-use==0.4.0`, `pillow==13.0.0`, `websockets==15.0.1`; optional `mcp==2.1.1`. Pytest sets `pythonpath = ["src"]`.

`README.md` positions the package as "the engine inside Browser Use": the `browser-use` CLI is the recommended entry, while `browser-harness` still works and keeps domain/interaction skills. `AGENTS.md` fixes the package/CLI name as `browser-harness` and the Skill identity as `browser-use` ("do not rename"). `SKILL.md` frontmatter, by contrast, is `name: browser-harness` — the same naming split recorded in the 0.1.13 baseline.

## 2. Module map (source, 8,428 lines)

| Module | Lines | Responsibility (from code/docstrings) |
|---|---:|---|
| `src/browser_harness/admin.py` | 1,810 | daemon lifecycle, spawn lock, protocol/endpoint checks, doctor, update, Chrome launch/approval, cloud browser lifecycle |
| `src/browser_harness/daemon.py` | 1,422 | long-lived CDP WebSocket holder + IPC relay; owner/tab/session routing; idle reclamation |
| `src/browser_harness/helpers.py` | 844 | CDP primitives auto-imported into CLI stdin scripts; tab/session helpers |
| `src/browser_harness/video.py` | 751 | video recording/export |
| `src/browser_harness/auth.py` | 569 | Browser Use Cloud auth |
| `src/browser_harness/video_render.py` | 525 | video rendering |
| `src/browser_harness/run.py` | 481 | CLI entry, named-local-daemon→owner aliasing, help/usage |
| `src/mcp_server.py` (top-level py-module) | 407 | optional MCP tool surface; `browser_harness/mcp_cli.py` imports it lazily |
| `src/browser_harness/report.py` | 368 | feedback/run reporting |
| `src/browser_harness/recorder.py` | 346 | local action recording |
| `src/browser_harness/telemetry.py` | 338 | telemetry consent and redaction |
| `src/browser_harness/_ipc.py` | 288 | AF_UNIX (POSIX) / TCP loopback + token (Windows) IPC; owner identity; runtime paths |
| `src/browser_harness/macos.py` | 196 | macOS approval automation |
| `src/browser_harness/paths.py` | 49 | private config/runtime/tmp/workspace directories |
| `src/browser_harness/naming.py` | 16 | CLI display name |
| `src/browser_harness/mcp_cli.py` | 16 | MCP CLI entry |

`agent-workspace/agent_helpers.py` is 7 lines; `agent-workspace/domain-skills/`, `interaction-skills/`, `skills/`, `docs/` are content assets, not core runtime.

## 3. Core runtime objects and state

`daemon.py:487` `Owner` is the per-agent state: `tabs` (target_id → session_id), `current`, `events` (deque maxlen 500), `dialog`, `gone` (one-shot typed error), `seen`, `lock`, `name`, `marks`, `created`, `released`.

`daemon.py:503` `Daemon` is the shared browser connection: `owners`, `tab_owner`, `session_owner`, `info`, `downloads`, `own_targets`, `browser_sessions`, plus captcha state. Docstring: "One browser connection shared by many owners. A tab has at most one owner."

Persistent state is `ipc.state_path(NAME)` → `bu-<name>.owners.json` (`_ipc.py:74`), written by `Daemon._save` (`daemon.py:635`) and restored by `_load` (`daemon.py:645`). The module comment at `daemon.py:34` states the intent: "owner -> tabs, so tabs survive a daemon restart".

## 4. Configuration surface

- `BU_NAME` selects the daemon endpoint/name; default `default`.
- `BH_OWNER` is the agent identity; fallback order `BH_OWNER` → `CLAUDE_CODE_SESSION_ID` → `CODEX_THREAD_ID` → `"default"` (`_ipc.py:36`).
- `BH_OWNER_IDLE` default 1800 s; `BH_OWNER_FORGET` default 86400 s (`daemon.py:146-147`).
- `BH_IDLE_TIMEOUT` opt-in daemon idle exit; default 1800 s only when `BH_OWNS_BROWSER=1`, otherwise off (`daemon.py:164`).
- `BH_IDLE_EXIT` default 1800 s for unused named local daemons (`daemon.py:148`).
- `BH_NAMED_LOCAL_DAEMON=1` prevents `BU_NAME` aliasing to a shared local owner (`run.py:39`).
- `BH_REQUIRE_EXISTING_DAEMON=1` makes a daemon non-idle-exiting and forbids spawning (`admin.py:859`).
- `BH_RUNTIME_DIR`, `BH_TMP_DIR`, `BH_HOME`/`BROWSER_HARNESS_HOME`, `BH_AGENT_WORKSPACE`, `BH_CONFIG_DIR` set filesystem roots (`_ipc.py:17`, `paths.py`).
- `BH_IPC_TIMEOUT` raises the 30 s slow-CDP budget (`helpers.py:60`).
- `BH_DOMAIN_SKILLS=1` enables domain skill lookup (`helpers.py`/`SKILL.md`).
- `BU_CDP_WS` / `BU_CDP_URL` select an external browser; `BU_BROWSER_ID` selects a cloud browser; `BH_OWNS_BROWSER=1` allows closing a daemon-owned browser on idle exit (`daemon.py:119-121`).

Filesystem isolation: `paths.ensure_private_dir` creates new directories `0700` on non-Windows (`paths.py:19`); POSIX IPC socket is bound under `umask 0o077` so it is `0600` with no chmod TOCTOU (`_ipc.py:253`); Windows uses `127.0.0.1` + a per-daemon random 64-hex token (`_ipc.py:264-272`).

## 5. Permission and governance mechanisms

- **Owner identity** is caller-supplied, not authenticated. `_ipc.owner()` trusts `BH_OWNER` / host session env vars. A sub-agent that inherits the parent's session id is the parent's owner; `helpers._shares_parents_owner()` detects the Claude child marker and refuses `close_tab`/`release_tabs` unless `BH_OWNER` is set (`helpers.py:109-118`).
- **Tab ownership** is enforced in the daemon: `tab_owner` maps target→owner; `session_owner` maps session→owner; `GUARDED = ("Target.attachToTarget", "Target.closeTarget", "Target.activateTarget")` prevents those browser-level methods on another owner's tab (`daemon.py:37`, `daemon.py:1210-1219`).
- **Iframe reachability**: `Target.attachToTarget` on an iframe is rejected with `frame-not-yours` unless `_frame_tab` can prove the frame belongs to one of the caller's tabs (`daemon.py:1216-1219`, `daemon.py:560-576`).
- **Visibility**: `_visible` blanks `url`/`title` for another agent's tab and for iframes not rooted in the caller's tab (`daemon.py:548-556`).
- **Idle reclamation**: `sweep` releases non-current tabs after `OWNER_IDLE`, last tab after `OWNER_FORGET`; a returning owner gets `tab-released` (`daemon.py:146-147`, `daemon.py:727-750`).
- **Daemon identity/competition**: readiness requires a `ping` that returns `{"pong": true}`, not a bare socket connect (`admin.py:391`); `_restart_if_outdated` refuses to stop a newer daemon or a daemon with active owners (`admin.py:422-446`); `_check_cdp_endpoint` rejects a name already bound to another CDP endpoint (`admin.py:587-620`); one `_spawn_lock` per daemon name serializes cold spawns (`admin.py:644-720`).
- **Shutdown**: `SIGTERM`/`SIGHUP` run the normal stop path; `BH_OWNER_PID` (opt-in) exits the daemon when that process dies (`daemon.py:1301-1341`, `daemon.py:1362-1378`).

## 6. Test system

Unit tests: `tests/unit/` (17 files), run with `uv run --with pytest python -m pytest tests/unit -q` (`AGENTS.md`). Largest suites: `test_admin.py` 1,872 lines, `test_daemon.py` 1,487, `test_helpers.py` 1,002, `test_run.py` 409, `test_report.py` 349, `test_ipc.py` 291. Integration tests: `tests/integration/` — `test_js.py` 193, `test_network_idle.py` 132, `test_reattach.py` 56, `test_solve_captcha.py` 56.

Ownership behavior is covered directly: `tab-taken` at `test_daemon.py:367,468,481,533,551,563,699,952,1009`; `tab-closed` at `:388,409,457,576,589,749,793`; `owner-shared` at `test_helpers.py:707`; `daemon-outdated` at `test_admin.py:239-249`; `remote-browser-ended` at `test_admin.py:440-454`. Idle release, last-page preservation, stale-session non-redirection, owner-state survival across daemon restart, and "only the current tab streams Network events" each have dedicated tests (`test_daemon.py:970-1221,1448-1477`).

No test command was executed in this snapshot; pass/fail is not claimed.

## 7. External dependencies and boundaries

Runtime talks to (a) a local Chrome/Chromium-family browser over CDP, (b) Browser Use Cloud `https://api.browser-use.com/api/v3`, and (c) `fetch-use` for HTTP fetch. It holds no database. The only durable local state is the owners JSON, config dir, recordings, and logs. The repository contains no submodules in the shallow clone. Target-project writes: none.

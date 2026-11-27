# 01 Project Layer — browser-harness 0.1.15

## Architecture

Three planes:

1. **Caller plane** — one short-lived `browser-harness` CLI process (or an embedding CLI / MCP server / the browser-use REPL worker). It reads a Python heredoc from stdin and executes it with helpers pre-imported (`run.py:276-286`, `run.py:472-474`).
2. **Daemon plane** — one long-lived process per `BU_NAME`, holding the single CDP WebSocket to the browser and relaying IPC requests (`daemon.py:503-522`, `daemon.py:1276-1300`).
3. **Browser plane** — local Chrome found through `DevToolsActivePort`, an explicit `BU_CDP_WS`/`BU_CDP_URL`, or a Browser Use Cloud browser.

The three planes are deliberately separated: a caller crash does not drop the browser session, and many caller processes can share one daemon. In 0.1.15 the daemon also becomes a **tab broker**: it records which owner may drive which target/session and returns typed errors when the answer is "not yours".

## Module boundaries

- `_ipc.py` is the transport and identity vocabulary: owner resolution, socket/port paths, request framing, ping/identify. It does not know about tabs.
- `daemon.py` is the stateful broker: `Owner`, `Daemon`, attachment, event routing, idle sweep, captcha routing, shutdown. It is the only component that mutates `tab_owner`/`session_owner`.
- `admin.py` is the lifecycle supervisor: spawn lock, protocol compatibility, endpoint match, health probe, restart fingerprint, Chrome launch/approval, cloud start/stop.
- `helpers.py` is the public primitive surface seen by the agent: tabs, navigation, input, screenshots, JS, HTTP, agent-helper injection.
- `run.py` is the CLI adapter: stdin script, helper tracing, named-local-daemon→owner aliasing, telemetry wrapper.
- `report.py`, `telemetry.py`, `recorder.py`, `video*` are output surfaces with their own consent/redaction rules.
- `src/mcp_server.py` (top-level py-module, not under `browser_harness/`) is an optional MCP facade; `mcp_cli.py` imports it lazily and reports a missing extra.

## Core data structures

`Owner` (`daemon.py:487-501`): `tabs: {target_id: session_id|None}`, `current`, `events` (deque 500), `dialog`, `dialog_session`, `gone` (one-shot typed error), `seen`, `lock`, `name`, `marks`, `created`, `released`.

`Daemon` (`daemon.py:503-543`): `owners`, `tab_owner`, `session_owner`, `info`, `downloads`, `own_targets`, `browser_sessions`, captcha maps, `stop`, `last`, `keep`, `idle_timeout`, `idle_exit`.

`tab_owner` says one target has at most one owner. `session_owner` routes CDP events and explicit-session commands back to that owner. `browser_sessions` marks sessions created by `Target.attachToBrowserTarget`; they are intentionally browser-wide.

## Lifecycle

**Caller**: read stdin → optionally alias `BU_NAME` to an owner → ensure daemon (`require_existing_daemon` or spawn) → install helper trace → `exec(code, globals())` → captcha note → exit.

**Daemon**: load env → connect CDP (patient retry only for an immediate local 403) → install event tap → discover targets → load owners JSON → serve IPC + sweep + CDP watcher + optional owner-PID watcher → on stop, clean endpoint; cloud browser stop is attempted first.

**Tab**: unknown → `_current` creates a background `about:blank` (or adopts a spare blank only for cloud) → `_session` attaches once, enables Page/DOM/Runtime/Network + auto-attach → current tab streams Network events → idle release detaches (does not close by default) → `release_tabs` closes unmarked tabs the owner created, never the last real page.

## Configuration and state roots

- Identity: `BH_OWNER` > `CLAUDE_CODE_SESSION_ID` > `CODEX_THREAD_ID` > `"default"`.
- Naming: `BU_NAME` (default `default`); `BH_NAMED_LOCAL_DAEMON=1` opts out of local aliasing.
- Time: `BH_OWNER_IDLE` 1800 s; `BH_OWNER_FORGET` 86400 s; `BH_IDLE_TIMEOUT` 1800 s only when `BH_OWNS_BROWSER=1`; `BH_IDLE_EXIT` 1800 s for unused named local daemons.
- Paths: `BH_RUNTIME_DIR`/`BH_TMP_DIR`/`BH_HOME`/`BROWSER_HARNESS_HOME`/`BH_CONFIG_DIR`/`BH_AGENT_WORKSPACE`; new dirs `0700` on POSIX.
- Durable state: `bu-<name>.owners.json` in the runtime dir; `reports.json` for per-hour feedback caps; recordings under the config dir.
- Protocol: `PROTOCOL = 2`; a same-protocol client reuses the daemon whatever package version it is.

## Test system

16 unit files and 4 integration files. Ownership is the densest area: guarded raw Target calls, iframe reachability, URL/title hiding, idle takeover + old-session detach, release/forget behavior, last-page preservation, owner state across restart, stale-session non-redirection, and "only the current tab streams Network events" all have direct tests. Tests use a fake CDP browser; integration tests require a real browser and were not run here.

## Governance boundaries

- The target repository was read-only; no source edits, no install, no test run.
- The daemon's owner check is **cooperative** (caller-provided string), not authenticated.
- `GUARDED` covers three browser-level target methods; it does not cover `Target.attachToBrowserTarget` or browser-domain commands.
- **Fact vs doc**: README "Isolation and privacy hold for raw CDP too" is a documentation claim. The code supports tab-scoped isolation, not browser-session isolation.

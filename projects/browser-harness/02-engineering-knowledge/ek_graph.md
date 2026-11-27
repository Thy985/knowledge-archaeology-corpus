# 02 Engineering Knowledge — browser-harness 0.1.15

> 60 EK nodes. Layer is `engineering` throughout; abstraction L1/L2 only. Every entry has a file:line anchor.
> `links` use the six edge types in `contracts/ek-graph-schema.md`. "EV" is the evidence row in `06-validation/evidence.md`.

## A. Execution and architecture (EK-01–08)

### EK-01 · CLI executes a stdin heredoc in-process (L1)
- source: `run.py:276-286`, `run.py:472-474`
- detail: When stdin is not a TTY and no command args are given, `_read_task` reads the whole script and replaces `sys.stdin` with a `StringIO`; `_run` calls `exec(code, globals())` after helpers are imported. This is the "one script = one call" execution model; there is no framework loop.
- scope: CLI scripts; embedding callers can bypass this and call helpers directly.
- evidence: EV-01
- links: mechanism→EK-02; mechanism→EK-03; subsystem→EK-05
- confidence: high

### EK-02 · Helper tracing records at most 500 steps and never records args (L2)
- source: `run.py:202-245`, `run.py:248-260`
- detail: `_traced` increments a counter on every helper call and appends `{"helper": name}` only while `len(_helper_trace) < 500`; it records duration but not args/kwargs. Agent-defined helpers are renamed `custom`, so their own names are not leaked into telemetry. `recorder.observe` receives args separately for local recording.
- scope: CLI helper calls; direct `cdp()` bypasses the helper wrapper only if called from non-helper code (it is itself a helper, so it is traced).
- evidence: EV-02
- links: mechanism→EK-01; causal→EK-53; dependency→EK-54
- confidence: high

### EK-03 · Agent helpers are injected from one workspace file into the helper module (L2)
- source: `helpers.py:828-844`; `agent-workspace/agent_helpers.py` (7 lines)
- detail: At import time, `_load_agent_helpers()` loads `<BH_AGENT_WORKSPACE>/agent_helpers.py`, executes it, and copies every non-underscore public name into `helpers.globals()`. `_OWN_NAMES` is snapshotted **before** injection, so tracing can distinguish core helpers from agent-defined ones. Core package source is not edited by the agent.
- scope: only when the agent workspace file exists; a name collision overwrites the core helper in the module namespace.
- evidence: EV-03
- links: mechanism→EK-01; causal→EK-02; constraint→EK-07
- confidence: high

### EK-04 · The daemon is the long-lived CDP session authority (L2)
- source: `daemon.py:1`, `daemon.py:503-522`, `daemon.py:1276-1300`
- detail: The daemon owns one `CDPClient`, one WebSocket, event tap, target info map and owners map. `serve()` runs IPC, a sweeper, a CDP watcher and an optional owner-PID watcher concurrently; when the CDP reader task ends, the daemon stops instead of leaving a stale connection.
- scope: one daemon process; callers are stateless between invocations.
- evidence: EV-04
- links: subsystem→EK-05; dependency→EK-08; causal→EK-50
- confidence: high

### EK-05 · Each CLI call speaks to the daemon over a one-shot IPC request (L2)
- source: `helpers.py:41`, `helpers.py:75-88`, `_ipc.py:130-150`
- detail: `OWNER = ipc.owner()` is captured at import. `_send` opens a new socket, sends one JSON line with `owner`, waits for one JSON line, closes. Request type is inferred: `meta` for broker operations, `method` for CDP. There is no persistent caller connection.
- scope: all helpers; `admin` uses the same transport for health/lifecycle.
- evidence: EV-05
- links: subsystem→EK-04; dependency→EK-39/EK-40; mechanism→EK-09
- confidence: high

### EK-06 · Protocol version 2 is the only compatibility contract (L2)
- source: `_ipc.py:47`, `admin.py:422-446`
- detail: `PROTOCOL = 2` changes only when the request format changes. A client with the same protocol reuses any running daemon regardless of package version. An older daemon may be restarted only under narrow conditions; a newer daemon is never stopped by an older client.
- scope: client/daemon compatibility; not a security boundary.
- evidence: EV-06
- links: constraint→EK-34; mechanism→EK-33
- confidence: high

### EK-07 · One daemon per `BU_NAME`; one browser connection per daemon (L1)
- source: `daemon.py:1`, `daemon.py:503-505`, `_ipc.py:68-78`
- detail: Daemon files are `bu-<name>.sock/.pid/.log/.owners.json`. The docstring states "One daemon per BU_NAME" and "One browser connection shared by many owners". Different names can point at the same local Chrome, producing more than one Chrome Allow prompt; `run.py` aliases a local `BU_NAME` to an owner only when no daemon of that name exists.
- scope: local Chrome aliasing is disabled by `BH_NAMED_LOCAL_DAEMON=1`; remote/CDP names are never aliased.
- evidence: EV-07
- links: subsystem→EK-08; constraint→EK-33; contrast→EK-09
- confidence: high

### EK-08 · The daemon does not reconnect its CDP socket (L2)
- source: `daemon.py:1350-1360`
- detail: `_watch_cdp` waits on the client's message-handler task; when it ends, the daemon logs the close code/reason and sets `stop`. There is no reconnect loop. A dead browser therefore ends the daemon, and the next caller starts a new generation.
- scope: all browser kinds; cloud death also ends billing session.
- evidence: EV-08
- links: causal→EK-04; contrast→EK-38
- confidence: high

## B. Owner identity and tab authority (EK-09–19)

### EK-09 · Owner identity is an environment-variable fallback chain (L1)
- source: `_ipc.py:34-39`
- detail: `owner()` returns `BH_OWNER`, else `CLAUDE_CODE_SESSION_ID`, else `CODEX_THREAD_ID`, else `"default"`. The docstring explicitly warns: "Sub-agents inherit their parent's session id, so they need their own BH_OWNER."
- scope: identity label, not authentication; any caller can set the variable.
- evidence: EV-09
- links: mechanism→EK-05; causal→EK-10; contrast→EK-12
- confidence: high

### EK-10 · A Claude sub-agent without `BH_OWNER` is its parent's owner (L2)
- source: `helpers.py:109-118`; `_ipc.py:36-38`
- detail: `_shares_parents_owner()` is true when `BH_OWNER` is unset and `CLAUDE_CODE_CHILD_SESSION == "1"`. In that state the sub-agent inherits the parent's session id and therefore the same owner key. `_warn_shared_owner` is written as an unconditional stderr warning with no once-flag, but its only production wiring is `run.py:471`, once at CLI startup; the helper layer does not warn per call.
- scope: Claude-specific marker; other hosts may not set it, in which case the fallback is default or host session id.
- evidence: EV-10
- links: causal→EK-09; constraint→EK-11; contrast→EK-13
- confidence: high

### EK-11 · `owner-shared` refuses only `close_tab` and `release_tabs` (L2)
- source: `helpers.py:114-118`, `helpers.py:441-445`, `helpers.py:521-525`
- detail: The refusal is called in exactly two public helpers. It does **not** wrap `cdp()`, `goto_url()`, `js()`, `click_at_xy()`, `switch_tab()`, or any other tab-driving primitive. A sub-agent without `BH_OWNER` can still act on the parent's current tab through those paths; it just cannot close or release the parent's tabs.
- scope: cooperative guard; bypassed by setting `BH_OWNER`, by calling `cdp()` directly, or by any non-Claude host without the child marker.
- evidence: EV-11
- links: constraint→EK-10; contrast→EK-14; dependency→EK-18
- confidence: high

### EK-12 · A target has at most one owner in the daemon (L2)
- source: `daemon.py:503-522`, `daemon.py:608-617`
- detail: `Daemon` docstring: "A tab has at most one owner." `_own(key, tid)` writes `tab_owner[tid] = key` and adds the tab to `Owner.tabs`; `_drop` pops the target from `tab_owner`, `Owner.tabs`, `session_owner`, marks and created.
- scope: in-process broker only; browser-level sessions and other OS processes are outside this map.
- evidence: EV-12
- links: mechanism→EK-13; causal→EK-14; contrast→EK-19
- confidence: high

### EK-13 · `Owner` is the per-agent tab/session/event record (L1)
- source: `daemon.py:487-501`
- detail: Fields: `tabs` (target→session or None), `current`, `events` (deque maxlen 500), `dialog`, `dialog_session`, `gone`, `seen`, `lock`, `name`, `marks`, `created`, `released`.
- scope: one owner key; owners are created lazily by `Daemon.owner(key)`.
- evidence: EV-13
- links: subsystem→EK-12; causal→EK-24/EK-27
- confidence: high

### EK-14 · Three browser-level target methods are guarded by ownership (L2)
- source: `daemon.py:37`, `daemon.py:1210-1219`
- detail: `GUARDED = ("Target.attachToTarget", "Target.closeTarget", "Target.activateTarget")`. In `_send`, any `Target.*` request is handled at browser level; if the method is guarded and `_holder(targetId)` is neither None nor the caller's key, it returns `tab-taken` without sending. `_holder` returns None when the recorded owner has been idle longer than `OWNER_IDLE`, which is the deliberate takeover path.
- scope: these three methods only. `Target.attachToBrowserTarget` and other Browser-domain methods are not in the tuple.
- evidence: EV-14
- links: constraint→EK-12; contrast→EK-19; dependency→EK-15
- confidence: high

### EK-15 · An iframe is attachable only when its containing tab is provable (L2)
- source: `daemon.py:1216-1219`, `daemon.py:560-576`
- detail: For `Target.attachToTarget` on an `iframe`, `_frame_tab` first climbs `parentFrameId` to the root. If the root is a known tab, it must equal the caller's key. If no parent is known, the owner's current tab must answer `DOM.getFrameOwner` for that frame. Failure returns `frame-not-yours`; an iframe is never recorded as a tab.
- scope: protects cross-origin iframe attachment; same-process frames with no parent rely on the current-tab proof.
- evidence: EV-15
- links: constraint→EK-14; dependency→EK-12; contrast→EK-16
- confidence: high

### EK-16 · Another owner's tab is URL/title-blanked at the daemon, not the client (L2)
- source: `daemon.py:548-556`; tests `test_another_agents_tab_shows_no_url_or_title_even_to_raw_cdp` (`test_daemon.py:533-556`)
- detail: `_visible` keeps keys (`targetId`, `type`, `parentFrameId`) but empties `url` and `title` when the target is an iframe not rooted in the caller's tab, or a non-iframe whose holder is neither None nor the caller. `list_tabs`, raw `Target.getTargets` and `Target.getTargetInfo` all pass through `_visible`. The test explicitly checks the filter sees empty values even through raw CDP, while the owner sees the real URL.
- scope: page and target-info metadata; not page content, cookies, or a browser-level session.
- evidence: EV-16
- links: mechanism→EK-14; contrast→EK-19; dependency→EK-17
- confidence: high

### EK-17 · Session→owner routing is the event and command gate (L2)
- source: `daemon.py:817-831`, `daemon.py:1219-1223`
- detail: `_route_event` looks up `session_owner[session_id]`; an auto-attached child (iframe/worker) inherits the parent's key. `_send` with an explicit session rejects the request if `session_owner.get(sid, key) != key`. Only the current tab's session has Network enabled.
- scope: sessions that were registered by attach/auto-attach.
- evidence: EV-17
- links: mechanism→EK-12; constraint→EK-18; contrast→EK-19
- confidence: high

### EK-18 · A command without a session is pinned to the owner's current tab (L2)
- source: `daemon.py:1219-1245`
- detail: For non-`Target` methods with no `sid`, the daemon calls `_current(key, o)` then `_session(key, o, tid)` and sends on that session. `Page.navigate`/`Page.reload` clear the captcha finish for that tab. On "Session with given id not found", it re-attaches **the same tab once** and retries; it never redirects to another tab.
- scope: all normal helpers; a caller wanting another own tab must `switch_tab` or pass an explicit own session.
- evidence: EV-18
- links: dependency→EK-17; causal→EK-29; contrast→EK-19
- confidence: high

### EK-19 · Browser-level CDP sessions are not owner-bound (L2) — isolation gap
- source: `daemon.py:1226-1228`, `daemon.py:1219-1223`, `daemon.py:503-522`
- detail: `Target.attachToBrowserTarget` is not in `GUARDED`; its returned session id is added to `self.browser_sessions`, **not** to `session_owner`. Later, an explicit session request checks `session_owner.get(sid, key) != key`; for an unregistered browser session this defaults to the caller's key, so the check passes for **any** owner that knows the session id. Browser-domain methods sent with that sid (e.g. `Storage.getCookies`, `Network.getAllCookies`) then execute browser-wide, outside tab ownership. The session id is unguessable, but it is a capability, not an owner check.
- scope: affects "Isolation and privacy hold for raw CDP too"; tab-routed raw CDP remains protected.
- evidence: EV-19
- links: contrast→EK-14/EK-16/EK-17; constraint→EK-56
- confidence: high (code path); exploitation requires the sid

## C. Tab lifecycle and reclamation (EK-20–30)

### EK-20 · A fresh owner gets its own background tab, never the person's (L2)
- source: `daemon.py:662-671`; tests `test_a_fresh_owner_gets_its_own_background_tab` (`test_daemon.py:305-325`)
- detail: `_current` creates `Target.createTarget` with `url="about:blank", background=True` when the owner has no current tab. The only adoption path is `REMOTE_ID` (cloud) and an unowned blank page; it never adopts the user's current tab.
- scope: local and cloud; cloud may adopt a spare blank because the live view shows the first tab.
- evidence: EV-20
- links: mechanism→EK-13; contrast→EK-21; causal→EK-22
- confidence: high

### EK-21 · Cloud may adopt one unowned blank tab; local may not (L2)
- source: `daemon.py:666-671`
- detail: `spare = [t for t ... if REMOTE_ID and t not in tab_owner and type page and is_blank(url)]`. If a spare exists it is used and `created=False`; otherwise a new target is created. This avoids wasting a cloud tab and keeps the visible live view on that tab.
- scope: `REMOTE_ID` only (`BU_BROWSER_ID` set).
- evidence: EV-21
- links: contrast→EK-20; constraint→EK-26
- confidence: high

### EK-22 · `new_tab(url)` reuses a blank current tab and otherwise creates one (L1)
- source: `daemon.py:1192-1197`, `helpers.py:510-518`
- detail: The broker reuses `o.current` only when `reuse` is true and the current URL is blank; otherwise creates a background target, owns it with `created=not reuse`, and switches to it. The helper then calls `goto_url` rather than passing the URL to `createTarget`, because create-then-attach races with load.
- scope: `new_tab("about:blank")` never reuses; `new_tab(url)` may.
- evidence: EV-22
- links: causal→EK-20; mechanism→EK-18
- confidence: high

### EK-23 · A popup is owned by its opener's owner (L2)
- source: `daemon.py:847-855`
- detail: On `Target.targetCreated`, if `openerId` is owned and the new target is a page, `_own(opener, tid, current=False)` assigns it to the opener's owner. The comment notes the person may have opened it; the tab is owned but not made current.
- scope: targetCreated events Chrome emits with `openerId`; popups without an opener stay unowned until switched.
- evidence: EV-23
- links: causal→EK-12; contrast→EK-20
- confidence: high

### EK-24 · Idle release keeps the current tab, detaches the rest (L2)
- source: `daemon.py:726-750`; test `test_idle_release_gives_tabs_back_and_closes_nothing` (`test_daemon.py:970-987`)
- detail: `_release(key,o,"idle")` skips `tid == o.current`; for other tabs it detaches the session, clears the marker for unmarked tabs, calls `_drop(tid,"idle")`, and records `o.released.add(tid)` so a returning owner gets `tab-released`. Default `BH_CLOSE_IDLE_TABS` unset means **nothing is closed** on idle, including tabs the owner created. Current tab and its session stay owned.
- scope: owner idle > `OWNER_IDLE` (1800 s). Marked tabs keep their title.
- evidence: EV-24
- links: mechanism→EK-25; causal→EK-13; contrast→EK-26
- confidence: high

### EK-25 · After `OWNER_FORGET`, even the last tab is given back (L2)
- source: `daemon.py:752-766`; test `test_after_a_day_idle_the_last_tab_is_given_back_too` (`test_daemon.py:1071-1085`)
- detail: `sweep` computes idle; over `OWNER_IDLE` it releases non-current tabs; over `OWNER_FORGET` it calls `_release(...,"forget")`, which does not skip the current tab. Nothing is closed. Idle owners with no tabs are deleted, except a short window preserving `gone` for a late returner.
- scope: 24 h default; tab remains open, owner records mostly gone.
- evidence: EV-25
- links: causal→EK-24; contrast→EK-26
- confidence: high

### EK-26 · `release_tabs` closes only unmarked tabs the owner created, never the last real page (L2)
- source: `daemon.py:726-750`; tests `test_release_tabs_closes_what_the_agent_opened_and_leaves_the_rest` (`test_daemon.py:1016-1028`), `test_idle_release_never_closes_the_browsers_last_real_page` (`:1182-1193`)
- detail: `close = how == "done" or (BH_CLOSE_IDLE_TABS == "1" and tid != current)`. A tab is closed only if `close and tid in o.created and tid not in o.marks and not last`, where `last` means at most one real page remains across the browser. Marked tabs are never closed; a tab taken from the person was not `created`, so it is detached, not closed.
- scope: `release_tabs` and opt-in idle close; default idle never closes.
- evidence: EV-26
- links: contrast→EK-24; dependency→EK-27; constraint→EK-21
- confidence: high

### EK-27 · Marks are the only persistent "leave this for the person" signal (L1)
- source: `daemon.py:151`, `daemon.py:1124-1134`; `helpers.py:455-460`
- detail: `MARKS = {"deliverable": 📌, "handoff": ✋}`. `mark_tab` accepts only these or None, validates `_mine`, stores in `o.marks`, and re-renders the title marker. `_release` preserves marked tabs; `_save` persists marks so they survive daemon restart.
- scope: title marker is cosmetic; marks do not grant access control beyond "do not close".
- evidence: EV-27
- links: causal→EK-26; mechanism→EK-30; dependency→EK-47
- confidence: high

### EK-28 · `close_tab` is ownership-checked and returns the closed id (L2)
- source: `daemon.py:1136-1145`; test `test_close_tab_closes_only_your_tabs` (`test_daemon.py:557-568`)
- detail: `_mine` returns `tab-taken` for another holder, `tab-released` for a tab this owner lost while idle, or "not yours" for a free person tab. Only then is `Target.closeTarget` sent and `_drop` called. The response is `{"closed": tid}`.
- scope: current tab by default; a person's free tab must be `switch_tab`ed first.
- evidence: EV-28
- links: dependency→EK-12; contrast→EK-26
- confidence: high

### EK-29 · A stale session is re-attached to the same tab exactly once (L2)
- source: `daemon.py:1230-1245`; tests `test_stale_session_never_replays_onto_another_tab` (`test_daemon.py:784-797`), `test_stale_session_reattaches_the_same_tab_once` (`:799-810`)
- detail: On "Session with given id not found", the daemon logs, sets `o.tabs[tid] = None` under the owner lock if it still matches, calls `_session` again, then retries the same command. `_session` raises `tab-closed` if the tab disappeared; it never falls back to another tab.
- scope: transport-level staleness after navigation/Chrome restart; not a license to switch tabs.
- evidence: EV-29
- links: causal→EK-18; contrast→EK-19
- confidence: high

### EK-30 · Tab title markers identify owner and mark, but not session ids (L2)
- source: `daemon.py:156-180`, `daemon.py:712-724`
- detail: `_mark` writes `🐴<label><mark> ` into `document.title` via `Runtime.evaluate`, fire-and-forget. Label is `o.name` or a sanitized owner key; a long hex session-like key is suppressed entirely ("A session id is no name"). The JS strips any earlier horse marker first. `BH_TAB_MARKER=0` disables it.
- scope: cosmetic UI; page scripts can read/modify the title; marker removal on release restores a clean title only for unmarked tabs.
- evidence: EV-30
- links: mechanism→EK-27; contrast→EK-16
- confidence: high

## D. Daemon competition and identity (EK-31–38)

### EK-31 · Liveness requires a `{pong:true}` dict, not a bare connect (L2)
- source: `admin.py:391-411`, `_ipc.py:151-170`
- detail: `ipc.ping` treats any non-dict or non-`pong:True` response as not our daemon. `daemon_alive` uses ping. The `ping` reply includes `pid`, `browser_kind`, `endpoint`, `protocol`, `active`, `cdp` label/keys.
- scope: detects stale port reuse / hostile listener; not proof of same process generation beyond pid.
- evidence: EV-31
- links: mechanism→EK-32; dependency→EK-33
- confidence: high

### EK-32 · The ready daemon's PID is type-checked and range-checked before signaling (L2)
- source: `_ipc.py:173-210`; `admin.py:904-1040`
- detail: `identify()` accepts only `type(pid) is int` (explicitly rejects bool), `0 < pid < 2**31`, from a `pong:True` dict. `restart_daemon` also snapshots process start time and verifies the daemon identity before SIGTERM; a stale PID file is never trusted.
- scope: ready daemons with an IPC socket. Pending handshake daemons use a different fingerprint path (EK-38).
- evidence: EV-32
- links: dependency→EK-31; contrast→EK-38
- confidence: high

### EK-33 · One spawn lock per daemon name prevents two cold daemons (L2)
- source: `admin.py:279-336`, `admin.py:700-720`
- detail: `_spawn_lock` is an advisory `flock`/`msvcrt` lock on `bu-<name>.spawnlock`, mode 0600. Callers wait up to `startup_wait`; if the lock cannot be acquired, `ensure_daemon` raises `daemon-starting` instead of spawning. Inside the lock, the caller re-checks liveness, sweeps dead endpoints, and reuses a parked/starting pid before `Popen`.
- scope: same daemon name; different names can still spawn concurrently and produce multiple Chrome prompts.
- evidence: EV-33
- links: mechanism→EK-07; dependency→EK-31; contrast→EK-07 (local alias)
- confidence: high

### EK-34 · A newer-protocol daemon is never stopped by an older client (L2)
- source: `admin.py:422-446`
- detail: If ping protocol > client protocol, `_restart_if_outdated` raises `daemon-outdated` with "it is newer than this client" and the update instruction. It also refuses to restart when `may_restart` is false (provisioned), when the daemon is cloud/CDP (would lose its browser), or when `active` owners exist. Only an older local daemon with no active owners may be restarted.
- scope: protocol mismatch only; same protocol ignores package version.
- evidence: EV-34
- links: constraint→EK-06; contrast→EK-35
- confidence: high

### EK-35 · Active owners block an automatic restart (L2)
- source: `daemon.py:1079-1092`, `admin.py:422-446`
- detail: Ping's `active` counts owners other than the pinging owner with at least one tab and `seen` within `OWNER_IDLE`. The restart path refuses when `active` is nonzero so a protocol bump cannot pull a daemon out from under other agents.
- scope: only in-flight owners with tabs; idle owners with no tabs do not block.
- evidence: EV-35
- links: dependency→EK-34; contrast→EK-25
- confidence: high

### EK-36 · A name already bound to another CDP endpoint fails with `cdp-mismatch` (L2)
- source: `admin.py:587-620`, `daemon.py:286-300`
- detail: Ping returns a redacted endpoint label plus hashes of the CDP URLs. `_check_cdp_endpoint` compares the caller's desired endpoint key against the daemon's advertised keys; mismatch raises `cdp-mismatch` and tells the caller to use another `BU_NAME` or reload. In a cold race, the loser gets `cdp-mismatch` rather than stealing the socket.
- scope: explicit `BU_CDP_URL`/`BU_CDP_WS`; local Chrome without an explicit endpoint is not compared.
- evidence: EV-36
- links: constraint→EK-07; mechanism→EK-33
- confidence: high

### EK-37 · A remote-browser marker makes local fallback fail closed (L2)
- source: `_ipc.py:80-105`, `admin.py:760-790`
- detail: The daemon writes `bu-<name>.remote` on connect for non-local browsers and removes it only on explicit shutdown. `ensure_daemon` in local mode checks the marker and raises `remote-browser-ended` instead of silently attaching local Chrome. Only an explicit `start_remote_daemon`/`restart_daemon` can clear the name.
- scope: local mode; CDP/cloud names.
- evidence: EV-37
- links: constraint→EK-36; causal→EK-08
- confidence: high

### EK-38 · Pending handshake daemons are killed only by matching a process-start fingerprint (L2)
- source: `admin.py:192-278`, `admin.py:904-1040`
- detail: When a daemon is parked at `handshake-wait` and has no IPC socket, `restart_daemon` reads an atomically published pid record containing process start time; it re-reads under the spawn lock and refuses if the generation changed, if the daemon became ready, or if no verifiable fingerprint exists. It then SIGTERMs the verified pid and cleans the endpoint.
- scope: local Chrome approval window; ready daemons use EK-32.
- evidence: EV-38
- links: contrast→EK-32; mechanism→EK-33
- confidence: high

## E. Transport and privacy (EK-39–43)

### EK-39 · POSIX IPC is a 0600 AF_UNIX socket created under umask (L2)
- source: `_ipc.py:246-262`
- detail: `serve` unlinks a stale path, sets `umask(0o077)`, binds `asyncio.start_unix_server` (so the socket is created 0600 with no TOCTOU chmod), restores umask, and waits forever. Runtime dir is also created 0700 (`paths.py:19-23`).
- scope: same-user processes can still connect; file mode is not an authorization check.
- evidence: EV-39
- links: mechanism→EK-05; contrast→EK-40
- confidence: high

### EK-40 · Windows IPC is TCP loopback plus a per-daemon 64-hex token (L2)
- source: `_ipc.py:263-278`, `daemon.py:1080-1086`
- detail: The server binds `127.0.0.1:0`, generates `secrets.token_hex(32)`, writes `{"port","token"}` atomically to `bu-<name>.port`, and requires the token on every request. The token is a capability; the port file is mode 0600? (not explicitly chmoded in `_ipc.serve`).
- scope: Windows only; POSIX expects `expected_token()` is None.
- evidence: EV-40
- links: contrast→EK-39; dependency→EK-41
- confidence: high

### EK-41 · One IPC request is capped at 32 MiB (L2)
- source: `_ipc.py:39-44`, `_ipc.py:212-230`
- detail: `MAX_REQUEST_BYTES = 32 << 20`; `read_request` raises `RequestTooLarge` after draining to newline, and the handler returns an error. The comment sizes it for a 10 MB file as base64 inside `js()` plus JSON framing.
- scope: request line; responses have no equivalent cap in this file.
- evidence: EV-41
- links: constraint→EK-05; mechanism→EK-40
- confidence: high

### EK-42 · Only the owner's current tab has Network enabled (L2)
- source: `daemon.py:694-710`; test `test_only_the_current_tab_streams_network_events` (`test_daemon.py:1205-1219`)
- detail: `_switch` disables Network on the old current session and enables it on the new one **in the same gather round** as attach. On failure it re-enables the old session. A background tab's network events cannot push the current tab's events out of the 500-event ring.
- scope: Network domain only; Page/DOM/Runtime stay enabled on all owned sessions.
- evidence: EV-42
- links: mechanism→EK-18; contrast→EK-43
- confidence: high

### EK-43 · Browser-level download events are routed by frame or download guid (L2)
- source: `daemon.py:826-838`
- detail: Events without a session (browser-level, e.g. downloads) resolve the owner from `downloads[guid]` or `tab_owner[frameId]`; the first event records the guid→owner and completion removes it. Unknown owners are dropped, not broadcast.
- scope: downloads that include `frameId` or `guid`; an event with neither is not routed.
- evidence: EV-43
- links: dependency→EK-17; contrast→EK-19
- confidence: medium (initial guid route depends on Chrome's event shape)

## F. Failure and lifecycle edges (EK-44–52)

### EK-44 · A crashed tab produces a one-shot `tab-crashed` error (L2)
- source: `daemon.py:853-858`
- detail: On `Target.targetCrashed`, if the target has an owner and is current, `o.gone` is set to `tab-crashed: ... Nothing was sent. Call goto_url(...) or new_tab(...)`. `_meta` returns it once, then clears it.
- scope: current tab; non-current crashed tabs are not specially handled.
- evidence: EV-44
- links: causal→EK-45; contrast→EK-29
- confidence: high

### EK-45 · Target destruction removes ownership, session and marks (L1)
- source: `daemon.py:619-633`, `daemon.py:847-852`
- detail: `Target.targetDestroyed` calls `_drop(tid)` and removes `info`; `_drop` pops `tab_owner`, `Owner.tabs`, `session_owner`, `o.marks`, `o.created`, and sets `o.gone` if it was current.
- scope: any target destruction; iframe/worker destruction also drops its session mapping.
- evidence: EV-45
- links: causal→EK-44; mechanism→EK-12
- confidence: high

### EK-46 · Captcha solve state is routed per owning tab (L2)
- source: `daemon.py:867-909`, `daemon.py:912-1000`
- detail: `_share_captcha_event` sends a solve on an owned target only to that tab's owner; a solve on an unowned tab or a solver state change goes to every owner. `captcha()` filters finished/active solves by `_captcha_mine(key, tid)`, where an empty target id matches all. A solve's finish is returned to one caller (`captcha_taken`), with a short `CAPTCHA_FRESH_S` window.
- scope: Browser Use Cloud solver events; local returns `solver unavailable`.
- evidence: EV-46
- links: contrast→EK-19; dependency→EK-17
- confidence: high

### EK-47 · Owner/tab state is persisted to JSON and restored across restarts (L2)
- source: `daemon.py:635-660`, `_ipc.py:73-75`
- detail: `_save` writes `{owner: {tabs, current, name, marks, created}}` to `bu-<name>.owners.json` via tmp+replace; `_load` restores only targets that still exist as pages, and marks whose value is in `MARKS`. Tabs not in the current page set are dropped.
- scope: same `BU_NAME` and same browser; a different browser/name gets a different file.
- evidence: EV-47
- links: mechanism→EK-13; causal→EK-27; contrast→EK-48
- confidence: high

### EK-48 · A damaged owners file never blocks daemon startup (L2)
- source: `daemon.py:645-660`
- detail: `_load` catches `FileNotFoundError` silently and catches any other exception with `log("load owners: ...")`; it does not re-raise. The daemon starts with fresh ownership rather than failing.
- scope: state corruption/partial write; owner history is lost, not recovered.
- evidence: EV-48
- links: contrast→EK-47; mechanism→EK-04
- confidence: high

### EK-49 · Idle exit is opt-in and scoped by browser kind (L2)
- source: `daemon.py:164-176`, `daemon.py:752-770`
- detail: `BH_IDLE_TIMEOUT` (default 1800 s) applies only when `BH_OWNS_BROWSER=1` (a browser this daemon started); otherwise 0 = never exit. `sweep` can exit an unused named local daemon after `IDLE_EXIT` (1800 s) or a CDP daemon only if `BH_IDLE_EXIT` is set; default local and cloud never exit this way. `BH_REQUIRE_EXISTING_DAEMON` sets `keep` and prevents exit.
- scope: local named daemons and explicitly owned browsers; the default local daemon stays for the user's Chrome.
- evidence: EV-49
- links: constraint→EK-07; contrast→EK-25; dependency→EK-52
- confidence: high

### EK-50 · A CDP disconnect stops the daemon; there is no reconnect loop (L2)
- source: `daemon.py:1350-1360`
- detail: `_watch_cdp` awaits the CDP message-handler task; when it completes (going away, timeout, browser death), it logs and `d.stop.set()`. `main` then cancels tasks and cleans the endpoint. The next caller spawns a new generation.
- scope: all browser kinds; cloud browser lifecycle ends with daemon.
- evidence: EV-50
- links: causal→EK-08; contrast→EK-38; dependency→EK-49
- confidence: high

### EK-51 · `BH_OWNER_PID` is an opt-in daemon lifetime tie (L2)
- source: `daemon.py:1301-1341`
- detail: If set to a positive int, `serve` starts `_watch_owner`, which polls `_pid_alive` every second and sets `stop` when the owner exits. On Windows it uses `OpenProcess(SYNCHRONIZE)`/`WaitForSingleObject`; on POSIX `os.kill(pid,0)`, treating `PermissionError` as alive.
- scope: opt-in; not set by default; does not cover the whole process tree.
- evidence: EV-51
- links: constraint→EK-49; mechanism→EK-04
- confidence: high

### EK-52 · SIGTERM/SIGHUP run the normal shutdown, so cloud cleanup happens (L1)
- source: `daemon.py:1362-1378`, `daemon.py:1094-1106`
- detail: Signal handlers remove themselves and set `d.stop`; normal shutdown attempts `stop_remote(strict=True)` before clearing the remote marker. A second signal falls back to default. `main` closes an owned browser only if no other CDP clients remain attached.
- scope: POSIX; Windows uses default signal behavior. Cloud stop failure keeps the daemon so a retry can stop billing.
- evidence: EV-52
- links: causal→EK-50; dependency→EK-37
- confidence: high

## G. Output surfaces and project identity (EK-53–56)

### EK-53 · Report texts are scrubbed before leaving the process (L2)
- source: `report.py:45-160`
- detail: `scrub` replaces env values whose key ends in key/token/secret/password/credentials, API-key shapes, bearer/basic tokens, cookies, `password=`-style assignments, URL credentials/ports/queries/fragments, path tokens, home usernames, emails, 6+ digit runs, and `js()` expressions. It is used for `library_feedback` and `library_judgement`, with 300/2000-character caps.
- scope: report/feedback text only; not the telemetry event's arbitrary properties.
- evidence: EV-53
- links: mechanism→EK-54; contrast→EK-55
- confidence: high

### EK-54 · Telemetry properties are key-filtered and URL-values redacted (L2)
- source: `telemetry.py:22-40`, `telemetry.py:149-163`
- detail: `_safe_properties` sanitizes keys to `[A-Za-z0-9_$.-]`, drops keys containing forbidden words, keeps bool/null/numbers, and replaces any string containing `://` with `[redacted]`; other strings are truncated to 120 chars. This is a key/shape filter, not a general text scrubber.
- scope: usage telemetry; report.py has its own stronger scrubber.
- evidence: EV-54
- links: contrast→EK-53; dependency→EK-02
- confidence: high

### EK-55 · `http_get` keeps private hosts and credentialed requests local (L2)
- source: `helpers.py:771-820`
- detail: With `BROWSER_USE_API_KEY`, public URLs go through fetch-use; `_proxy_can_reach` excludes localhost/private/intranet hosts and requests with `Authorization`/`Cookie` headers, which use urllib locally. Binary bodies return bytes, text returns str.
- scope: HTTP only; browser navigation is unaffected. A user-supplied URL hostname is the trust input, so DNS rebinding is outside this check.
- evidence: EV-55
- links: contrast→EK-53; subsystem→EK-03
- confidence: high

### EK-56 · Raw CDP availability is documented; browser-domain CDP has no owner binding (L1)
- source: `SKILL.md:52`, `daemon.py:1220-1228`, `daemon.py:1219-1223`, `daemon.py:503-522`
- detail: `SKILL.md:52` says "Raw CDP is available with `cdp(...)`" — availability, not isolation. The code enforces tab ownership for `Target.attachToTarget`/`closeTarget`/`activateTarget`, iframe proof, visibility blanking, and session-owner routing. It does **not** bind `Target.attachToBrowserTarget` sessions to an owner: the sid goes only to `browser_sessions`, and an unregistered sid passes the default `session_owner.get(sid, key)` check. Browser-domain methods are not tab-scoped. A quoted README sentence "Isolation and privacy hold for raw CDP too" is **absent from this snapshot** (137-line README, grep zero), so it must not be cited; the code gap stands independently.
- scope: browser-level CDP sessions; tab-routed raw CDP remains protected.
- evidence: EV-56
- links: contrast→EK-19; constraint→EK-14/EK-16
- confidence: high

### EK-57 · Package/CLI identity and Skill identity diverge (L1)
- source: `AGENTS.md` ("Skill identity (name + trigger) = browser-use (do not rename)"), `SKILL.md:2` (`name: browser-harness`), `pyproject.toml` (`browser-harness`)
- detail: The package and CLI are `browser-harness`; the skill identity that should trigger is `browser-use`; the shipped `SKILL.md` frontmatter still says `browser-harness`. The 0.1.13 baseline recorded the same conflict. The README now says Browser Harness is the engine inside Browser Use and recommends the `browser-use` CLI, which explains the product rename but not the frontmatter mismatch.
- scope: distribution/discovery identity; no runtime behavior.
- evidence: EV-57
- links: contrast→EK-58
- confidence: high

### EK-58 · browser-use 1.0.8 pins 0.1.13 and does not consume the owner API (L1, cross-repo)
- source: `browser-use` 1.0.8 `pyproject.toml:19` (`browser-harness==0.1.13`); `browser_use/browser/browser.py:900-940` (unique `bu<8hex>` per non-shared browser, shared default daemon for `Browser('chrome')`); source-wide grep finds no `BH_OWNER`/`release_tabs`/`list_browsers`/`use_browser`/`name_session`/`mark_tab`
- detail: The downstream solves concurrency by giving each non-shared `Browser` its own daemon and browser; the shared `Browser('chrome')` path uses the default daemon with no owner key. Therefore 0.1.15's owner model is not used by 1.0.8, and the shared-Chrome path still relies on one daemon's serialized current tab. The 0.1.14 changelog claim "many agents, one Chrome" is upstream capability, not downstream adoption.
- scope: browser-use 1.0.8 only; future versions may adopt it.
- evidence: EV-58
- links: contrast→EK-07; causal→EK-09; contrast→EK-19
- confidence: high for the pin and grep; medium for the concurrency consequence (inferred from code, not a runtime test)

### EK-59 · 0.1.15 trims CLI telemetry payloads without a changelog entry (L1)
- source: `run.py:207-237`, `run.py:323-356`, `telemetry.py:282-315`, `helpers.py:843`; diff against 0.1.14 wheel (`audit/_scratch/bh0114`)
- detail: Relative to 0.1.14, `cli_event` no longer sends the script text (`task`) or stdout/stderr tails (`output`), only `task_length`/`output_length`. Trace steps no longer record args or exception text; they record helper name, duration, and a boolean error flag. Error class is the nearest builtin in the MRO; `SystemExit` is recorded as `"SystemExit"`, not an exit code or stderr tail. Helper names from `agent_helpers.py` are classified `"custom"` using the `_OWN_NAMES` snapshot.
- scope: CLI telemetry only; `report_issue`/`report_run` are separate surfaces (EK-53). `CHANGELOG.md` has no 0.1.15 section, so this delta is release evidence from the wheel diff, not a changelog claim.
- evidence: EV-59
- links: contrast→EK-02; dependency→EK-53; contrast→EK-54
- confidence: high (diff), medium for intent (no changelog text)

### EK-60 · The owner-state file has no explicit file-mode tightening (L1)
- source: `daemon.py:635-643`, `_ipc.py:73-75`, `paths.py:19-23`
- detail: `bu-<name>.owners.json` stores target ids, current tab, marks, created set and owner name, written via tmp+`os.replace`. The runtime directory is created 0700, but `_save` does not set `0600` on the final file explicitly. A same-user process can read or write it. The data is handles/booleans, not cookies or page content.
- scope: same OS user; cross-user access blocked by directory mode. Combined with C-01, target ids are handles but not credentials.
- evidence: EV-60
- links: contrast→EK-47; contrast→EK-19; dependency→EK-39
- confidence: high (code); file mode not measured at runtime

# 04 Flow Atlas — browser-harness 0.1.15

> Edges are anchored to real symbols. "role" is the function in the chain, not a claim about intent.
> Evidence rows are in `06-validation/evidence.md`.

## F-01 Control — script call to CDP command

Question: who decides that a command goes to a tab, and where can it stop?

| node | symbol | role |
|---|---|---|
| stdin read | `_read_task` `run.py:276-286` | input |
| helper execution | `exec(code, globals())` `run.py:472-474` | dispatch |
| helper call | `cdp()` `helpers.py:96-107` | request construction |
| IPC send | `_send` `helpers.py:75-88` → `ipc.request` `_ipc.py:130-150` | transport |
| broker entry | `Daemon.handle` `daemon.py:1070-1096` | admission |
| meta/target route | `Daemon._meta` `daemon.py:1098-1208` | branch |
| raw CDP route | `Daemon._send` `daemon.py:1210-1245` | guard + session selection |
| CDP send | `self.cdp.send_raw` | execution |
| typed failure | `_taken` / `_closed` `daemon.py:580-600` | stop |

Gates:
- `GUARDED` check before `Target.attachToTarget`/`closeTarget`/`activateTarget` (`daemon.py:1213`).
- `_frame_tab` proof before iframe attach (`daemon.py:1216-1219`).
- `session_owner` gate before explicit-session command (`daemon.py:1221-1223`).
- `_current`->`_session` fallback when no session is given (`daemon.py:1226-1230`).

Failure paths: `tab-taken`, `tab-closed`, `tab-released`, `frame-not-yours`, `owner-shared` (client), `daemon-outdated`, `cdp-mismatch`, `remote-browser-ended`.

## F-02 State — owner and tab lifecycle

Question: how does a tab move from unowned to owned, idle, forgotten, closed?

| state | symbol | transition |
|---|---|---|
| owner absent | `Daemon.owner(key)` `daemon.py:545-548` | lazily creates `Owner` |
| tab absent | `_current` `daemon.py:662-671` | create background blank or adopt cloud spare |
| tab owned | `_own` `daemon.py:608-617` | `tab_owner[tid]=key`, `Owner.tabs[tid]=None` |
| session attached | `_session` `daemon.py:673-692` | `Target.attachToTarget`, enable 4 domains |
| current | `_switch` `daemon.py:694-710` | detach old Network, enable new, `o.current=tid` |
| idle (30 min) | `_release(...,"idle")` `daemon.py:726-750` | current kept; others detached, `o.released` |
| forgotten (24 h) | `sweep` `daemon.py:752-766` | `_release(...,"forget")`; last tab given back |
| released/done | `release_tabs` `daemon.py:1115-1119` | close unmarked created, detach all |
| closed/destroyed | `_drop` `daemon.py:619-633` | remove owner/session/marks; one-shot `gone` |

Reversibility: idle release is reversible by `switch_tab` (the tab is still open); close is not. `_save`/`_load` restore owner state across daemon restart for still-existing pages.

## F-03 Data — page metadata, screenshots and downloads

Question: how does browser data reach the caller, and where is it filtered?

| node | symbol | form |
|---|---|---|
| target info | `Target.getTargetInfo` `daemon.py:1210-1219` | `targetInfo` |
| visibility filter | `_visible` `daemon.py:548-556` | same keys, empty `url`/`title` for other tabs |
| tab list | `_meta tabs` `daemon.py:1110-1114` | `{tabs, owner, mark}` |
| current tab | `_meta current_tab` `daemon.py:1195-1207` | `{targetId, url, title}` |
| screenshot | `Page.captureScreenshot` `helpers.py:389-420` | base64 → PNG file |
| JS value | `_runtime_value` `helpers.py:155-180` | JSON value / unserializable decode |
| download event | `_route_event` `daemon.py:826-838` | `guid` → owner, then dropped |
| HTTP | `http_get` `helpers.py:788-820` | str or bytes |

Conservation points: `_visible` preserves keys so client-side filters still work; `list_tabs` explicitly sets `url/title=""` for `owner=="taken"` (`helpers.py:424-439`). A screenshot is base64 in the IPC response; the 32 MiB request cap does not cap responses.

## F-04 Evidence — trace, recording, telemetry, report

Question: what becomes observable, and where is it redacted?

| node | symbol | output |
|---|---|---|
| helper call | `_traced` `run.py:215-240` | `{helper, duration}` in memory, max 500 |
| recorder | `recorder.observe` `run.py:233` | local action recording |
| CLI event | `telemetry.capture_cli_event` `run.py:326-355` | usage event |
| property filter | `_safe_properties` `telemetry.py:149-163` | key whitelist, `://`→`[redacted]` |
| run report | `report.report_run` `report.py:320-326` | `library_judgement` if telemetry on |
| feedback | `report.report_issue` `report.py:328-339` | `library_feedback` regardless of telemetry |
| scrub | `report.scrub` `report.py:134-160` | credentials/URLs/emails/numbers/code |

Gates: `BROWSER_USE_OFFLINE=1` blocks both; telemetry off blocks judgement, not feedback; `BROWSER_USE_ZDR=1` withholds texts; per-hour caps are 3+3. The trace does not capture args; the recorder does, locally.

## F-05 Authority — who may act on which target

Question: what authority check stands between a caller and a browser object?

| node | symbol | authority |
|---|---|---|
| owner claim | `_ipc.owner` `_ipc.py:34-39` | env-provided string |
| request owner | `_send` `helpers.py:81` | `owner` field on every request |
| broker key | `Daemon.handle` `daemon.py:1093` | `req.get("owner") or "default"` |
| tab holder | `_holder` `daemon.py:550-554` | `tab_owner[root]`, unless idle |
| guarded methods | `GUARDED` `daemon.py:37`, `:1213` | `attach/close/activate` refused |
| iframe proof | `_frame_tab` `daemon.py:560-576` | parent chain or `DOM.getFrameOwner` |
| session gate | `session_owner.get(sid,key)` `daemon.py:1221-1223` | explicit session must be registered to caller |
| visibility | `_visible` `daemon.py:548-556` | metadata redaction |
| browser session | `browser_sessions.add` `daemon.py:1226-1228` | **no owner binding** |

Boundary: the authority is a cooperative owner label plus tab routing. It is not authenticated and does not cover browser-domain CDP sent on a `Target.attachToBrowserTarget` session (EK-19).

## F-06 Memory — owners, events, captcha, reports

Question: what survives a call, a tab switch, or a daemon restart?

| store | symbol | lifetime |
|---|---|---|
| `Owner` maps | `Daemon.owners` `daemon.py:503-522` | daemon process |
| owners JSON | `_save`/`_load` `daemon.py:635-660` | across daemon restart, same name/browser |
| event ring | `Owner.events` deque(500) `daemon.py:487-501` | per owner; cleared on `drain_events` |
| captcha state | `captcha_active/finished/taken` `daemon.py:503-543` | daemon process; finish expires 60 s |
| report caps | `reports.json` `report.py:189-218` | across CLI processes, per hour |
| recordings | `recorder` | local files |

Forget paths: `_drop` removes tab state; `sweep` deletes empty owners after a grace window; `_load` drops targets no longer present. A damaged JSON file is logged and ignored (EK-48).

## F-07 Policy — how release and restart rules change future behavior

Question: where does a one-time decision become a rule that governs later calls?

| stage | symbol | policy |
|---|---|---|
| decision | user/agent calls `release_tabs` or goes idle | intent to stop |
| approval | none (cooperative) | no human gate for release |
| policy | `_release` `daemon.py:726-750`; `sweep` `daemon.py:752-766` | "never close the last real page"; "marked tabs survive"; "idle keeps current" |
| enforcement | `close_tab`/`_release`/`_drop` | close only `created && !marked && !last` |
| future decision | next `switch_tab`/`new_tab` | released tabs reappear as free/person tabs; `tab-released` explains |

Restart policy: protocol mismatch (`_restart_if_outdated`), endpoint mismatch (`_check_cdp_endpoint`), active owners, remote marker, pending fingerprint. There is **no** approval/policy persistence for daemon restart; each caller re-evaluates. That makes restart a control decision, not a governed policy loop.

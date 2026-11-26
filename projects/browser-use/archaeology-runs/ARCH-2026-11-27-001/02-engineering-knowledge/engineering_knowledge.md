# 02 Engineering Knowledge — browser-use 1.0.8

Engineering Knowledge (EK) is the wide base. Each entry is a code-anchored fact or mechanism, not a conclusion. EK is never deleted when a Knowledge Object (KO) is formed; KOs must point back to the EK and evidence they aggregate.

Source prefix `repo:` is `/home/user/.doubao/agent_mode/workspace/archaeology-jobs/ARCH-2026-11-27-001/repo`.

## A. Architecture and construction

### EK-01 — `Agent` is a 0.x facade over a different engine
`Agent` keeps the 0.x constructor/`run()` surface but delegates to a Pydantic AI loop and persistent REPL. Removed 0.x parameters raise with a 1.0 alternative; ignored parameters warn. This is a compatibility layer, not evidence that 0.x behavior survived.
- Evidence: `repo:browser_use/agent/service.py:1`, `81-180`, `224-247`; `repo:browser_use/agent/views.py:1-30`; `repo:browser_use/legacy.py`.
- Boundary: some 0.x names still resolve through `legacy.py`, but their implementation is new.

### EK-02 — `build_agent` selects capabilities by model/provider kind
Responses models get `BrowserLoop + LazyCompaction + StreamResponses + Compaction(overflow_only=True) + Recording`; chat models get `BrowserLoop + Compaction + Recording`; other providers get provider-specific thinking settings. `python` is a sequential tool with `max_retries=TOOL_RETRIES`; custom tools are appended after it.
- Evidence: `repo:browser_use/agent/loop.py:1122-1240`.
- Boundary: `build_agent`’s own default model string is `openai-responses:gpt-6-luna`, but the public `Agent` path resolves `None` to `bu-4.0` first (`repo:browser_use/llm/__init__.py:33`, `532-560`).

### EK-03 — `python` is the only built-in action channel
The model-facing built-ins are `python`, `finish`, `finish_from_repl`; `on_text` is an output mode. `python` sends code to the REPL, appends error text, redacts output, and converts native screenshots to `BinaryContent`. There is no DOM serializer or action registry.
- Evidence: `repo:browser_use/agent/loop.py:178-220`, `1122-1180`; `repo:browser_use/agent/prompt.py:7-60`; `repo:browser_use/tools.py:24-30`.
- Boundary: custom `Tools` functions are extra function tools, but they do not receive injected browser/page context.

### EK-04 — REPL parent is the control plane
The parent (`Repl`) owns the worker process, token, per-cell serialization, timeout, redaction of returned values, search/downloads/feedback proxying, and `after_cell` domain sweeping. It never executes model code in its own process.
- Evidence: `repo:browser_use/repl/session.py:304-560`.
- Boundary: the parent holds the Browser Use key and search key; the worker does not receive them.

### EK-05 — REPL worker is a persistent execution plane
The worker runs `python -P -m browser_use.repl.worker` with cwd = workspace, imports browser-harness helpers, builds one namespace, and executes cells with top-level `await` support. Variables persist across cells; stdout/stderr are pumped through a redactor into `cells/<n>.txt`.
- Evidence: `repo:browser_use/repl/worker.py:1-20`, `494-530`, `1142-1200`.
- Boundary: `-P` keeps the workspace off `sys.path[0]`; the worker appends cwd after imports so model-written modules still import.

### EK-06 — browser-harness daemon decouples agent processes from the browser
`Browser.start()` creates or reuses a browser-harness daemon and returns env (`BU_NAME`, `BU_OWN_BROWSER`, optional remote/captcha). The daemon owns the CDP connection; the REPL worker speaks to it through browser-harness helpers. A lost daemon can be restarted without recreating the browser when the env is unchanged.
- Evidence: `repo:browser_use/browser/browser.py:899-999`; `repo:browser_use/repl/session.py:520-560`.
- Boundary: a new daemon with a different env forces a new REPL and loses tabs/logins.

### EK-07 — Four browser kinds are mutually exclusive
`kind` is `cloud`, `chromium`, `chrome`, or `cdp`; `local=True`/`use_cloud=False` map to `chromium`; `cdp_url` maps to `cdp`. Conflicting kind options raise before start.
- Evidence: `repo:browser_use/browser/browser.py:1-9`, `530-680`.
- Boundary: `chrome` with `profile=` is a launch on a profile copy, not an attach.

### EK-08 — Local launch flags are fixed and use an ephemeral debug port
`chrome_args()` uses `--remote-debugging-port=0`, a temp `--user-data-dir`, `--no-first-run`, background-throttling flags, platform keychain/password-store settings, `--headless=new` when no display, and `--no-sandbox` only for Linux root or container markers.
- Evidence: `repo:browser_use/browser/local.py:332-377`.
- Boundary: users cannot pass arbitrary Chrome args in 1.0; `args` raises.

### EK-09 — Attach mode refuses to guess a debug port
`require_debug_port()` refuses to attach unless a live `DevToolsActivePort` exists; it explicitly rejects the old 9222/9223 discovery because any local process could own those ports. `_check_one_chrome()` raises when multiple live debug dirs exist and the current profile is not the intended one.
- Evidence: `repo:browser_use/browser/local.py:66-156`; `repo:browser_use/browser/browser.py:434-487`.
- Boundary: the check prevents attaching to an arbitrary local Chrome, but does not add authentication to the Chrome it does attach.

### EK-10 — Cloud browser is the default and has a billed lifecycle
`Browser()` defaults to cloud with a US proxy and a printed live-view URL. Create has a 120 s total deadline; late browsers are stopped when they land. `_stop_now()` sends PATCH `stop` up to three times and keeps ownership on failure.
- Evidence: `repo:browser_use/browser/browser.py:1000-1125`; `repo:browser_use/browser/cloud.py`; `repo:tests/ci/test_browser.py:45-160`.
- Boundary: cloud behavior is tested with fakes; no live cloud run was performed.

### EK-11 — Detached watchdog kills the browser process group
`start_watchdog()` spawns a detached process that watches the parent PID and kills the Chrome process group plus the daemon when the parent dies. `_kill()` handles POSIX process groups and Windows job objects.
- Evidence: `repo:browser_use/browser/local.py:548-640`.
- Boundary: a SIGKILL of the watchdog itself can leave cleanup to the next stale-dir sweep.

### EK-12 — Runtime directory permissions are checked before start
The browser-harness runtime dir must be mode 0700, not other-writable or other-owned; the Unix socket path length is checked against the OS limit; the daemon name and cloud marker are part of the check.
- Evidence: `repo:browser_use/browser/browser.py:420-487`; `repo:tests/ci/test_browser.py:181-205`.
- Boundary: this protects the daemon directory, not the Chrome debug port.

## B. Control loop and budgets

### EK-13 — Session defaults are larger than `Agent.run` defaults
`Session` defaults: `timeout_s=1700`, `max_steps=1000`, `cell_timeout_s=120`, `vision=True`. `Agent.run` defaults: `max_steps=100`, `timeout=1700`, and it passes `max_steps` into the session. The effective default request budget is therefore 100, not 1000.
- Evidence: `repo:browser_use/agent/loop.py:128-155`; `repo:browser_use/agent/service.py:452-475`.
- Boundary: direct `run(agent, task, Session(...))` callers can get the 1000 default.

### EK-14 — Delivery turn starts at 90% of the budget
`Session.delivery_at()` returns `timeout_s - max(0.1*timeout_s, min(30s, 0.5*timeout_s))`: 90% of the budget, but at most 30 s early, and on very short budgets at least half the budget remains.
- Evidence: `repo:browser_use/agent/loop.py:157-164`.
- Boundary: `timeout` covers the agent loop, not browser start/stop or cloud creation.

### EK-15 — A cell never runs into the delivery turn
`python()` computes `left = delivery_at() - elapsed`; if `left <= 0` it returns `NO_TIME`; otherwise the cell timeout is `min(cell_timeout_s + grace, max(left, 1.0))`. Ultrafast gets 10 s grace out of the remaining budget.
- Evidence: `repo:browser_use/agent/loop.py:178-220`.
- Boundary: a cell can still exceed the timeout in wall-clock if the worker is blocked in an uninterruptible syscall.

### EK-16 — Finalizing forces a delivery-only turn
`BrowserLoop.before_model_request` sets `finalizing` at the delivery deadline or `max_steps`, enqueues `DELIVER_NOW`, and restricts `tool_choice` to `finish`/`finish_from_repl`. If the provider rejects a forced tool choice, `unforced=True` leaves only delivery tools available and lets the model choose.
- Evidence: `repo:browser_use/agent/loop.py:426-520`, `1023-1043`.
- Boundary: in the unforced path, a model can still call a custom tool? `function_tools` are emptied in the unforced request, per the summary; delivery tools remain.

### EK-17 — `max_steps` is a request limit plus one
`run()` passes `UsageLimits(request_limit=session.max_steps+1)`; `UsageLimitExceeded` maps to `status=max_steps`. The extra request is the delivery-only turn.
- Evidence: `repo:browser_use/agent/loop.py:1301-1345`; `repo:browser_use/agent/service.py:452-475`.
- Boundary: `max_steps=1` means only the delivery turn, no exploratory cell.

### EK-18 — Transient provider errors get one retry per run
`on_model_request_error` retries one transient error per run, honors `retry_after` capped at 60 s, does not retry past the delivery deadline, and does not re-retry SDK-retried Gemini requests. A timeout becomes `ModelRetry`; a forced-tool-choice rejection becomes `unforced`.
- Evidence: `repo:browser_use/agent/loop.py:460-520`; `repo:tests/ci/test_loop.py:229-324`.
- Boundary: non-transient errors end the run; a second transient error is not retried.

### EK-19 — Responses requests are streamed with hard deadlines
`StreamResponses` forces `stream=true` for OpenAI Responses, assembles the response from terminal events, treats a stream without a terminal event as a cut connection, and uses 300 s first-event / 930 s total deadlines. Unreadable compaction or reasoning items are dropped and re-sent.
- Evidence: `repo:browser_use/agent/loop.py:591-718`.
- Boundary: chat models do not use this capability.

### EK-20 — Images are pruned in one batch at 20 results
`_prune_images` removes images from all but the newest two tool results once 20 tool results carry images. The comment records the trade-off: pruning breaks the provider prompt cache from that point, but not pruning hits Anthropic’s 100-image limit.
- Evidence: `repo:browser_use/agent/loop.py:1055-1100`; `repo:tests/ci/test_loop.py:118-130`.
- Boundary: `use_vision=False` sends no images at all.

### EK-21 — Parallel `python` calls are serialized
`python` is registered `sequential=True`; `Repl.call` also holds an `asyncio.Lock`. Tests verify that two parallel calls do not interleave and that the second call’s clock starts when it actually starts.
- Evidence: `repo:browser_use/agent/loop.py:1122-1180`; `repo:browser_use/repl/session.py:505-520`; `repo:tests/ci/test_loop.py:141-196`.
- Boundary: the lock is recreated when the event loop changes, so `run_sync` reuse works.

## C. Delivery and verdicts

### EK-22 — Three delivery outputs, one contract
`finish` takes a value and optional `success`; `finish_from_repl` evaluates an expression in the REPL and validates it against the output schema; `on_text` accepts a plain-text reply for `str` output and runs one repair turn for structured output. Tool calls written as text are never delivered; they trigger repair.
- Evidence: `repo:browser_use/agent/loop.py:336-405`.
- Boundary: a pure text delivery has no success verdict (`success=None`).

### EK-23 — Split delivery is rejected once
`check_split_delivery` rejects a response that calls delivery and then another tool, or calls delivery twice, and records one retry per run. A delivery-only turn is exempt.
- Evidence: `repo:browser_use/agent/loop.py:226-250`; `repo:tests/ci/test_loop.py:483-501`.
- Boundary: after the one repair, a second split is accepted with a warning rather than looping.

### EK-24 — Delivery values are checked for placeholders and missing files
`check_delivery` performs one repair for JSON with no value (empty list/object/string/null) and for markdown links or backticked filenames that point to non-existent workspace files. The last request is not repaired; it warns and delivers.
- Evidence: `repo:browser_use/agent/loop.py:252-300`; `repo:tests/ci/test_loop.py:385-482`.
- Boundary: the check is lexical; it does not verify semantic correctness.

### EK-25 — `success` is an agent-authored optional verdict
`finish(success=...)` writes `Session.success`; `AgentHistoryList` keeps it only when `status == completed`, otherwise forces `None`. `is_successful()` returns `self.success`, and `is_done()` only checks `status == completed`.
- Evidence: `repo:browser_use/agent/loop.py:389-400`; `repo:browser_use/agent/views.py:81-150`.
- Boundary: a run that delivers on the delivery-only turn is `completed` even if `success=False`.

### EK-26 — Cached judge is a second request from the same model and prefix
`judge_cached` reuses the run’s history, instructions, and tool definitions as a prefix, appends `CACHED_ASK`, task, optional `judge_context`, and final result, and sends it to the agent’s own model. If the model calls a tool or returns no JSON object, it falls back to `judge_run`.
- Evidence: `repo:browser_use/agent/judge.py:200-276`; `repo:tests/ci/test_judge.py:61-75`.
- Boundary: shared prefix is a prompt-cache optimization, not evidence independence.

### EK-27 — Standalone judge is bounded and evidence-oriented
`JUDGE_PROMPT` treats trace and screenshots as evidence, not instructions; requires no partial credit; caps gotchas at 5; caps trace at 120k chars, part at 5k, task at 20k, screenshots at 4; uses a 180 s timeout; does not retry invalid schema.
- Evidence: `repo:browser_use/agent/judge.py:36-49`, `75-190`.
- Boundary: the judge sees the agent’s trace and final result, not fresh execution; it cannot independently re-run the task.

### EK-28 — Judge failure never changes the run
A failed judge logs only the exception class and HTTP status, returns `None`, and leaves `status`, `text`, and `success` untouched. Judge usage and cost are still added to the result.
- Evidence: `repo:browser_use/agent/judge.py:130-150`; `repo:browser_use/agent/service.py:628-656`; `repo:tests/ci/test_judge.py:95-120`.
- Boundary: judge failures are counted in requests/cost but not surfaced as a run error.

### EK-29 — `is_successful()` ignores `judgement`
`AgentHistoryList.is_successful()` returns the agent’s `success`; `judgement` is a separate field. There is no code path that makes the judge verdict authoritative for `is_successful()`.
- Evidence: `repo:browser_use/agent/views.py:105-150`; `repo:browser_use/agent/judge.py:40-70`.
- Boundary: callers can read `judgement` explicitly, but the convenience API does not.

## D. Context, memory, and partial results

### EK-30 — Server-side compaction is lazy and thresholded
`LazyCompaction` requests OpenAI `context_management` only when the last response’s input tokens exceed `min(150k, 65% of context window)`. The comment records that sending context management on every request was slower at 45k tokens.
- Evidence: `repo:browser_use/agent/loop.py:719-771`.
- Boundary: only Responses models with server-side compaction support use this path.

### EK-31 — Client-side compaction keeps the last two groups and a checkpoint
`Compaction` triggers when chars exceed 75% of `max_chars` or tokens exceed 65% of the window. It keeps the last two assistant/tool groups, summarizes earlier messages with the same model at low effort, and replaces them with one checkpoint plus original user messages.
- Evidence: `repo:browser_use/agent/loop.py:772-900`.
- Boundary: if the summary fails, full history is retained; a context-window error forces one compaction, then ends the run.

### EK-32 — Compaction archive is private but not secret-free by construction
The evidence archive is written to `workspace/.browser-use/context/<uuid>.json` with directory 0700 and file 0600. `_plain` removes system prompts, reasoning, usage, IDs, timestamps, signatures, and provider details; images are replaced with a placeholder.
- Evidence: `repo:browser_use/agent/loop.py:900-1000`.
- Boundary: the archive can contain page text and tool outputs; redaction is applied where the code applies it, not as a blanket guarantee.

### EK-33 — `checkpoint()` is atomic and `partial=True` is a fallback result
`checkpoint(filename, value, partial=False)` redacts the JSON value, writes atomically, and with `partial=True` also writes `state/partial.json` containing the file path and redacted value. `Session.partial()` reads that file if a timeout occurs.
- Evidence: `repo:browser_use/repl/worker.py:742-751`; `repo:browser_use/agent/loop.py:166-175`.
- Boundary: checkpoint files are workspace files; they are not delivered unless the model references them.

### EK-34 — Timeout fallback prefers partial checkpoint, then last text
On timeout, `run()` uses the partial checkpoint if present; otherwise it uses the last redacted text from the transcript. The status is `timeout`, and `success=None`.
- Evidence: `repo:browser_use/agent/loop.py:1338-1365`.
- Boundary: a timeout can return text that was never delivered through `finish`; callers must inspect `status`.

### EK-35 — Transcript redaction is conditional and incomplete for secrets
If secrets exist, `run()` redacts text, thinking, and tool-call arguments in `result.messages` before returning history. It does not redact screenshots, `js()` reads, files the model writes itself, or follow-up transcripts that continue unredacted.
- Evidence: `repo:browser_use/agent/loop.py:1355-1370`; `repo:MIGRATING.md` “Nothing else is redacted”.
- Boundary: the redactor covers many encodings/forms but not arbitrary transforms (for example rot13 or split strings).

## E. Security, secrets, and authority

### EK-36 — Worker env allowlist strips secret-looking names
`worker_env()` copies only allowlisted variables and prefixes (`PATH`, `HOME`, locale, temp, `DISPLAY`, `PYTHONPATH`, `VIRTUAL_ENV`, proxies/CA, `BU_*`, `BH_*`), removes `BU_CDP_WS`/`BU_CDP_URL`/`BU_MODE`/`BU_HIGHLIGHT_ACTIONS`, and excludes names matching `KEY|TOKEN|SECRET|PASSW|CREDENTIAL|AUTH`.
- Evidence: `repo:browser_use/repl/session.py:42-88`; `repo:tests/ci/test_repl.py:91-133`.
- Boundary: allowlisted proxy URLs can contain passwords; the worker still shares the user’s UID and filesystem.

### EK-37 — Redaction is a multi-form output filter
`redact`/`redact_forms` replace secret originals and JSON/repr/HTML-escaped, lower/upper/reverse, base64, hex, URL-encoded forms with `<secret:name>`; short secrets are replaced only as standalone words/numbers. It is applied to cell output/error/value, checkpoints, custom tool results, task text, final result fields, logs, and telemetry.
- Evidence: `repo:browser_use/repl/session.py:135-200`; `repo:browser_use/repl/worker.py:1016-1034`.
- Boundary: the code explicitly says screenshots, `js()` values, files written by the model, cloud live view, and arbitrary transforms are not covered.

### EK-38 — `type_secret` scopes values to a focused frame
`type_secret(name, id=None)` finds the secret, converts a `*bu_2fa_code` seed to a current TOTP code, follows focus through iframes/shadow roots in an isolated world, and types only if the receiving frame URL matches the domain scope. If the frame cannot be established, scoped secrets fail closed.
- Evidence: `repo:browser_use/repl/worker.py:753-897`.
- Boundary: the page can move focus between the check and the key events; the code documents this race.

### EK-39 — The worker is not a sandbox
`Repl` and `MIGRATING.md` state that model-written Python runs as the user and can read user-readable files, the parent’s `.env`, and other processes’ environments. The process boundary exists to kill hung cells and isolate variables, not to contain the model.
- Evidence: `repo:browser_use/repl/session.py:1-18`; `repo:MIGRATING.md` “Trust model for secrets”; `repo:tests/ci/test_repl.py:121-128`.
- Boundary: the test named `test_trust_model_says_the_worker_can_still_reach_the_parents_keys` encodes this as expected behavior.

### EK-40 — Workspace is a delivery convention, not a filesystem boundary
The workspace is the REPL cwd and the recommended place for deliverables; the system prompt says files outside it are not delivered. There is no file allowlist, and `available_file_paths`/`file_system_path` were removed.
- Evidence: `repo:browser_use/agent/loop.py:1215-1240`; `repo:browser_use/agent/service.py:94-120`; `repo:MIGRATING.md`.
- Boundary: the model can read and upload any file the user can read.

### EK-41 — Domain matching canonicalizes as the browser does
`parse_url` strips C0 controls, tabs/newlines, treats backslashes as slashes in HTTP(S), resolves dot segments, IDNA-maps hosts, and normalizes IPv4 forms. Patterns support exact host, `*.` subdomains, scheme/port, path prefixes, globs, and `localhost:port`. `about:blank`, new-tab, `data:`, `blob:` are always allowed; `file:` and hostless URLs are denied.
- Evidence: `repo:browser_use/domains.py:47-197`.
- Boundary: pattern semantics are intentionally close to 0.x but add subdomain blocking and localhost:port.

### EK-42 — Domain Guard intercepts main-frame document requests and sweeps tabs
`Guard` opens an independent CDP connection, enables `Fetch`, and fails disallowed main-frame document requests with `net::ERR_BLOCKED_BY_CLIENT`; `sweep()` resets tabs still on disallowed pages after each cell. It does not filter iframes or subresources.
- Evidence: `repo:browser_use/domains.py:281-455`; `repo:browser_use/browser/browser.py:1054-1072`.
- Boundary: speculation rules, prerender, and bfcache can navigate without a request the Guard sees; sweep is the backstop.

### EK-43 — Attach mode only sweeps agent-owned tabs
When attached to the user’s Chrome, `Browser._sweep()` passes `only=self._touched_tabs()`; `_user_tabs` are preserved. The tests verify that a user’s own tab on a disallowed page is not reset.
- Evidence: `repo:browser_use/browser/browser.py:1044-1072`; `repo:tests/ci/test_domains.py:643-730`.
- Boundary: attached tabs the agent used are reset; the user’s unrelated tabs are not.

### EK-44 — Local debug ports are unauthenticated loopback capabilities
Local Chrome is launched with `--remote-debugging-port=0` and the WebSocket is read from `DevToolsActivePort` or stderr. Chrome’s DevTools WebSocket has no authentication; any local account that can reach loopback can attach while the browser runs. `keep_alive=True` extends the window.
- Evidence: `repo:browser_use/browser/local.py:104-145`, `332-377`; `repo:browser_use/browser/browser.py:1-9`; `repo:MIGRATING.md`.
- Boundary: attaching to the user’s already-open Chrome can require Chrome’s “Allow remote debugging?” approval; the launched Chromium/profile copy has no such gate. Approval is a UI gate, not an authentication token.

### EK-45 — Profile copies carry cookies and require real Chrome
`Browser('chrome', profile='Work')` resolves a profile by display/directory name, copies `Local State` plus the profile directory to a temp dir, launches with `--disable-sync`, and uses mock keychain / basic password store for fresh profiles. The copy can include cookies and passwords; stale copies are swept when the owner process is dead.
- Evidence: `repo:browser_use/browser/local.py:260-330`, `415-470`, `498-545`.
- Boundary: the code requires a real Chrome binary for a profile copy; Chromium is rejected.

### EK-46 — Cloud live view and API key are bearer capabilities
The cloud live-view URL is printed to stderr and, by the docs, anyone with the link can control the browser. The API key authorizes cloud browser creation and the default `bu-4.0` model; `Browser(live_view=False)` only suppresses printing, not the capability.
- Evidence: `repo:browser_use/browser/browser.py:1000-1125`; `repo:README.md` cloud section; `repo:MIGRATING.md`.
- Boundary: recording is off by default, but live view remains a control channel.

### EK-47 — API key precedence and automatic login are bounded
Key resolution order: `BROWSER_USE_API_KEY`, then browser-harness stored key, then a gateway model key. Automatic login runs only on an interactive main-thread run with a single-task event loop; SSH/no-display uses device code; CI/notebook/container/background runs do not auto-login.
- Evidence: `repo:browser_use/auth.py:76-150`, `202-247`; `repo:browser_use/cli.py:379-455`.
- Boundary: a missing key in an interactive terminal can block for up to 600 s; the login attempt is once per process.

### EK-48 — Daemon directory checks prevent cross-user socket hijack
Before start, the runtime dir must be 0700 and not other-writable/owned; socket path length is checked; multiple live debug dirs raise. This prevents another local user from planting a daemon socket or a stale port file.
- Evidence: `repo:browser_use/browser/browser.py:420-487`; `repo:tests/ci/test_browser.py:181-205`.
- Boundary: it does not authenticate the Chrome DevTools endpoint itself.

## F. Extensibility and interfaces

### EK-49 — Custom tools are plain functions, with no injected context
`Tools.action` registers a Pydantic AI `Tool` with `takes_ctx=False`; injected 0.x parameters (`browser_session`, `page`, `cdp_client`, etc.) raise. `ActionResult` is kept as JSON-visible compatibility; removed fields (`is_done`, `images`, `success`, `attachments`) raise or redirect.
- Evidence: `repo:browser_use/tools.py:1-30`, `90-270`.
- Boundary: a custom tool runs in the parent process, not the worker; it cannot drive the browser directly unless it calls the daemon itself.

### EK-50 — `--cli-mcp` exposes four code-execution tools
The MCP server exposes `browser_exec`, `browser_screenshot`, `browser_login`, `browser_feedback`; `browser_exec` runs Python on a thread with browser-harness helpers, replies within the client’s ~60 s call window, and supports empty code to wait for a still-running cell. Old 0.x tool names return a migration error.
- Evidence: `repo:browser_use/mcp/cli_mcp.py:61-100`, `509-560`, `675-730`.
- Boundary: there is no per-action schema; the MCP client gets code execution and a screenshot, not a typed browser API.

### EK-51 — CLI prefers the user’s Chrome and falls back to its own Chromium
`--cli-mcp` tries to attach to the user’s Chrome; if Chrome debugging is unavailable, it starts its own Chromium on a temp profile. `--doctor` checks install, daemon, and browser; `login` supports browser/device/API-key-stdin; `skill install` writes the agent skill.
- Evidence: `repo:browser_use/cli.py:251-300`, `379-455`, `598-706`.
- Boundary: the fallback browser is a different authority domain from the user’s logged-in Chrome.

### EK-52 — 0.x compatibility is name-by-name, not behavioral
`legacy.py` and `removed_names` map old module names to explicit errors naming the 1.0 alternative. `agent/service.py` has `REMOVED` and `IGNORED` tables. Tests assert old MCP tool names and old option names produce the migration error.
- Evidence: `repo:browser_use/legacy.py`; `repo:browser_use/agent/service.py:81-180`; `repo:tests/ci/test_compat.py:1195-1240`.
- Boundary: compatibility is import/constructor-level; old control flow does not survive.

## G. Testing and operations

### EK-53 — Test suite is split into non-Chrome and Chrome jobs
25 test files, 429 test functions, 18,492 lines. Local `tests/ci -m 'not chrome'` gives 964 passed, 3 skipped, 140 deselected; CI runs the 140 Chrome tests in a separate ubuntu job with scripted `FunctionModel`.
- Evidence: snapshot metrics; `repo:.github/workflows/test.yaml`; `repo:tests/ci`.
- Boundary: the local run did not exercise real Chrome, cloud, or providers.

### EK-54 — Scripted `FunctionModel` is the main loop test double
Loop, judge, and domain tests use a scripted model to emit exact tool calls/text and assert on requests, results, and side effects. Real Chrome tests are marked `chrome` and use a local site fixture.
- Evidence: `repo:tests/ci/test_judge.py:61-75`; `repo:tests/ci/test_domains.py:671-730`; `repo:tests/ci/conftest.py`.
- Boundary: scripted models verify orchestration, not model quality.

### EK-55 — Telemetry defaults on and separates auth events from keys
`ANONYMIZED_TELEMETRY` defaults true (an empty value also counts true); telemetry runs a background thread with an outbox; auth events record stage/method/automatic/fields but not the key or account id.
- Evidence: `repo:browser_use/config.py:64-66`; `repo:browser_use/telemetry/service.py:201-300`; `repo:browser_use/auth.py:150-200`.
- Boundary: telemetry is opt-out; task text, typed text, code, and secrets are excluded by design, but the docs still tell users to review the telemetry section.

## EK graph summary

- **Runtime spine**: EK-01 → EK-02 → EK-03 → EK-04 → EK-05 → EK-06.
- **Browser authority**: EK-06 → EK-07 → EK-08/EK-09/EK-10 → EK-11/EK-12.
- **Budget and delivery**: EK-13 → EK-14 → EK-15 → EK-16 → EK-17 → EK-22 → EK-23/EK-24 → EK-33/EK-34.
- **Verdict**: EK-25 → EK-26/EK-27 → EK-28 → EK-29.
- **Context**: EK-20 → EK-30/EK-31 → EK-32/EK-33 → EK-34/EK-35.
- **Security**: EK-36 → EK-37/EK-38 → EK-39/EK-40; EK-41 → EK-42/EK-43; EK-44/EK-45/EK-46/EK-47/EK-48.
- **Interfaces/tests**: EK-49/EK-50/EK-51/EK-52; EK-53/EK-54/EK-55.

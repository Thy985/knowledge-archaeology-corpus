# 02 · Engineering Knowledge（EK Graph）— browser-use 1.0.8rc1

> EK is the wide, lossless base. Each EK states a concrete mechanism, implementation, decision, failure mode, test-revealed behavior, configuration, or boundary, and carries at least one graph edge. No EK is deleted to make a KO look cleaner.
>
> Edge types: `mechanism`, `subsystem`, `causal`, `dependency`, `constraint`, `contrast`. Evidence IDs map to `06_validation/evidence_map.md`.

## EK-01 Run lifecycle is a single ordered control chain

- **Claim**: One `Agent.run()` executes a fixed order: argument validation, model resolution, recorder/telemetry, provider/search preflight, workspace creation, browser start, REPL creation, session creation, loop, history capture, browser shutdown, then judge.
- **Detail**: `_run()` calls `_check_task()`, `raise_fd_limit()`, `_build()`, then starts `RunRecorder` and telemetry, then preflight, then `workspace`/`downloads`, then `browser.start()`, then constructs `Repl`, then `Session`, then `run()`, then saves history and captures the final page. Its `finally` shuts down the browser, and only after that calls `_judge()` (inside the shutdown `finally`). `_build()` does model resolution before browser start.
- **Why (source-stated)**: the code comments and order are designed to fail on configuration before paying for a browser and to keep shutdown unconditional.
- **Evidence**: EV-19, EV-21, EV-22; `agent/service.py:380-627,699-718`.
- **Boundary**: `run_sync()` is a wrapper; re-entrant loops and concurrent runs are guarded, not supported.
- **Links**: mechanism→EK-02 (preflight), causal→EK-27 (browser lifecycle), subsystem→EK-04 (loop).
- **Layer**: L2 (implementation decision).

## EK-02 Preflight runs before browser spend

- **Claim**: Model/key/extra resolution and official-provider probes happen before `Browser.start()`.
- **Detail**: `resolve_model()` turns `llm=None` into `bu-4.0`, raises on missing key/extra, and rewrites Pydantic AI `UserError` with env hints. `provider_preflight()` only probes official OpenAI/Anthropic/Google endpoints; invalid key or 404 fails, while 403/proxy/unknown client/non-invalid 401 return `None` and do not block. `_build()` also resolves gateway/pricing and builds prompts before returning.
- **Why**: a bad key or missing provider extra should not create a billable cloud browser.
- **Evidence**: EV-19, EV-83, EV-84; `agent/service.py:380-450`; `llm/__init__.py:461-619`.
- **Boundary**: prefixed proxies and unknown clients are not probed; preflight is not a guarantee that the model call will succeed.
- **Links**: mechanism→EK-28 (auth/key), causal→EK-27 (browser spend), constraint→EK-01.
- **Layer**: L2.

## EK-03 One active run per Agent and one active Agent per Browser

- **Claim**: The runtime refuses overlapping runs on the same `Agent`, and refuses a second Agent using a `Browser` already owned by another active Agent.
- **Detail**: `run()` checks `self._running` / `_browser._agent` and raises `RuntimeError`; `max_steps` and `timeout` are validated as positive integers.
- **Why**: the browser is stateful and the Repl is one worker; concurrent agents would interleave tabs and delivery.
- **Evidence**: EV-20; `agent/service.py:452-474`.
- **Boundary**: this is process-local; separate processes can attach to the same external Chrome if the user does so.
- **Links**: constraint→EK-27, constraint→EK-04, contrast→EK-15 (MCP has a different concurrency model).
- **Layer**: L2.

## EK-04 The model acts by writing Python in a persistent REPL

- **Claim**: The 1.0 action interface is a `python(code)` tool executed in a persistent worker namespace, not a predefined DOM action schema.
- **Detail**: `build_agent()` registers `python` as a sequential tool; `python()` calls `Repl.call('exec', code, ...)`; the worker imports `browser_harness.helpers` and exposes a star-import plus `cdp()`. Variables persist across cells until a worker restart. `finish`, `finish_from_repl`, and `on_text` are separate output tools.
- **Why (docs/comments)**: `AGENTS.md` describes “one persistent Python REPL on raw CDP”; the package description says the same.
- **Evidence**: EV-09, EV-27, EV-33, EV-43; `agent/loop.py:175-225,1122-1300`; `repl/worker.py:494-560,1135-1229`; `AGENTS.md`.
- **Boundary**: the model still sees only helper names and `cdp()`; it does not get kernel-level authority. Helper behavior lives in `browser-harness`, outside this repository.
- **Links**: subsystem→EK-05, mechanism→EK-06, dependency→EK-43 (external helpers), contrast→EK-15.
- **Layer**: L2.

## EK-05 Cells are serialized; each cell has a deadline

- **Claim**: One REPL executes one cell at a time, and every cell is bounded by the remaining run budget plus a cell timeout/grace.
- **Detail**: `Repl.call()` holds a single `asyncio.Lock`. `python()` computes `left = delivery_at() - elapsed`; if `left <= 0`, returns `NO_TIME` without executing. Otherwise it calls `repl.call('exec', code, min(cell_timeout + grace, max(left, 1)))`. Ultrafast mode adds a 10-second grace.
- **Why**: the browser is a single stateful resource; parallel cells would race on tabs and page state. The deadline prevents a hung cell from consuming the entire run.
- **Evidence**: EV-26, EV-27, EV-40; `agent/loop.py:157-225`; `repl/session.py:460-523`.
- **Boundary**: a cell can start work that outlives the call (threads/subprocesses); killing the worker is the containment mechanism.
- **Links**: mechanism→EK-06, constraint→EK-26 (partial recovery), causal→EK-01.
- **Layer**: L2.

## EK-06 Timeout kills the worker process group and restarts it

- **Claim**: A cell timeout is handled by closing the worker, killing its process tree, reading the partial cell output, and starting a new worker.
- **Detail**: `_call()` on timeout/disconnect calls `close()`, which kills the worker process group and waits up to 5 s. It then reads `cells/<n>.txt` for partial output and returns an error that says actions may already have happened. The next cell starts a new worker. Files and browser state persist; Python variables do not.
- **Why**: a hung cell may have already sent CDP commands; the safe recovery is to stop the process tree and tell the model to inspect before retrying, not to pretend the cell did nothing.
- **Evidence**: EV-39, EV-41, EV-42, EV-45; `repl/session.py:443-568`; `repl/worker.py` cell loop; `tests/ci/test_loop.py:71-92`.
- **Boundary**: killing the worker cannot undo browser actions already committed. It also cannot kill a process that escaped the process group (for example a daemon started outside).
- **Links**: causal→EK-05, mechanism→EK-26, constraint→EK-04.
- **Layer**: L2.

## EK-07 Worker transport is loopback with a token

- **Claim**: The parent and worker communicate over a loopback TCP connection authenticated by a random token, without inheriting a file descriptor.
- **Detail**: the worker connects to `127.0.0.1` on an ephemeral port and presents a random token; the parent uses `hmac.compare_digest`. This is documented as cross-platform (Windows/macOS/Linux) and avoids fd inheritance.
- **Why**: a socket + token is portable and avoids exposing a file descriptor to child processes.
- **Evidence**: EV-37; `repl/session.py` module docstring, `_start`.
- **Boundary**: loopback is not a sandbox; any same-user process that can connect and guess/steal the token is outside the threat model. The token is held in process memory and passed via environment to the worker.
- **Links**: mechanism→EK-08, constraint→EK-04, contrast→EK-15.
- **Layer**: L1 (implementation).

## EK-08 Worker environment is allowlisted and credential-stripped

- **Claim**: The worker receives only an allowlisted environment plus `LC_`, `BU_`, `BH_` prefixes; names matching key/token/secret/password/credential/auth are removed.
- **Detail**: `worker_env()` denies `BU_CDP_WS`, `BU_CDP_URL`, `BU_MODE`, `BU_HIGHLIGHT_ACTIONS`; it preserves proxy/CA variables. The parent then adds explicit `BU_REPL_PORT/TOKEN`, `BH_REQUIRE_EXISTING_DAEMON=1`, a sentinel `BU_CDP_WS`, `BH_AGENT_WORKSPACE`, `BU_REPL_PARENT_PID`, `BH_TELEMETRY=0`, and `BU_MODEL_CODE=1`.
- **Why**: the child should not inherit API keys or arbitrary parent configuration. `BH_REQUIRE_EXISTING_DAEMON` prevents a naive harness CLI call in a cell from discovering the user's local Chrome.
- **Evidence**: EV-38, EV-39; `repl/session.py:1-180,357-442`.
- **Boundary**: environment filtering does not hide `/proc/<pid>/environ`, same-user files, or the parent process; `test_repl.py` explicitly asserts the docs must say so (EV-52).
- **Links**: mechanism→EK-09, constraint→EK-10, contrast→EK-15.
- **Layer**: L2.

## EK-09 Secrets travel through the control channel, not the worker environment

- **Claim**: `sensitive_data` values are injected into the worker through the REPL control channel and redacted on return; they are not exported as environment variables.
- **Detail**: `Repl` receives `secrets`; the worker wraps helper sends; `type_secret()` resolves a named secret, supports TOTP, focuses the field, and types it. The return path redacts output/error/value/error_message and cell files.
- **Why**: keeping secrets out of the environment reduces accidental exposure to subprocesses and third-party libraries. The redaction layer protects model-facing output.
- **Evidence**: EV-38, EV-46, EV-48, EV-50; `repl/session.py:304-568`; `repl/worker.py:742-900`.
- **Boundary**: this is not isolation. The worker can read files, environment, screenshots, and live view with the user's permissions; short secrets are only whole-word redacted; `open()` writes are not redacted; screenshots are not redacted.
- **Links**: mechanism→EK-10, contrast→EK-15, constraint→EK-44 (MCP lacks this).
- **Layer**: L2.

## EK-10 Redaction covers model-facing channels, not the machine

- **Claim**: `redact()` covers plain/JSON/repr/HTML/case/reversed/base64/hex/URL-encoded forms, but leaves explicit residual channels.
- **Detail**: `secret_forms()` generates variants, including a core substring inside a longer base64 string; replacements run longest first. `SHORT_SECRET=4`; shorter values are only replaced as whole words. Cell stdout/stderr and between-cell background output pass through a redacting pump. Arbitrary `open()` writes, image pixels, live view, recording, and environment reads do not.
- **Why**: the source documentation explicitly distinguishes “redaction” from “sandbox”; tests assert the docs must admit the worker can reach parent keys.
- **Evidence**: EV-14, EV-50, EV-51, EV-52; `repl/session.py` secret_forms/redact; `MIGRATING.md:335-404`; `tests/ci/test_repl.py`.
- **Boundary**: a secret shorter than 4 characters can survive as part of a longer token; a page can display a secret in a screenshot; a worker-written file can contain it.
- **Links**: mechanism→EK-09, constraint→EK-11, contrast→EK-44.
- **Layer**: L2.

## EK-11 Scoped secret typing fails closed outside its domain

- **Claim**: A domain-scoped secret is typed only when the current URL matches its domain pattern; otherwise the worker raises `PermissionError`.
- **Detail**: `type_secret()` checks the current URL against the secret's domain; `_recipient()` resolves the focused element through the frame tree and an isolated world rather than page JS; `type_into_field()` rejects BODY/HTML and non-editable fields. If the recipient cannot be confirmed, a scoped secret fails closed.
- **Why**: the threat is typing a password into a lookalike or cross-origin iframe after focus moves. The isolated-world read and fail-closed behavior reduce that risk.
- **Evidence**: EV-48, EV-49; `repl/worker.py:807-900`.
- **Boundary**: documentation admits a focus race between the check and the keystrokes; a page can move focus after the check. This is a mitigation, not a proof.
- **Links**: mechanism→EK-09, constraint→EK-10, causal→EK-04.
- **Layer**: L2.

## EK-12 Domain matching is pattern-based, with explicit allow/prohibit precedence

- **Claim**: Domain policy treats `[]` as unrestricted; if both allow and prohibit lists are given, prohibit is ignored; `block_ips` blocks IP literals even when allowed.
- **Detail**: `DomainRules.of()` builds rules. `blocked()` allows `about:blank`, `data:`, `blob:`, and Chrome internal schemes; blocks `file:` and hostless URLs; when `block_ips` is set, rejects IP literals; when allowed is non-empty, requires a match; otherwise uses prohibited with subdomain matching.
- **Why**: an allowlist should be a positive list; a contradictory “allow + prohibit” combination is resolved by dropping prohibit with a warning, matching 0.x behavior.
- **Evidence**: EV-61, EV-62; `domains.py:198-243`.
- **Boundary**: pattern semantics (glob/subdomain) and IDN/IP canonicalization are handled by the code; this run did not fuzz them. `block_ips` only blocks literal IPs, not DNS names resolving to an IP.
- **Links**: mechanism→EK-13, constraint→EK-14.
- **Layer**: L2.

## EK-13 Guard is a separate CDP connection enforcing main-frame document requests

- **Claim**: Layer 2 uses a second CDP WebSocket with `Fetch.enable` for `Document` requests and resets disallowed targets to `about:blank`.
- **Detail**: `Guard` enables target discovery and Fetch at Request stage; `_handle` checks main-frame document requests only; iframes pass. `Target.targetInfoChanged` to a disallowed URL triggers `_reset`. `_failed` records the guard-caused error page.
- **Why**: helper checks can be bypassed by page JS, redirects, or direct CDP; a browser-level request interception is a second enforcement point.
- **Evidence**: EV-63, EV-64; `domains.py:281-441`.
- **Boundary**: it does not cover subresources, iframes, or a worker's own `requests/socket/curl` (except `http_get`). It also relies on request/target events for prefetch/prerender and does not see paths that commit without a request. Attached `Browser('chrome')` has no Guard at all, so its `Runtime.evaluate` navigations are outside this layer.
- **Links**: mechanism→EK-12, constraint→EK-14, causal→EK-15 (fail-open).
- **Layer**: L2.

## EK-14 Guard failure is fail-closed at start, fail-open at disconnect

- **Claim**: If a Guard is requested and cannot start, Browser start raises; if an already-started Guard disconnects later, Chrome stops intercepting until reconnect.
- **Detail**: `Guard.start()` has `GUARD_START_S` and raises `RuntimeError`; `Browser._start()` raises and `_stop()` runs when `self.domains` is set. A dropped connection is debug-logged and reconnected with exponential backoff; during the gap only layer 1/3 apply. Attached `Browser('chrome')` does not instantiate a Guard at all and returns before this block, so the start guarantee does not apply to that mode.
- **Why**: starting without a guard while the user requested restrictions would be a silent policy downgrade. A transient disconnect cannot block the entire run, so the code reconnects, but the gap is real.
- **Evidence**: EV-65, EV-68; `domains.py:308-390`; `MIGRATING.md:232-249`; `tests/ci/test_domains.py:731-762`.
- **Boundary**: the test only proves reconnect and a block after reconnect, not that no request escapes during the gap. The docs explicitly call it fail-open.
- **Links**: causal→EK-13, constraint→EK-12, contrast→EK-15.
- **Layer**: L2.

## EK-15 Attached `Browser('chrome')` has a reduced policy set

- **Claim**: Attaching to the user's open Chrome deliberately omits Guard and only sweeps agent-created tabs.
- **Detail**: `_attached()` is true for `Browser('chrome')` without profile; `_touched_tabs()` covers tabs file, current tab, popups; `sweep(only=agent_tabs)` leaves the user's tabs. Pre-existing disallowed tabs are recorded in `_user_tabs` and exempt while they remain on the same URL. `Browser('chrome')` cannot open a second CDP connection for Guard because Chrome would prompt again.
- **Why**: the alternative is a second authorization prompt or resetting the user's own tabs. The design chooses non-interference with the user's browser.
- **Evidence**: EV-13, EV-54, EV-58; `browser/browser.py:620-760,1042-1060`; `MIGRATING.md:180-249`; `tests/ci/test_domains.py:434-458`.
- **Boundary**: this is not equivalent to cloud/chromium policy. A page-level `Runtime.evaluate` navigation is not caught by Guard; the after-cell sweep is the only layer 3.
- **Links**: contrast→EK-13, constraint→EK-12, subsystem→EK-27.
- **Layer**: L2.

## EK-16 MCP `browser_exec` runs in the server process

- **Claim**: MCP code execution is not the Agent REPL; it uses `exec(code, ns)` in the MCP server process with a persistent namespace.
- **Detail**: `CLIMCPServer._execute()` executes code in-process, captures stdout/stderr, returns tool errors. It has an exec lock between calls. If a call exceeds `CALL_SECONDS`, it returns a reply while the code continues in a separate thread; the next `browser_exec(code="")` waits and collects output.
- **Why**: MCP is a lightweight stdio server; reusing the Agent worker would add a process group, env allowlist, and secrets plumbing. The trade-off is a different authority model.
- **Evidence**: EV-92, EV-93; `mcp/cli_mcp.py:675-774,880-1027`.
- **Boundary**: no `sensitive_data`, no `type_secret`, no worker env allowlist, no checkpoint redaction, no independent process group. The code runs with the MCP server's user permissions.
- **Links**: contrast→EK-04, contrast→EK-09, causal→EK-17.
- **Layer**: L2.

## EK-17 MCP cloud mode refuses domain-constrained admin calls

- **Claim**: When MCP starts a remote cloud daemon, cloud admin calls are refused if domains are configured, because the cloud browser can reach any site.
- **Detail**: `_refuse_cloud` rejects admin actions when domains exist; MCP server exit stops the cloud browsers it started; `BH_OPEN_LIVE_URL=1` controls live view/CDP URL exposure.
- **Why**: a cloud browser is outside the local Guard's control; pretending domain restrictions apply would be a false policy claim.
- **Evidence**: EV-94; `mcp/cli_mcp.py:463-472,856-879,1028-1100`.
- **Boundary**: this is a refusal of admin calls, not a Guard on cloud browser traffic. Other MCP calls may still drive the cloud browser.
- **Links**: causal→EK-16, constraint→EK-12, contrast→EK-13.
- **Layer**: L2.

## EK-18 Delivery has a budget turn and one repair turn

- **Claim**: The loop reserves a delivery-only window and gives one repair turn when delivery is invalid.
- **Detail**: `delivery_at()` starts at 90% of the budget (or leaves up to 30 s). `DELIVER_NOW` tells the model to call `finish`/`finish_from_repl`, reference saved files, state missing evidence, and stop browser actions. If a run ends without valid delivery, `DELIVERY_REPAIR` gives one turn to deliver from existing results or a concise answer.
- **Why**: partial results are more valuable than a timeout with no answer; the repair turn converts a formatting error into a deliverable.
- **Evidence**: EV-26, EV-29; `agent/loop.py:86-98,157-174,336-424`.
- **Boundary**: the repair is only one turn; a model can still fail to produce a valid delivery after it.
- **Links**: mechanism→EK-19, causal→EK-20, constraint→EK-34.
- **Layer**: L2.

## EK-19 Delivery checks are one-shot and skip the last request

- **Claim**: Placeholder/missing-file checks raise `ModelRetry` only before the final request; on the last request they warn and deliver.
- **Detail**: `check_delivery()` inspects placeholders (no alphanumeric text or empty structured result) and markdown/file links to non-existent workspace files. It records `delivery_placeholder`/`delivery_missing_files`. At `ctx.run_step > s.max_steps`, it does not reject. Tests assert a repair on the first occurrence and acceptance of the same finish on the last request.
- **Why**: preventing an infinite repair loop at the step limit; the alternative is returning no answer. The trade-off is that a bad delivery can be accepted at the boundary.
- **Evidence**: EV-31, EV-36; `agent/loop.py:252-335`; `tests/ci/test_loop.py:710-737,1194-1215`.
- **Boundary**: this is a best-effort check, not a proof of factual correctness; a non-placeholder but false answer passes.
- **Links**: mechanism→EK-18, constraint→EK-20, causal→EK-21.
- **Layer**: L2.

## EK-20 Split delivery is refused per request step

- **Claim**: A response that combines a delivery call with another tool call, or contains two delivery calls, is refused; the guard is per request step, despite a comment saying “once per run”.
- **Detail**: `check_split_delivery()` detects a delivery call plus another tool call after it, or another delivery call before it. It allows `python` before a delivery in the same response. `split_delivery_step` stores the current `ctx.run_step` and skips only that same step, so a later step can be refused again. No test for a second-step refusal was found. A delivery-only turn is exempt.
- **Why**: a delivery should be the final action of that response; further tool calls after delivery would make the answer stale and confuse the transcript.
- **Evidence**: EV-30, EV-36; `agent/loop.py:139,226-251`; `tests/ci/test_loop.py` split-delivery tests.
- **Boundary**: it is a per-response structural check, not a global exactly-once guarantee; a model can deliver on a later response. The “once per run” comment is contradicted by the step-scoped condition.
- **Links**: constraint→EK-18, mechanism→EK-19.
- **Layer**: L2.

## EK-21 Judge is a cached final turn that cannot fail the run

- **Claim**: The judge is an optional post-hoc verification turn; a failed judge leaves `judgement=None` and the run's status/text unchanged, while its tokens/cost are counted.
- **Detail**: `_judge()` only runs if `judge` is true, there is a result with steps, and `_agent` exists. It first calls `judge_cached()` to append a final turn and reuse the run's instructions/tool definitions for prompt cache; if that fails or produces no schema answer, it calls `judge_run()`. The judge prompt treats trace/screenshots as evidence, not instructions; bounds trace/task/parts/screenshots; redacts input/output; a failed judge logs a warning and returns None. `result.status` is not changed. The final page screenshot is captured before `Browser.stop()`; `_judge()` itself runs after the browser is stopped.
- **Why (source/comments)**: a verification layer should not become a new failure source in the execution path; the run's own `finish(success=...)` verdict remains separate.
- **Evidence**: EV-69 to EV-75; `agent/judge.py:37-276`; `agent/service.py:628-657`; `tests/ci/test_judge.py:61-95`.
- **Boundary**: judge is advisory; it does not gate `status`, retry the task, or block a false `success`. The same model may share the agent's blind spots.
- **Links**: mechanism→EK-18, contrast→EK-24 (agent verdict), constraint→EK-34.
- **Layer**: L2.

## EK-22 Compaction triggers on a character budget or context window

- **Claim**: Compaction begins past 75% of the char budget or 65% of the model context window, with a default 4 chars/token and 240k chars when the window is unknown.
- **Detail**: `Compaction` / `LazyCompaction` project the message history; the trigger is checked before a model request. The tests verify 75% char-budget behavior and 65% window behavior.
- **Why**: long browser runs accumulate tool outputs and screenshots; compacting before overflow keeps the next request inside the provider limit.
- **Evidence**: EV-76; `agent/loop.py:772-1000`; `tests/ci/test_compaction.py:80-120`.
- **Boundary**: the budget is an estimate, not a provider tokenizer; a provider can still reject for context length.
- **Links**: mechanism→EK-23, causal→EK-24, dependency→EK-25.
- **Layer**: L2.

## EK-23 Compaction pins the user request and writes a redacted evidence archive

- **Claim**: The projected prompt contains a summary checkpoint plus the last two assistant/tool groups, with user prompts pinned verbatim; the covered messages are written to a redacted archive.
- **Detail**: the archive has version, summary, `covered_messages`, and covered `messages`; it is written 0600 under `.browser-use/context`; images are replaced with `[Image omitted from text archive]`. The projected history is rejected if it is not smaller than the original.
- **Why**: a summary is fallible; retaining the covered evidence separately lets a later reader audit it without keeping the whole history in the model prompt.
- **Evidence**: EV-77, EV-78, EV-79; `agent/loop.py:976-1020`; `tests/ci/test_compaction.py:80-101,217-227`.
- **Boundary**: the archive is on disk for the run's workspace; it is not automatically uploaded or retained after the caller deletes the workspace.
- **Links**: mechanism→EK-22, causal→EK-24, constraint→EK-25.
- **Layer**: L2.

## EK-24 Compaction failure keeps the full history and retries later

- **Claim**: An empty summary or a non-shrinking projection raises and leaves the original evidence intact; summarizer failures keep full history and are retried on a later request.
- **Detail**: `_compact` raises `ValueError` when the summary is empty or the projected history is not shorter. A summarizer exception is logged and full history is sent; tests assert three retries and the full history. A provider context-window error forces one compaction and a resend; a second rejection ends the run.
- **Why**: a failed summary must not silently drop evidence; retrying is safer than truncating.
- **Evidence**: EV-79, EV-80; `agent/loop.py:976-1020`; `tests/ci/test_compaction.py:230-260`.
- **Boundary**: repeated failures can still hit the provider limit; the runtime may then end with `error`.
- **Links**: causal→EK-22, mechanism→EK-23, contrast→EK-19 (best-effort boundary).
- **Layer**: L2.

## EK-25 Image pruning is batched to preserve the prompt cache

- **Claim**: Images are pruned only after more than 20 tool results carry images, keeping only the latest 2 image-bearing results; the batch is designed not to break the provider cached prefix.
- **Detail**: `PRUNE_IMAGES_AT=20`, `KEEP_IMAGE_RESULTS=2`. `_prune_images()` returns `None` when nothing changes so the history stays byte-identical between prunes. Tests expect old screenshots replaced with `[older screenshot removed]`.
- **Why (code comment)**: pruning on every new image would repeatedly invalidate provider prompt cache; Anthropic also has an image-count limit.
- **Evidence**: EV-28; `agent/loop.py:86-87,1055-1120`; `tests/ci/test_loop.py` image-pruning tests.
- **Boundary**: images in the most recent results are still sent; a user-provided image is not removed by this budget.
- **Links**: mechanism→EK-22, dependency→EK-33, constraint→EK-26.
- **Layer**: L1/L2.

## EK-26 Checkpoint and partial are the recovery channel

- **Claim**: `checkpoint(filename, value, partial=True)` writes a redacted, atomic file and a `partial.json`; a run that ends without delivery falls back to the partial value.
- **Detail**: the worker JSON-serializes the value, redacts it, writes atomically; `partial=True` also writes `STATE/partial.json`. `Session.partial()` reads the JSON `value`, tolerates errors, redacts again. `result.text` uses the partial JSON when status is not `completed`.
- **Why**: a timeout or crash should not lose the last computed rows. The model is told to checkpoint evidence during the run.
- **Evidence**: EV-35, EV-46; `agent/prompt.py` checkpoint instruction; `repl/worker.py:742-751`; `agent/loop.py:166-172,1338-1350`; `tests/ci/test_loop.py:95-115`.
- **Boundary**: partial is only what the model chose to checkpoint; it is not an automatic transcript. A late checkpoint after the budget can still be lost.
- **Links**: mechanism→EK-06, causal→EK-18, constraint→EK-24.
- **Layer**: L2.

## EK-27 Browser lifecycle selects a mode and owns spend/cleanup

- **Claim**: `Browser` selects cloud/chromium/chrome/cdp, starts the appropriate browser/daemon, and stops the cloud browser to end billing.
- **Detail**: default cloud; `Browser('chromium')` starts a fresh local Chromium with a temp profile; `Browser('chrome')` attaches or copies a profile; `cdp_url` connects externally. `_stop_now()` kills watchdog, closes Guard, clears tabs, shuts down daemon, stops local Chrome, PATCHes cloud stop up to 3 times; a still-creating cloud browser is stopped later.
- **Why**: cloud browser time is billable, and local Chrome is a user process; lifecycle must be explicit and idempotent.
- **Evidence**: EV-53 to EV-56; `browser/browser.py:519-760,1074-1130`; `README.md:166-258`.
- **Boundary**: if the process is hard-killed, cloud cleanup depends on the provider; the local debug port has no authentication.
- **Links**: subsystem→EK-15, causal→EK-02, mechanism→EK-01.
- **Layer**: L2.

## EK-28 One key is documented to cover model, cloud browser, CLI, and MCP; code confirms env/stored/fallback ordering

- **Claim**: `BROWSER_USE_API_KEY` wins over a stored key; a gateway model's key can be used as a fallback. The docs describe one key covering multiple surfaces; this run confirmed the resolution order, not every surface's runtime use.
- **Detail**: `browser_use_api_key()` order: env -> stored `browser-harness` auth -> fallback (gateway model key). `clean_api_key()` strips outer whitespace and raises on internal whitespace/control/non-ASCII without printing the key. Auto-login is once per process, 600 s browser wait or device-code expiry.
- **Why**: a single product key reduces configuration; the fallback lets a gateway model key also pay for a cloud browser.
- **Evidence**: EV-81, EV-82, EV-85; `auth.py:76-222`; `agent/service.py:380-450`; README/one-key docs (read as intent).
- **Boundary**: “one key covers CLI/MCP” is a product/doc claim, not a runtime proof in this run. The stored key is in `browser-harness` auth storage; this repository does not encrypt it. The worker can read it with the user's permissions (documented).
- **Links**: mechanism→EK-02, constraint→EK-27, contrast→EK-44.
- **Layer**: L2.

## EK-29 Provider retries are limited to transient failures

- **Claim**: The loop retries rate limit/unavailable/server/connection errors once; provider 200-response failure codes are also transient; non-transient errors end the run as `error`.
- **Detail**: `TRANSIENT` includes `ModelRateLimitError`, `ModelUnavailableError`, `ModelServerError`, `ModelConnectionError`; `TRANSIENT_FAILED_CODES` includes `server_error`, `overloaded`, `request_timeout`, `upstream_stream_failed`, `model_request_failed`. `MODEL_TIMEOUT_S=300`, `STREAM_TIMEOUT_S=930`, `GATEWAY_MAX_TOKENS=16_384`. HTTP retry is only connect/timeout; google-genai retries itself.
- **Why**: retrying a schema error or a bad request just burns budget; transient transport failures are worth one retry.
- **Evidence**: EV-33, EV-34; `agent/loop.py:100-120,426-600`; `tests/ci/test_loop.py` provider-retry tests.
- **Boundary**: one retry is not a guarantee; a provider outage longer than the budget leads to timeout/error.
- **Links**: mechanism→EK-30, causal→EK-34.
- **Layer**: L2.

## EK-30 Status is a transport/control outcome, not a task verdict

- **Claim**: `status` distinguishes `completed`, `timeout`, `max_steps`, `error`; `success` is the agent's own finish verdict; judge is separate; `completed` can still have `success=False`.
- **Detail**: `AgentHistoryList.status` and `success`; `is_successful()` returns the agent verdict. `run()` maps budget timeout to `timeout`, `UsageLimitExceeded` to `max_steps`, tool timeout to `error`, other exceptions to `error`. A forced delivery turn can produce `completed` with `success=False`; `max_steps`/`timeout` may deliver nothing and require `partial`.
- **Why**: conflating “the loop ended” with “the task succeeded” hides partial success and makes evaluation wrong.
- **Evidence**: EV-24, EV-34, EV-35; `agent/views.py:84-151`; `agent/loop.py:1301-1361`.
- **Boundary**: `success` is self-reported and can be wrong; judge is advisory, not a gate.
- **Links**: contrast→EK-21, constraint→EK-19, subsystem→EK-01.
- **Layer**: L2.

## EK-31 Telemetry is schema-minimized and scrubbed, not trusted

- **Claim**: Telemetry sends counts/booleans/enums/names, scrubs task text/secrets/URLs/numbers/credentials, and excludes full messages, but still sends some URLs/error text/output.
- **Detail**: `_config()` sends categories and counts; `_capture()` sends one `agent_event` per run, maps unknown tools to `invalid`, strips URL query, scrubs error/result, excludes `result.messages`. `scrub()` replaces task text, secrets, typed text, URL userinfo/query/fragment, path tokens, home user, email, 6+ digit numbers, credential-like env values, auth headers, provider-masked key fragments. `ANONYMIZED_TELEMETRY` defaults true; disable stops the sender.
- **Why**: telemetry is useful for product improvement but should not become a data exfiltration channel; schema minimization is the first line and scrub the second.
- **Evidence**: EV-86, EV-87, EV-88, EV-89; `config.py:41-70`; `telemetry/stats.py:175-557`; `agent/service.py:742-1000`.
- **Boundary**: result text can still contain page content; error messages can contain provider details; URLs retain host/path tokens. This is minimization, not zero collection.
- **Links**: mechanism→EK-32, constraint→EK-09, contrast→EK-44.
- **Layer**: L2.

## EK-32 Legacy names fail with a migration error instead of silently degrading

- **Claim**: Accessing removed 0.x modules/names raises a unified removed error, and `--mcp` serves the new tool surface.
- **Detail**: `legacy.py` installs a meta_path finder; `removed_names` raises the same error; `Agent.__getattr__` points to removed names. `MIGRATING.md` lists removed planner/DOM/replay/pause/stop/callbacks/script export and the replacements. CLI `--mcp` is deprecated and serves `--cli-mcp`; old 0.x tool names return removed errors.
- **Why**: silent fallback would let a 0.x user believe the old semantics still apply; explicit errors force migration.
- **Evidence**: EV-12, EV-15, EV-90, EV-92; `legacy.py:1-120`; `agent/service.py:1045`; `mcp/cli_mcp.py:509-674`.
- **Boundary**: this is import/name compatibility only; it does not preserve old behavior or old config semantics.
- **Links**: contrast→EK-04, subsystem→EK-33.
- **Layer**: L2.

## EK-33 CLI and MCP are separate surfaces with different authority

- **Claim**: The CLI pipes Python to the harness; MCP executes Python in-process; both expose `browser_exec`/`browser_screenshot`, but only the Agent path has the REPL worker and `sensitive_data`.
- **Detail**: CLI entry is `browser_use.cli:main`; MCP tool definitions are `browser_exec`, `browser_screenshot`, `browser_login`, `browser_feedback`; MCP `_execute()` runs in the server process with a persistent namespace and an exec lock. `--mcp` legacy is deprecated; `--cli-mcp` is the supported MCP surface.
- **Why**: the CLI/MCP are lightweight control surfaces; the Agent path is the full runtime with secrets and per-cell isolation.
- **Evidence**: EV-07, EV-15, EV-92, EV-93; `pyproject.toml:52-55`; `mcp/cli_mcp.py:473-1027`.
- **Boundary**: MCP does not get `sensitive_data`, `type_secret`, worker env allowlist, checkpoint redaction, or a process group. It is not the same security model.
- **Links**: contrast→EK-16, contrast→EK-09, subsystem→EK-01.
- **Layer**: L2.

## EK-34 Tests encode concrete contracts, not just coverage

- **Claim**: The CI tests are executable specs for environment filtering, secret redaction, Guard failure/reconnect, delivery repair, last-request delivery, judge isolation, compaction evidence, and timeout recovery.
- **Detail**: named tests assert `worker_env` has no credentials, a harness call in a cell fails closed, the trust model admits worker access to parent keys, secrets are redacted from returned content/checkpoint/partial, a scoped secret is refused across origin, Guard start failure raises, Guard reconnects, attached Chrome keeps user tabs, placeholder/missing-file delivery gets one repair, a last-request repair is delivered, a split delivery is rejected, a failed judge leaves status completed, compaction keeps tool pairs and user messages, the archive is redacted, summarizer failure keeps history, timeout returns partial, and cell timeout restarts the worker.
- **Why**: these tests mark the boundary between claimed and verified behavior; they are more specific than README claims.
- **Evidence**: EV-08, EV-36, EV-52, EV-68, EV-75, EV-78, EV-80, EV-95.
- **Boundary**: this run did not execute them; collection failed on missing `pydantic_ai`. Test names and bodies are evidence of intended contracts, not a pass result.
- **Links**: subsystem→EK-19, subsystem→EK-21, subsystem→EK-23, subsystem→EK-06.
- **Layer**: L1/L2.

## EK-35 Bundled skill install overwrites existing `SKILL.md` by default

- **Claim**: `browser-use skill install` writes bundled skill files into agent skill directories and overwrites an existing `SKILL.md` by default; `--force` is accepted for compatibility.
- **Detail**: `skills/install.py` has `SKILL_NAME='browser-use'`, enumerates install targets, loads the packaged skill text, and writes it; `--force` is accepted but not required to overwrite.
- **Why**: an installer that silently overwrites a user's edited skill file is dangerous; the default behavior is documented in the code but not a safe merge.
- **Evidence**: EV-91; `skills/install.py:10-120`.
- **Boundary**: this is a tooling side effect, not core Agent behavior. It only matters when the user runs the install command.
- **Links**: subsystem→EK-33, contrast→EK-32.
- **Layer**: L1.

## EK graph summary

| Cluster | EKs | Edge pattern |
|---|---|---|
| Run lifecycle and spend | EK-01, EK-02, EK-03, EK-27, EK-28 | mechanism/causal |
| Execution substrate | EK-04, EK-05, EK-06, EK-07, EK-15, EK-16 | subsystem/contrast |
| Secrets and authority | EK-08, EK-09, EK-10, EK-11, EK-12, EK-13, EK-14 | mechanism/constraint |
| Delivery and verification | EK-18, EK-19, EK-20, EK-21, EK-30 | causal/contrast |
| Context and recovery | EK-22, EK-23, EK-24, EK-25, EK-26 | dependency/constraint |
| Surfaces and compatibility | EK-29, EK-31, EK-32, EK-33, EK-34, EK-35 | contrast/subsystem |

No EK is isolated: each has at least one incoming or outgoing edge.

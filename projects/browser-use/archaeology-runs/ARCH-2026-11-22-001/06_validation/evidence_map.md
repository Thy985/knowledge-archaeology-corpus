# Evidence Map — ARCH-2026-11-22-001

> Each EV is a repository-grounded observation. “Read” means source code or docs were read this run; “test-code” means the test body or test name was read but not executed. No EV is a runtime pass/fail result.

## Repository and build

| ID | Observation | Source |
|---|---|---|
| EV-01 | Package name `browser-use`, version `1.0.8rc1`, description “Browser agent on Pydantic AI: one persistent Python REPL on raw CDP”, license MIT, Python `>=3.11,<4.0` | `pyproject.toml:1-17` |
| EV-02 | HEAD is `e682e0dc02d77a307a5b2aedf17142c2cd69af5b`, branch `main`, commit time `2026-11-20T14:48:08-08:00`, subject `1.0.8rc1: cached judge, SSH sign-in, step-limit answers and 3 fixes (#6489)`; tag `1.0.8rc1` points at HEAD | `git log -1`, `git tag --points-at HEAD` |
| EV-03 | Diff from prior `d8110c5` to HEAD is 546 files changed, 37,339 insertions, 135,830 deletions | `git diff --stat d8110c5..HEAD` |
| EV-04 | 146 tracked files; 75 Python files; 35,428 Python lines; engine 16,768 lines; tests 18,492 lines | `git ls-files`, line counts |
| EV-05 | `LICENSE` is MIT, copyright 2024 Gregor Zunic, both at HEAD and at `d8110c5` | `LICENSE:1-8`, `git show d8110c5:LICENSE` |
| EV-06 | Exact pins: `browser-harness==0.1.13`, `pydantic==2.14.3`, `pydantic-ai-slim[...]==2.66.0`, `google-genai==2.34.0`, `genai-prices==0.2.1`, `idna==3.20`, `pillow==12.3.0`, `python-dotenv==1.2.4`, `uuid7==0.1.0`, `mcp==2.3.0` | `pyproject.toml:18-36` |
| EV-07 | Python entry points: `browser-use`/`browseruse` -> `browser_use.cli:main`; `browser-use-tui` deprecated alias | `pyproject.toml:52-55` |
| EV-08 | Pytest config: timeout 300, asyncio auto, `chrome` marker, testpaths tests, strict markers | `pyproject.toml` pytest section |

## Public behavior and docs

| ID | Observation | Source |
|---|---|---|
| EV-09 | Main chain documented as `Agent(task, llm, browser).run()` -> model/browser -> Pydantic AI loop -> `python(code)` -> REPL worker -> browser-harness helpers + raw CDP -> `AgentHistoryList` | `AGENTS.md` main architecture section |
| EV-10 | Default model is `bu-4.0`; default browser is Browser Use Cloud; `Browser('chromium')` is a fresh local Chromium; `Browser('chrome')` attaches to the user's Chrome; `Browser(cdp_url=...)` uses explicit CDP | `README.md:134,166,232-258`; `agent/service.py:244-358`; `browser/browser.py:519-530` |
| EV-11 | Run defaults `max_steps=100`, `timeout=1700`; `max_steps` belongs to `run()`, not the constructor | `agent/service.py:452-475`; `README.md` |
| EV-12 | `MIGRATING.md` says 0.x planner, DOM engine tuning, replay, pause/resume/stop, per-step callbacks, `save_as_playwright_script` are removed; replacements include `result.messages` and `asyncio.Task.cancel()` | `MIGRATING.md` migration sections |
| EV-13 | Domain restrictions are explicitly “not a sandbox”; mode table lists cloud/chromium/chrome-profile/cdp with layer 2, and attached `Browser('chrome')` with only layer 1 + layer 3 | `MIGRATING.md:180-249` |
| EV-14 | Secrets section says the worker can still read files, environment, screenshots, live view, recordings; local debug port has no authentication; shared machines should use container/VM/Cloud | `MIGRATING.md:335-404` |
| EV-15 | 0.x MCP tool names are removed; `--mcp` is deprecated and serves the new `browser_exec`/`browser_screenshot` surface | `MIGRATING.md:298-333` |

## Agent lifecycle

| ID | Observation | Source |
|---|---|---|
| EV-16 | `Agent` class at `:221`; constructor validates task, booleans, positive ints, reasoning enum, browser/browser_session exclusivity, Tools type, sensitive_data dict, output schema serializability | `agent/service.py:221-358` |
| EV-17 | Constructor rejects unknown 0.x knobs with `TypeError`; some tuning knobs only warn; `use_vision=False` is preserved | `agent/service.py:244-358` |
| EV-18 | `llm=None` resolves `bu-4.0`; `DEFAULT_LLM` is only warned about in 1.0 | `agent/service.py:244-358`; `llm/__init__.py` |
| EV-19 | `_build()` calls `resolve_model()` before browser start; resolves provider prices; detects gateway; passes gateway key to browser; builds prompts and agent | `agent/service.py:380-450` |
| EV-20 | `run()` enforces one active run per Agent and one active Agent per Browser; validates `max_steps`/`timeout` | `agent/service.py:452-474` |
| EV-21 | `_run()` order: `_check_task` -> `raise_fd_limit` -> `_build` -> `RunRecorder` -> telemetry -> preflight -> workspace dirs -> `browser.start()` -> `Repl` -> `Session` -> loop -> history/final page -> shutdown -> judge | `agent/service.py:475-627,699-718` |
| EV-22 | `BrowserConfigError` / `MissingAPIKeyError` are re-raised at start; other browser start failures become `status='error'` with `steps=0` and still stop the browser | `agent/service.py:520-627` |
| EV-23 | `close(keep_alive=True)` closes REPL but leaves Browser; otherwise `_shutdown()` always stops browser and downloads | `agent/service.py:699-718` |
| EV-24 | `AgentHistoryList.status` is `completed|timeout|max_steps|error`; `success=None` unless completed; `is_successful()` returns the agent verdict, not the judge | `agent/views.py:84-151` |

## Pydantic AI loop and delivery

| ID | Observation | Source |
|---|---|---|
| EV-25 | `Session` defaults `timeout_s=1700`, `max_steps=1000` (Agent passes 100), `cell_timeout_s=120`, `vision=True`; tracks delivery/finalizing/split/budget/error/checkpoint fields | `agent/loop.py:127-156` |
| EV-26 | `delivery_at()` is `timeout_s - max(0.1*timeout_s, min(30, 0.5*timeout_s))`; `time_left()` stops at `delivery_at` before finalizing | `agent/loop.py:157-174` |
| EV-27 | `python()` executes one cell at a time, computes remaining time to delivery, returns `NO_TIME` at zero, calls `repl.call('exec', ...)`, returns screenshots as `BinaryContent`, suppresses images when `use_vision=False` | `agent/loop.py:175-225` |
| EV-28 | Image budget: `PRUNE_IMAGES_AT=20`, `KEEP_IMAGE_RESULTS=2`; pruning is batched to keep provider prompt cache stable | `agent/loop.py:86-87,1055-1120` |
| EV-29 | Delivery tools are `finish`, `finish_from_repl`, `on_text`; `DELIVER_NOW` tells the model to deliver immediately near the budget; `DELIVERY_REPAIR` gives one repair turn | `agent/loop.py:88-98,336-424` |
| EV-30 | `check_split_delivery()` rejects a response that contains a delivery call plus another tool call, or two delivery calls; comment says once per run, implementation stores the request step and only skips the same step; delivery-only turn exempt | `agent/loop.py:139,226-251` |
| EV-31 | `check_delivery()` checks placeholder results and missing workspace files; failures raise `ModelRetry`; at `ctx.run_step > max_steps` it only warns and delivers | `agent/loop.py:252-335` |
| EV-32 | `finish` and `finish_from_repl` validate against `output_model_schema`, coerce `str`, redact results, and set success | `agent/loop.py:336-363` |
| EV-33 | `build_agent()` selects Responses/Chat/Google/Anthropic settings, adds `BrowserLoop`, `Compaction`/`LazyCompaction`, `StreamResponses`, `Recording`; tool retries `TOOL_RETRIES=1_000_000`; `python` is sequential | `agent/loop.py:1122-1300`; `tools.py:34-38` |
| EV-34 | `run()` wraps the agent in `asyncio.timeout`; maps budget timeout to `timeout`, tool timeout to `error`, `UsageLimitExceeded` to `max_steps`, other exceptions to `error`; does not raise after start | `agent/loop.py:1301-1361` |
| EV-35 | When status is not `completed`, output falls back to `partial.json` or last model text; usage/cost covers this run and compaction summaries | `agent/loop.py:1338-1361` |
| EV-36 | Tests: cell timeout kills worker, saves file, variables lost; timeout returns partial JSON; placeholder/missing-file delivery gets one repair; last-request delivery needing repair is delivered; split delivery is rejected; structured text triggers one repair | `tests/ci/test_loop.py:71-115,445-481,710-819,1194-1227` |

## REPL execution plane

| ID | Observation | Source |
|---|---|---|
| EV-37 | Worker connects back to `127.0.0.1` ephemeral port with a random token; parent verifies with `hmac.compare_digest`; no inherited fd | `repl/session.py` module docstring and `_start` |
| EV-38 | `worker_env()` allows a fixed list plus `LC_`/`BU_`/`BH_` prefixes; explicitly denies `BU_CDP_WS`, `BU_CDP_URL`, `BU_MODE`, `BU_HIGHLIGHT_ACTIONS`; removes names matching key/token/secret/password/credential/auth; keeps proxy/CA variables | `repl/session.py:1-180` |
| EV-39 | `_start()` closes an old worker, creates a server and token, launches `python -P -m browser_use.repl.worker` with `cwd=workspace`, `stdin=DEVNULL`, `start_new_session=True`; Windows uses a Job Object; sets `BU_REPL_PORT/TOKEN`, `BH_REQUIRE_EXISTING_DAEMON=1`, `BU_CDP_WS` sentinel, `BH_AGENT_WORKSPACE`, `BU_REPL_PARENT_PID`, `BH_TELEMETRY=0`, `BU_MODEL_CODE=1` | `repl/session.py:357-442` |
| EV-40 | `Repl.call()` serializes with an `asyncio.Lock`; `BrowserConnectionLost` triggers `reattach`; a changed daemon closes Repl and reports lost tabs/logins/variables; failed reattach sets `browser_gone=True` | `repl/session.py:460-523` |
| EV-41 | `_call()` timeout/disconnect closes worker, reads partial `cells/<n>.txt`, returns an error saying actions may already have happened; error type `CellTimeout`/`WorkerExited` | `repl/session.py:524-568` |
| EV-42 | `close()` kills the worker process group, waits 5 s; `kill_workers()` at exit/SIGTERM kills all live workers | `repl/session.py` close/kill_workers |
| EV-43 | `worker._install()` imports `browser_harness.helpers` first (which loads `.env` and agent helpers), then clears and restores only the allowlisted environment; import side effects cannot be undone | `repl/worker.py:494-560` |
| EV-44 | Worker wraps `helpers._send`, rejects IPC requests over `MAX_IPC_REQUEST=65_504` bytes, maps connection errors to `BrowserConnectionLost`, tracks stats | `repl/worker.py` install/send wrapper |
| EV-45 | `cell` loop clears images, writes to `cells/<n>.txt`, returns to the agent tab, joins the output pump, reads up to `MAX_OUTPUT_CHARS+1`, clips and reports the full file path | `repl/worker.py` main/cell loop |
| EV-46 | `checkpoint(filename, value, partial=False)` JSON-serializes, redacts, atomically writes; `partial=True` also writes `state/partial.json` | `repl/worker.py:742-751` |
| EV-47 | `screenshot()` writes JPEG q70 to `state/screenshots` and adds the image to the model; `show_image()` reads png/jpeg/webp/gif; neither redacts image pixels | `repl/worker.py:728-741` |
| EV-48 | `type_secret()` finds a secret by name, supports TOTP for `bu_2fa_code`, focuses the field, selects existing text, resolves the actual recipient through the frame tree in an isolated world, and fails closed when the recipient cannot be confirmed | `repl/worker.py:807-900` |
| EV-49 | Scoped secret is only typed when the current URL matches the secret's domain pattern; otherwise `PermissionError` | `repl/worker.py` type_secret scoped branch |
| EV-50 | Redaction forms include plain, JSON, repr, HTML, case variants, reversed, base64 (including core inside a longer string), hex, URL-encoded; replaced longest first; short secrets are whole-word only | `repl/session.py` secret_forms/redact |
| EV-51 | Worker stdout/stderr and between-cell background output pass through a redacting pump; arbitrary files written by the worker with `open()` are outside that pump | `repl/worker.py:1113-1134`; `MIGRATING.md:378-393` |
| EV-52 | Tests: env has no credentials; harness call in a cell fails closed; trust model says worker can reach parent keys; secrets redacted from output/error/output file/checkpoint/partial; scoped secret refused across origin; cell timeout kills child process | `tests/ci/test_repl.py` named tests |

## Browser plane

| ID | Observation | Source |
|---|---|---|
| EV-53 | `_kind()` resolves `cloud|chromium|chrome|cdp`; `Browser()` defaults cloud; `local=True`/`use_cloud=False` selects chromium; mutually exclusive parameters raise | `browser/browser.py:519-620` |
| EV-54 | `Browser('chrome')` without profile ensures only a shared daemon and does not create a browser; with profile it copies the profile and starts an owned Chromium; `Browser('chromium')` starts fresh with a temp profile | `browser/browser.py:620-760`; `browser/local.py:260-332` |
| EV-55 | `start()` is idempotent; keep-alive reattaches; a dead daemon restarts; a gone cloud browser is replaced during keep-alive, but mid-run reattach of a different browser raises | `browser/browser.py:701-800` |
| EV-56 | `_stop_now()` kills watchdog, closes Guard, clears tabs file, shuts down daemon, stops local Chrome, PATCHes cloud stop up to 3 times; a cloud browser still creating is stopped later | `browser/browser.py:1074-1130` |
| EV-57 | `after_cell()` runs Guard notices plus `sweep()` when domains exist; sweep failure is debug-only and does not fail the cell | `browser/browser.py:1061-1073` |
| EV-58 | `Browser('chrome')` `_attached()` is true; `_touched_tabs()` covers tabs file, current tab, popups; `sweep(only=agent_tabs)` leaves user tabs alone; pre-existing disallowed tabs are exempt while the same URL remains | `browser/browser.py:1042-1060`; `domains.py:442-475` |
| EV-59 | Local Chrome args use a dedicated `--user-data-dir`, `--remote-debugging-port=0`, `--no-first-run`, `--no-default-browser-check`; debug port has no authentication | `browser/local.py:332-384`; `MIGRATING.md:395-404` |
| EV-60 | `require_debug_port()` requires a real DevToolsActivePort or explicit CDP URL, after browser-harness once attached to any process on 9222/9223 | `browser/local.py:104-180` |

## Domain guard

| ID | Observation | Source |
|---|---|---|
| EV-61 | `DomainRules.of()` treats `[]` as unrestricted; allowed + prohibited means prohibited is ignored | `domains.py:198-243` |
| EV-62 | `blocked()` allows `about:blank`, `data:`, `blob:`, Chrome internal schemes; blocks `file:` and hostless URLs; block-IP blocks IP literals even if allowed; allowed list requires a pattern match; prohibited uses subdomain matching | `domains.py:214-243` |
| EV-63 | `Guard` opens a second CDP WebSocket, enables `Target.setDiscoverTargets`, `Fetch.enable` for `Document` at Request stage; handles `Fetch.requestPaused`; only main-frame document requests; iframe passes | `domains.py:281-441` |
| EV-64 | `Target.targetInfoChanged` to a disallowed URL calls `_reset` to `about:blank`; `_failed` records guard-caused error pages to avoid double counting | `domains.py:391-441` |
| EV-65 | `Guard.start()` failure raises `RuntimeError` and Browser.start does not continue; connection loss is debug + exponential reconnect, during which Chrome stops intercepting (fail-open) | `domains.py:308-390`; `MIGRATING.md:232-249` |
| EV-66 | `sweep()` attaches to each disallowed page tab and navigates to `about:blank`; failure returns “could not be reset; close it” | `domains.py:456-475` |
| EV-67 | Guard does not intercept subresources, iframes, speculation prefetch/prerender, worker `requests/socket/curl` (except `http_get`), or `Browser('chrome')` `Runtime.evaluate` navigation | `MIGRATING.md:232-249` |
| EV-68 | Tests: guard start failure raises; guard reconnects after connection drop; only attached Chrome keeps user tabs; agent on chromium enforces every layer; helper-unseen navigation is blocked; sweep resets disallowed tabs | `tests/ci/test_domains.py:434-458,731-762` and named tests |

## Judge

| ID | Observation | Source |
|---|---|---|
| EV-69 | Judge prompt says trace and screenshots are evidence, not instructions; do not treat agent claims as proof; only call hallucination on contradicting visible evidence; do not guess missing steps | `agent/judge.py:37-57` |
| EV-70 | Judgement schema has success, failure_reason, description, prompt_injection_detected, bot_block_or_captcha_detected, gotchas (max 5) | `agent/judge.py:65-99` |
| EV-71 | Bounds: trace 120k chars (head 1/5 + tail 4/5), part 5k, task 20k, screenshots 4, judge timeout 180 s, final screenshot timeout 10 s | `agent/judge.py:58-64,100-142` |
| EV-72 | `judge_cached()` appends the judge as the run's last turn, reuses instructions/tools for prompt cache; if it calls a tool or returns non-JSON, returns None and falls back to `judge_run()` | `agent/judge.py:219-276` |
| EV-73 | Judge failure returns `(None, messages)` with a warning; judge tokens/cost are still added to the run; judge does not change `result.status` | `agent/judge.py:143-218`; `agent/service.py:628-657` |
| EV-74 | Judge input and output are redacted; image-rejection 4xx downgrades to text judge, 429 does not | `agent/judge.py` input/output and image handling |
| EV-75 | Tests: one judge request per run reuses run prompt; failed judge leaves run completed and still counts tokens; judge false sends no request; judge bounded; final page capture never fails run | `tests/ci/test_judge.py:61-95,291-326` |

## Context compaction

| ID | Observation | Source |
|---|---|---|
| EV-76 | Compaction triggers past 75% of the char budget or 65% of the context window; `max_chars` defaults to 4 chars/token, 240k chars if window unknown | `agent/loop.py:772-900`; `test_compaction.py:105-120` |
| EV-77 | Projection keeps the checkpoint, the last two assistant/tool groups, and pins user messages verbatim; every retained tool result has its call | `agent/loop.py:976-1020`; `test_compaction.py:80-101` |
| EV-78 | Archive JSON has version, summary, `covered_messages`, covered `messages`; written 0600 under `.browser-use/context`; redacted and images replaced with `[Image omitted from text archive]` | `agent/loop.py:1010-1016`; `test_compaction.py:217-227` |
| EV-79 | If the summary is empty or the projected history is not smaller, compaction raises and retains the original evidence | `agent/loop.py:1008-1019` |
| EV-80 | Summarizer failure keeps full history and is retried on a later request; a context-window provider error forces one compaction and resends the request | `test_compaction.py:230-260` |

## Auth, LLM, telemetry, compatibility

| ID | Observation | Source |
|---|---|---|
| EV-81 | Auto-login: interactive desktop -> browser; SSH/Linux no display + tty -> device code; otherwise None; only main thread and an otherwise idle loop can block; one login per process; 600 s browser wait or device-code expiry; `BROWSER_USE_AUTO_LOGIN=false`, CI, GitHub Actions, ipykernel disable | `auth.py:76-165` |
| EV-82 | Key order: `BROWSER_USE_API_KEY` -> stored browser-harness key -> gateway fallback; key is stripped; internal whitespace/control/non-ASCII raises without printing the key | `auth.py:167-222` |
| EV-83 | `resolve_model()` raises missing key/extra before browser start; Pydantic AI `UserError` is rewritten with env guidance; bare model names suggest 1.0 syntax rather than guessing a provider | `llm/__init__.py:532-619` |
| EV-84 | `provider_preflight()` only probes official OpenAI/Anthropic/Google endpoints; invalid key/404 fails before browser; 403, proxy, unknown client, non-invalid 401 do not block | `llm/__init__.py:461-515` |
| EV-85 | `gateway_llm()` recognizes `ChatBrowserUse` or `ChatOpenAI(base_url=gateway)`; gateway key can be passed to Browser | `llm/__init__.py:516-531`; `agent/service.py:380-450` |
| EV-86 | Telemetry defaults: `ANONYMIZED_TELEMETRY='true'`; empty string also true; config dir `$XDG_CONFIG_HOME/browseruse`; disable stops thread/network/file except explicit feedback | `config.py:41-70`; `telemetry/service.py` |
| EV-87 | `scrub()` replaces task text, secrets, typed text, URL userinfo/query/fragment, path tokens, home user, email, 6+ digit numbers, credential-like env values, auth headers, provider-masked key fragments | `telemetry/stats.py:175-557` |
| EV-88 | `_config()` sends model/provider, reasoning, mode, max_steps, timeout, structured output, custom tool names, secret/domain counts, browser mode, cdp host category, keep_alive, cloud profile/proxy/timeout, live view, restrictions counts, search, vision, highlight, judge, captcha, headless, profile, user data dir, browser exe category, allowed/prohibited counts, block_ip, gateway, model_api, compaction mode, context window, sync, key source | `agent/service.py:742-875` |
| EV-89 | `_capture()` sends one agent_event per run, collects mutable fields, maps unknown tool names to `invalid`, strips URL query, scrubs error/result, excludes `result.messages` | `agent/service.py:876-1000` |
| EV-90 | `legacy.py` installs a meta_path finder that raises unified removed-name errors for 0.x modules/names | `legacy.py:1-120`; `agent/service.py:1045` |
| EV-91 | `browser_use/skills/install.py` default-installs and overwrites existing `SKILL.md`; `--force` is accepted for compatibility | `skills/install.py:10-120` |
| EV-92 | MCP tool definitions: `browser_exec`, `browser_screenshot`, `browser_login`, `browser_feedback`; old 0.x names return removed errors pointing to `browser_exec` | `mcp/cli_mcp.py:509-674` |
| EV-93 | MCP `_execute()` runs `exec(code, ns)` in the server process, captures stdout/stderr, uses an exec lock; `browser_exec` returns within `CALL_SECONDS`, then code continues in a separate thread and the next empty call collects output | `mcp/cli_mcp.py:675-774,880-1027` |
| EV-94 | MCP cloud path calls `start_remote_daemon()`; cloud admin calls are refused when domains are configured because cloud can reach any site; MCP server stops cloud browsers it started on exit | `mcp/cli_mcp.py:463-472,856-879,1028-1100` |
| EV-95 | Tests: `test_compat.py`, `test_missing_key.py`, `test_login.py`, `test_llm.py`, `test_telemetry.py`, `test_browser_use_skill_install_docs.py` exist and cover compatibility/key/login/LLM/telemetry/skill install behavior; not executed this run | `tests/ci/` listing |

## Evidence limitations

| ID | Limitation |
|---|---|
| EV-L1 | No dependency install, no `pytest`, no Chrome launch. All test references are static test-code observations. |
| EV-L2 | `browser-harness==0.1.13` internals were not read; helper behavior is treated as an external contract. |
| EV-L3 | No LLM key, cloud key, cloud browser, live view, CAPTCHA, proxy, or SSH device-code flow was exercised. |
| EV-L4 | `1.0.8rc1` is a release candidate; no claim about a stable 1.0.8 GA is made. |
| EV-L5 | README benchmark references were not converted to numeric claims. |

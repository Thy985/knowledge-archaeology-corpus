# 01 Project Layer — browser-use 1.0.8

## 1. What this project is

`browser-use` 1.0.8 is a browser agent whose action space is **model-written Python in a persistent REPL**, not a typed DOM action schema. The model receives a task, writes Python cells, calls `browser-harness` helpers or raw CDP, reads stdout/images, and ends with `finish`, `finish_from_repl`, or plain text. The repository describes it as “one persistent Python REPL on raw CDP, cloud or local Chrome” (`pyproject.toml`), and the code path matches.

The 1.0 rewrite is not a refactor of 0.13.10. It deletes the intermediate representation — DOM serialization, action selection, per-action validation — and replaces it with code execution. This changes the governance surface: there is no action allowlist to inspect; authority comes from the process, the browser attach mode, domain rules, secret routing, and budgets.

## 2. Architecture in one page

```
User task
  │
  ▼
Agent.run()                       browser_use/agent/service.py
  ├─ resolve_model(llm)           default bu-4.0 on the Browser Use gateway
  ├─ Browser.start()              cloud | chromium | chrome | cdp
  │    └─ browser-harness daemon  owns the browser connection
  ├─ Repl(workspace, env)         parent-side control plane (loopback + token)
  │    └─ python -P -m browser_use.repl.worker
  │         ├─ persistent namespace
  │         ├─ browser-harness helpers + raw cdp()
  │         ├─ screenshot / checkpoint / type_secret / solve_captcha
  │         └─ search / downloads / feedback proxied to parent
  ├─ Pydantic AI loop             browser_use/agent/loop.py
  │    ├─ python(code) tool       one cell at a time
  │    ├─ delivery-only turn      near budget: finish / finish_from_repl
  │    ├─ LazyCompaction          server-side, Responses models
  │    └─ Compaction              client-side summary checkpoint
  ├─ judge (optional)             same model, separate request
  └─ AgentHistoryList             status / text / output / success / judgement
```

## 3. The 1.0 delta against the 0.13.10 baseline

| Dimension | 0.13.10 baseline | 1.0.8 | Evidence |
|---|---|---|---|
| Action space | DOM-serialized elements + typed actions | Python code + browser-harness helpers + raw CDP | `AGENTS.md`; `agent/loop.py:178-220`; `agent/prompt.py` |
| Intermediate representation | page serializer, element IDs, action schemas | none; the model prints what it wants to see | no serializer in tree; `agent/prompt.py` |
| Execution unit | one action step | one Python cell in a persistent namespace | `repl/session.py:304-560`; `repl/worker.py:1142-1200` |
| Isolation of a hung step | watchdog / loop detector | kill worker process group, restart; browser survives | `repl/session.py:524-560`; `local.py:592-640` |
| Loop control | step budget + duplicate/loop detection | request budget + delivery-only turn; no duplicate-action detector | `agent/loop.py:128-176`, `426-520` |
| Context management | message compaction policy | server-side compaction or client-side summary checkpoint + 2-image retention | `agent/loop.py:719-900`, `1055-1100` |
| Delivery | `done()` text / final result | `finish` / `finish_from_repl` / `on_text` with validation | `agent/loop.py:226-405` |
| Verdict | agent `is_successful` + optional judge | agent `success` + separate `judgement`; `is_successful()` reads only agent verdict | `agent/views.py:105-150`; `agent/judge.py` |
| Domain policy | security watchdog that observed navigations | helper refusal + `Guard` main-frame Fetch + after-cell sweep | `domains.py:1-24`, `281-455` |
| Extensibility | action registry with injected context | function tools beside `python`; no injected browser context | `tools.py:1-30`, `230-270` |
| Compatibility | native 0.x API | 0.x facade over 1.0 engine; removed names raise with alternatives | `agent/service.py:81-180`; `legacy.py` |

## 4. Lifecycle model

### 4.1 Browser lifecycle

`Browser()` defaults to a Browser Use Cloud browser. `Browser('chromium')` launches a fresh local Chrome/Chromium/Edge with a temporary profile. `Browser('chrome')` attaches to the user’s already-open Chrome through the shared browser-harness daemon. `Browser('chrome', profile='Work')` launches a new Chrome on a temporary copy of that profile.

`Agent.run()` stops the browser at the end unless `keep_alive=True`; cloud browsers are stopped on process exit, SIGTERM, SIGHUP, and after the server-side cloud timeout. `_stop_now()` stops the daemon, the Guard, the watchdog, and the cloud browser; a failed cloud stop leaves the browser owned for a later retry (`browser_use/browser/browser.py:1036-1125`).

### 4.2 REPL lifecycle

1. Parent creates a loopback server on an ephemeral port and a 32-byte hex token.
2. Parent spawns `python -P -m browser_use.repl.worker` in a new process group (POSIX) or a kill-on-close job object (Windows), with an allowlisted environment.
3. Worker connects, sends the token, receives secrets/search/downloads flags, and reports `ready`.
4. Each cell is sent with a monotonic `n`; the parent answers only messages with the matching `n`.
5. On timeout or worker exit, the parent kills the process group/job, returns the redacted partial output from `cells/<n>.txt`, and starts a new worker on the next call. Python variables are lost; files, checkpoints, and browser state survive.

### 4.3 Run lifecycle

`Agent.run()`:
1. rejects overlapping runs on the same Agent or Browser;
2. resolves the model and key before creating a browser;
3. preflights gateway/model/search; a bad key or unknown model never creates a cloud browser;
4. creates the workspace, browser, REPL, and `Session`;
5. runs the Pydantic AI loop under a `timeout_s` budget and `UsageLimits(request_limit=max_steps+1)`;
6. constructs a judge prefix and final screenshot if judging;
7. stops the browser/REPL in `finally`;
8. judges the result (if enabled and steps exist), then records telemetry;
9. returns `AgentHistoryList`.

The judge happens **after** shutdown, using a final-page screenshot captured before shutdown (`agent/service.py:475-627`).

## 5. Control surfaces

### 5.1 Model-facing tools

Only three built-ins exist: `python`, `finish`, `finish_from_repl`; `on_text` is an output mode, not a tool. Custom `Tools().action` functions are additional function tools. There are no browser action tools, no `click`, `type`, `scroll` schemas. The model must write code.

### 5.2 Human/operator surfaces

- CLI: `browser-use`, `browser-use-tui`, `browser-use --doctor`, `browser-use login`, `browser-use skill`, `browser-use --cli-mcp`.
- MCP server: `browser_exec`, `browser_screenshot`, `browser_login`, `browser_feedback`; old 0.x tool names return a migration error.
- Python API: `Agent`, `Browser`, `Tools`, `AgentHistoryList`.

### 5.3 Limits and budgets

- `Agent.run(max_steps=100, timeout=1700)`: request limit and loop budget.
- `Session(max_steps=1000, timeout_s=1700, cell_timeout_s=120)`: internal defaults; `Agent.run` passes 100 by default.
- model request timeout 300 s; streamed Responses first-event 300 s, total 930 s; judge 180 s.
- delivery turn starts at 90% of budget or at most 30 s before the end.
- cell timeout is `min(cell_timeout_s + ultrafast grace, time left before delivery)`.

## 6. Failure and degradation model

| Failure | Behavior | User-visible consequence |
|---|---|---|
| Cell hangs | kill worker process group, restart | Python variables lost; files/checkpoints/browser survive; partial output returned |
| Worker dies | new worker on next call | same as above |
| Browser connection lost | reattach same daemon if possible; otherwise new browser and REPL restart | tabs/logins/page state may be lost; if reattach fails, next turn is delivery-only |
| Model request timeout | retry once if transient, capped before delivery | run continues or ends at budget |
| Context overflow | compact once, retry; second failure ends run | history summarized; evidence archive retained |
| Judge fails | warning only, `judgement=None` | run result unchanged; judge cost still counted |
| Delivery missing file/placeholder | one repair turn, then warning and deliver | possibly incomplete artifact, but not silently rewritten |
| Cloud create timeout | late browser stopped when it lands | no orphan billing if stop succeeds |

## 7. Security and authority model (initial, not final)

The project’s own trust model is explicit:

- The REPL worker runs model-written Python **as the user**. It can read any file the user can read, the parent’s `.env`, `/proc/<pid>/environ` on Linux, and `ps -E` on macOS. Redaction is a display/output boundary, not a sandbox.
- `sensitive_data` values are routed through a control channel and `type_secret`; the model sees names, not values. Domain-scoped secrets only type into a focused frame whose URL matches. TOTP seeds are converted to current codes.
- The worker environment is allowlisted and strips secret-looking names, but the worker can still reach the parent’s keys through the filesystem/process table.
- Local Chrome debugging ports bind to loopback and have no authentication. Any local account can attach while the browser runs; `keep_alive=True` extends that window. The docs recommend container/VM/cloud on shared machines.
- Domain rules are a guard for an aligned model, not a sandbox: subresources, iframes, and the REPL’s own Python networking are not filtered.
- Cloud live-view URLs are bearer capabilities: anyone with the link can control the browser. The Browser Use API key is also a bearer capability for cloud browsers and the default model.

## 8. Test architecture

- 25 test files, 429 `def test_` functions, 18,492 test lines; 964 non-Chrome tests pass locally.
- `tests/ci` uses a scripted `FunctionModel` to exercise model/loop behavior without a provider.
- 140 tests are marked `chrome` and require a real headless Chrome; CI runs them in a separate ubuntu job.
- Package CI installs the wheel on three OSes and two Python versions and verifies the CLI.
- Docker CI builds per architecture and runs a scripted smoke test.

## 9. Open questions carried into archaeology

1. What exactly replaced the 0.x action-validation layer, and what governance disappeared with it?
2. Is the persistent REPL a process boundary, a security boundary, or only a failure-isolation boundary?
3. Does local Chrome attachment create an ambient local control plane?
4. Is the judge an independent check, or a second opinion from the same model?
5. Which 0.13.10 knowledge objects survive the architecture change, which are downgraded to “0.x implementation”, and which must be retired?

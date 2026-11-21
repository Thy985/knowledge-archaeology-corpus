# 01 · Project Layer — browser-use 1.0.8rc1

> Project map only. Facts here are traceable to the repository at `e682e0dc02d77a307a5b2aedf17142c2cd69af5b`. No behavior was executed this run.

## 1. What the project is

`browser-use` is a Python library and CLI for LLM-driven browser automation. The 1.0 line describes itself as “Browser agent on Pydantic AI: one persistent Python REPL on raw CDP, cloud or local Chrome” (`pyproject.toml:3`). The model does not choose from a fixed DOM action menu; it writes Python in a persistent REPL, where `browser-harness` helpers and a raw `cdp()` function are available.

The same package also exposes:
- a CLI (`browser-use`, `browseruse`) that pipes Python to the harness;
- a stdio MCP server (`--cli-mcp`, plus deprecated `--mcp` compatibility);
- a cloud browser and gateway-model integration under one Browser Use API key;
- an install command for bundled agent skills.

## 2. Repository shape

```
browser_use/
  agent/        Agent lifecycle, Pydantic AI loop, judge, prompt, views
  browser/      Browser lifecycle, cloud/local/CDP, local Chromium launch
  repl/         REPL parent (session.py) and worker (worker.py)
  llm/          Model resolution, gateway detection, provider preflight
  mcp/          CLI MCP server
  telemetry/    PostHog sender, scrub, RunStats, RunRecorder
  skills/       bundled skill assets and installer
  ax.py         accessibility-tree helpers for ultrafast mode
  cli.py        CLI entry and login
  domains.py    DomainRules and CDP Guard
  auth.py       Auto-login and API-key resolution
  config.py     Telemetry/config directory
  legacy.py     0.x removal shims
tests/ci/       25 CI test files, real REPL worker + headless Chrome in several tests
skills/         packaged browser-use/cloud/open-source/qa/remote-browser/x402 skills
examples/       usage examples
scripts/, docker/, static/, bin/, .github/
```

## 3. Entry points

| Surface | Entry | What it does |
|---|---|---|
| Python | `Agent(task, llm=None, browser=None).run()` | Builds model, browser, REPL, session; runs Pydantic AI loop; returns `AgentHistoryList` |
| Python sync | `Agent.run_sync()` | Runs `run()` in an event loop; not re-entrant inside a running loop |
| CLI | `browser-use` -> `browser_use.cli:main` | Interactive/one-shot CLI; pipes Python to harness |
| CLI MCP | `browser-use --cli-mcp` | stdio MCP server with `browser_exec`, `browser_screenshot`, `browser_login`, `browser_feedback` |
| Legacy MCP | `browser-use --mcp` | Deprecated; serves the `--cli-mcp` tool surface, old tool names return removal errors |
| Skill install | `browser-use skill install` | Copies bundled skill files into agent skill directories |

Evidence: `pyproject.toml:52-55`; `agent/service.py:221,452,665`; `mcp/cli_mcp.py:473-509`; `skills/install.py:10-109`.

## 4. Core modules and responsibilities

### Agent control plane

`agent/service.py` owns construction and lifecycle:
- Validates constructor arguments, including removed 0.x knobs.
- Resolves the model and provider pricing before a browser is started.
- Builds prompts and tools via `build_agent`.
- Starts the browser, creates `Repl`, creates `Session`, and enters `run()`.
- Records telemetry, captures the final page before stopping the browser, shuts down, then judges the result (the judge runs after the browser is stopped).
- Exposes `add_new_task()` for multi-task reuse on the same agent/browser.

`agent/loop.py` owns the Pydantic AI execution loop:
- `python(code)` tool: one sequential cell at a time.
- Delivery tools: `finish`, `finish_from_repl`, plus `on_text`.
- Delivery checks, repair turns, budget turn, split-delivery rejection.
- Provider retry policy, streaming, compaction, image pruning.
- `run()` maps timeout, step limit, provider errors to statuses.

`agent/views.py` defines `AgentHistoryList`: status, text, output, steps, usage, cost, partial, error, messages, success, judgement.

### Execution plane

`repl/session.py` is the parent:
- Starts a worker process in a new session/process group.
- Passes an allowlisted environment.
- Connects to a loopback TCP server with a random token.
- Serializes cells under one `asyncio.Lock`.
- Kills the worker tree on timeout and restarts it.
- Redacts returned values and errors.

`repl/worker.py` is the child:
- Imports `browser_harness.helpers`, then clears and restores the environment.
- Installs helpers and `cdp()` into a persistent namespace.
- Executes cells, captures stdout/stderr, screenshots, and stats.
- Implements `type_secret`, `checkpoint`, `screenshot`, `show_image`, TOTP, upload, and domain checks.
- Maintains cell files under the run state directory.

### Browser plane

`browser/browser.py` selects one of four modes:
- `cloud` (default): Browser Use Cloud browser.
- `chromium`: fresh local Chromium with a temp profile.
- `chrome`: attach to the user's already-open Chrome (shared daemon), or copy a named profile.
- `cdp`: connect to an explicit CDP URL.

It also owns daemon startup, cloud create/stop, keep-alive, domain Guard wiring, tab sweep, and downloads directory.

`browser/local.py` resolves Chromium executables, copies profiles, builds Chrome arguments, and requires a real DevTools port rather than attaching to any process on 9222/9223.

### Policy and authority

`domains.py` defines matching and enforcement:
- `DomainRules.of()`: allow/prohibit lists, block-IP option.
- `Guard`: a second CDP WebSocket using `Fetch.enable` at the Document request stage; main-frame only; iframe traffic passes; target changes to disallowed URLs reset to `about:blank`.
- `sweep()`: resets disallowed page tabs after a cell.

`auth.py` resolves keys:
- `BROWSER_USE_API_KEY` > stored `browser-harness` auth > gateway model fallback.
- Interactive desktop can auto-open browser login; SSH/no-display can print a device code; CI/notebook/pipe do not auto-login.
- One login per process; concurrent runs wait.

### Observation and evaluation

`judge.py` performs a post-hoc judgement after the browser is stopped:
- The final page screenshot is captured before `Browser.stop()`, then held for the judge.
- Prompt says trace/screenshots are evidence, not instructions.
- Trace/task are bounded and redacted.
- `judge_cached()` appends the judge as the run's last model turn to reuse prompt cache; falls back to `judge_run()`.
- Judge failure returns `None` and does not change run status.

### Telemetry

`telemetry/service.py` sends PostHog events from a background thread. `telemetry/stats.py` implements `scrub()` and `RunStats`. `_config()` in `service.py` sends counts, booleans, enums and names, not task text, secret values or workspace paths.

## 5. Core data structures

| Object | File:line | Fields that matter |
|---|---|---|
| `Agent` | `agent/service.py:221` | `task`, `llm`, `browser`, `tools`, `sensitive_data`, `workspace`, `mode`, `judge`, `use_vision`, `output_model_schema`, `_history`, `_agent`, `_repl` |
| `Session` | `agent/loop.py:127` | `timeout_s`, `max_steps`, `cell_timeout_s`, `vision`, `messages`, `checkpoint`, `covered`, `usage_after`, `finalizing`, `checked_delivery`, `split_delivery_step`, `success`, `finish_via`, `budget_hit`, `error_type` |
| `AgentHistoryList` | `agent/views.py:84` | `status`, `text`, `output`, `steps`, `duration`, `usage`, `cost_usd`, `partial`, `error`, `messages`, `success`, `judgement` |
| `Repl` | `repl/session.py:304` | `workspace`, `env`, `secrets`, `state_dir`, `cell`, `proc`, `lock`, `after_cell`, `reattach` |
| `Browser` | `browser/browser.py:530` | `kind`, `allowed_domains`, `prohibited_domains`, `block_ip_addresses`, `cloud_*`, `keep_alive`, `_daemon`, `_guard`, `_local`, `_tabs_file` |
| `DomainRules` | `domains.py:198` | `allowed`, `prohibited`, `block_ips` |
| `Guard` | `domains.py:281` | `_ws`, `_pages`, `_failed`, `_notices` |
| `Compaction` | `agent/loop.py:772` | `checkpoint`, `covered`, `archive path`, projected message list |
| `RunStats` | `telemetry/stats.py:558` | cells, helpers, CDP methods, hosts, errors, images, navigations, typed text, codes |

## 6. State model

### Durable state

- Files created by cells in the workspace.
- `checkpoint(filename, value)` outputs; `partial=True` also writes `state/partial.json`.
- Downloaded files under `workspace/downloads`.
- Browser tabs, cookies, and login state inside the browser process/cloud browser.
- Compaction archive JSON under `workspace/.browser-use/context/`.

### Volatile state

- Python variables and imports in the worker namespace.
- In-flight cell subprocesses and threads.
- Model message history in the `Session` object.
- Cloud browser existence while `keep_alive=True`.

### State transitions

- Cell timeout: worker process group is killed, partial cell file is read, worker restarts; Python variables are lost, browser and files remain.
- Browser connection lost: `Repl.call()` invokes `reattach`; if the daemon attaches a new browser, Repl closes and reports that tabs/logins and Python variables are gone.
- Run timeout: `asyncio.timeout` cancels the loop; `result.status='timeout'`; `result.text` falls back to `partial.json` if present.
- Run max steps: `status='max_steps'`; the last request is a delivery-only turn; delivery may still produce `completed`, but `success=False`.
- Provider/browser error after start: `status='error'`, not an exception to the caller.
- Compaction: messages before the cut are replaced by a summary checkpoint plus a redacted evidence archive; if the projected history is not smaller, compaction is rejected and the full history is kept.

Evidence: `agent/prompt.py:36`; `repl/session.py:443-568`; `repl/worker.py:742-751`; `agent/loop.py:157-172,976-1020,1338-1350`; `agent/views.py:84-151`.

## 7. Test system

- 25 files in `tests/ci`; 18,492 lines of tests.
- Pytest config: `timeout=300`, `asyncio_mode=auto`, `chrome` marker, `testpaths=["tests"]`.
- Tests use a real REPL worker in many cases and a real headless Chrome for Chrome-marked tests.
- Scripted `FunctionModel` replaces paid LLM calls; fake browsers replace cloud where needed.
- Test names reveal concrete contracts: worker env allowlist, secret redaction, domain guard failure/reconnect, delivery repair, last-request delivery, judge failure isolation, compaction archive, partial on timeout, cell timeout worker restart.

**This run did not execute tests.** Collection failed because `pydantic_ai` is not installed. There are no pass/fail numbers in this package.

## 8. Configuration surface

| Area | Main knobs |
|---|---|
| Agent | `llm`, `browser`, `tools`, `sensitive_data`, `workspace`, `mode`, `max_context_chars`, `reasoning`, `judge`, `use_vision`, `highlight_actions`, `web_search`, `extend_system_message` |
| Run | `max_steps`, `timeout` |
| Browser | `cloud_profile_id`, `cloud_proxy_country_code`, `cloud_timeout`, `cloud_browser_ids`, `keep_alive`, `allowed_domains`, `prohibited_domains`, `block_ip_addresses`, `headless`, `user_data_dir`, `executable_path`, `cdp_url` |
| REPL | `cell_timeout_s`, `workspace`, `secrets`, `after_cell` |
| Telemetry | `ANONYMIZED_TELEMETRY`, `BROWSER_USE_CONFIG_DIR`, `browser-use telemetry disable` |
| Auth | `BROWSER_USE_API_KEY`, `BROWSER_USE_AUTO_LOGIN`, stored `browser-harness` auth |

## 9. Permission and governance model

The repository uses **layered policy, not a sandbox**:

1. **Helper-level checks** before `Page.navigate` / `Target.createTarget` inside the worker.
2. **Browser-level `Guard`** for main-frame document requests through a second CDP connection.
3. **Post-cell sweep** that resets disallowed page tabs.

The documented exception matrix matters:
- `Browser()` cloud: all three layers.
- `Browser('chromium')`: all three layers.
- `Browser('chrome', profile=...)`: all three layers on the owned browser.
- `Browser('chrome')` attached to the user's Chrome: only layer 1 and layer 3, and layer 3 only touches agent-created tabs. No Guard, because a second CDP connection would prompt the user again.
- `Browser(cdp_url=...)`: supported layers, but the external CDP owner is outside this process.
- MCP `browser_exec`: no REPL worker, no `sensitive_data`, no `type_secret`, no worker env allowlist, no Guard unless the MCP server owns a Browser with domains.
- `Guard` disconnect: fail-open until reconnect.

`MIGRATING.md` explicitly says domain restrictions are “not a sandbox”, and that secrets are not removed from the machine: the worker can read files, environment, screenshots, live view, and recordings with the user's permissions.

## 10. External dependency boundary

- `browser-harness==0.1.13`: CDP daemon, helper functions, auth storage, CLI bridge. Not audited internally here.
- Pydantic AI 2.66: model loop, tools, streaming, compaction hooks.
- Provider SDKs: OpenAI, Anthropic, Google, plus OpenAI-compatible base URLs.
- `mcp==2.3.0`: stdio MCP server.
- `pillow`: image validation and downsize before model.
- `uuid7`: anonymous telemetry device id.
- Browser Use Cloud: browser provisioning, live view, proxy, CAPTCHA, gateway model.
- PostHog EU: telemetry endpoint.
- Chrome/Chromium: local execution target.

## 11. Project-layer conclusions (not generalized)

1. The 1.0 execution unit is a Python cell, not a browser action object.
2. The policy boundary is split across helper checks, a CDP Guard, and a post-cell sweep; each mode has a different set.
3. The run result is a status plus optional judge verdict; `completed` is not the same as `success`.
4. Context compaction is evidence-preserving: a summary is projected into the prompt, while a redacted archive holds the covered messages.
5. The MCP path is an independent execution and authority plane, not the Agent path with a different transport.

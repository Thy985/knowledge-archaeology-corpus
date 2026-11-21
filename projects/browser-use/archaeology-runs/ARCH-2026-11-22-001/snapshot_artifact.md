# Snapshot Artifact — ARCH-2026-11-22-001

## Target

- repository: `https://github.com/browser-use/browser-use.git`
- local clone: `archaeology-jobs/ARCH-2026-11-22-001/repo`
- branch: `main`
- HEAD: `e682e0dc02d77a307a5b2aedf17142c2cd69af5b`
- HEAD timestamp: `2026-11-20T14:48:08-08:00` (2026-11-21 06:48 +08:00)
- HEAD subject: `1.0.8rc1: cached judge, SSH sign-in, step-limit answers and 3 fixes (#6489)`
- tag at HEAD: `1.0.8rc1`
- version (`pyproject.toml`): `1.0.8rc1`
- license (`pyproject.toml` + `LICENSE`): MIT
- Python requirement: `>=3.11,<4.0`
- shallow clone depth: 50; `git rev-list --count HEAD` = 393
- prior baseline fetch: `d8110c5ff87ccba887aaa726cdb780f2f84bef8d` (0.13.10, 2026-09-18)
- diff prior → HEAD: `546 files changed, 37,339 insertions(+), 135,830 deletions(-)`

## Size and shape

- tracked files: 146
- all tracked Python: 35,428 lines across 75 files
- `browser_use/` engine: 16,768 lines
- `tests/`: 18,492 lines; `tests/ci/` has 25 `test_*.py` files
- top-level directories: `.github`, `bin`, `browser_use`, `docker`, `examples`, `scripts`, `skills`, `static`, `tests`

### Largest engine files (approx.)

| File | Lines | Role |
|---|---:|---|
| `browser_use/agent/loop.py` | 1,378 | Pydantic AI loop, delivery, compaction, provider streaming |
| `browser_use/browser/browser.py` | 1,328 | Browser lifecycle, cloud/local/CDP modes, daemon, guard wiring |
| `browser_use/cli.py` | 1,262 | CLI, harness pipe, login, MCP dispatch |
| `browser_use/repl/worker.py` | 1,245 | Worker process, helpers, secrets, cell execution |
| `browser_use/agent/service.py` | 1,048 | Agent constructor, build, run lifecycle, judge, telemetry |
| `browser_use/mcp/cli_mcp.py` | 1,041 | stdio MCP server with persistent Python namespace |
| `browser_use/ax.py` | 871 | ultrafast accessibility-tree helpers |
| `browser_use/telemetry/stats.py` | 661 | scrub and run statistics |
| `browser_use/telemetry/service.py` | 638 | PostHog sender and opt-out |
| `browser_use/llm/__init__.py` | 619 | model resolution, gateway detection, provider preflight |
| `browser_use/browser/local.py` | 617 | Local Chromium/profile launch |
| `browser_use/legacy.py` | 598 | 0.x import/name removal errors |
| `browser_use/repl/session.py` | 568 | Parent side of REPL, worker env, transport, timeouts |
| `browser_use/domains.py` | 475 | DomainRules and CDP Guard |
| `browser_use/auth.py` | 451 | Auto-login methods and key resolution |

## Entry points

- Python library: `from browser_use import Agent`; `Agent(task, llm, browser).run()` or `.run_sync()`.
- CLI: `browser-use` / `browseruse` -> `browser_use.cli:main`; `browser-use-tui` is a deprecated alias.
- MCP: `browser-use --cli-mcp` (`mcp/cli_mcp.py`) and the deprecated `browser-use --mcp` compatibility entry.
- Engine root: `browser_use/__init__.py` lazy-exposes `Agent`, `Browser`, `Tools`, `Controller`, `ActionResult`, `ChatBrowserUse`, `ChatOpenAI`, `ChatAnthropic`, `ChatGoogle`, `feedback`, `login`.

## Core architecture map

```
User task
  -> Agent(task, llm=None, browser=None)
       llm=None  -> ChatBrowserUse('bu-4.0') on the Browser Use gateway
       browser=None -> Browser(); default = Browser Use Cloud
  -> Agent.run(max_steps=100, timeout=1700)
       -> _build(): resolve_model, provider/base_url price, gateway detection, build_agent()
       -> preflight(): gateway/provider/search before any browser is created
       -> Browser.start(): cloud POST or local Chromium, then browser-harness daemon
       -> Repl(workspace, env, secrets, after_cell, search, downloads...)
            -> worker process: browser_use.repl.worker
       -> Session(repl, timeout_s, max_steps, vision)
       -> run(agent, task, Session)  [Pydantic AI]
            -> model calls a python(code) tool
            -> Repl.call('exec', code) -> worker executes code in persistent namespace
            -> browser-harness helpers + raw cdp()
            -> cell output/screenshot returned to model
       -> finish / finish_from_repl / text output
       -> judge (same model, optional)
       -> shutdown: Repl.close, Browser.stop
```

Evidence: `AGENTS.md` main chain; `browser_use/agent/service.py:221-358,380-582,699-718`; `browser_use/agent/loop.py:1301-1361`; `browser_use/repl/session.py:304-491`; `browser_use/repl/worker.py:1135-1229`; `browser_use/browser/browser.py:701-998`.

## Core data structures and states

| Structure | Location | What it holds |
|---|---|---|
| `Agent` | `agent/service.py:221` | task, llm, browser, tools, sensitive_data, workspace, mode, judge config, runtime caches |
| `Session` | `agent/loop.py:127` | timeout, max_steps, cell timeout, messages, checkpoint/covered/usage_after, delivery/finalizing flags, budget hit, success, error type |
| `AgentHistoryList` | `agent/views.py:84` | status, text, output, steps, duration, usage, cost, partial, error, messages, success, judgement |
| `Repl` | `repl/session.py:304` | worker process, lock, env, secrets, state dir, cell numbering, reattach/after_cell hooks |
| `Browser` | `browser/browser.py:530` | kind, domains, cloud/local/cdp state, daemon name, guard, local launcher, tabs, downloads |
| `DomainRules` / `Guard` | `domains.py:198,281` | allow/prohibit/block-IP rules; dedicated CDP connection and blocked-tab notices |
| `Compaction` | `agent/loop.py:772` | summary checkpoint, evidence archive path, covered message index, projected history |
| `RunStats` | `telemetry/stats.py:558` | counters for cells, helpers, CDP methods, domains, errors, images, navigations, typed text |

### State durability

- Browser mutations, downloaded files, workspace files, and `checkpoint()` outputs survive.
- Python variables and imports survive normal Python/CDP errors, but are lost when a cell times out or the worker exits.
- A timeout kills the worker process group and restarts the worker; browser state and files survive.
- `partial=True` writes `state/partial.json`; if the run ends without delivery, `result.text` falls back to that partial.

Evidence: `agent/prompt.py:36`; `repl/session.py:1-12,443-568`; `repl/worker.py:742-751`; `agent/loop.py:1338-1343`; `tests/ci/test_loop.py:71-115`.

## Tests and quality baseline

`pyproject.toml` pytest configuration:
- `timeout=300`
- `asyncio_mode=auto`
- marker `chrome: needs a local Chrome/Chromium (skipped when none is found)`
- `testpaths=["tests"]`
- `addopts="--strict-markers --tb=short"`

Security/behavioral test names read this run include:
- `test_worker_env_has_no_credentials`
- `test_a_browser_harness_call_from_a_cell_fails_closed`
- `test_trust_model_says_the_worker_can_still_reach_the_parents_keys`
- `test_secret_values_are_redacted_from_everything_returned`
- `test_checkpoints_and_partial_results_are_redacted`
- `test_domain_scoped_secret_is_refused_on_another_origin`
- `test_guard_start_failure_raises`
- `test_guard_reconnects_after_its_connection_drops`
- `test_only_your_open_chrome_keeps_its_tabs_on_disallowed_pages`
- `test_placeholder_or_missing_file_delivery_gets_one_repair`
- `test_a_delivery_sharing_its_response_with_a_later_call_is_not_taken`
- `test_a_delivery_needing_repair_on_the_last_request_is_delivered`
- `test_a_failed_judge_leaves_the_run_as_it_was_and_is_still_counted`
- `test_compacts_past_the_threshold_keeping_tool_pairs_and_user_messages`
- `test_archive_is_written_redacted_and_without_images`
- `test_summarizer_failure_keeps_the_history_and_is_tried_again`
- `test_timeout_returns_partial`
- `test_cell_timeout_restarts_worker_keeps_files`

**Execution status:** no test was run. `python3 -m pytest tests/ci/test_domains.py` failed at collection with `ModuleNotFoundError: No module named 'pydantic_ai'`. No dependency installation, no headless Chrome run, no API key, no cloud browser. All test statements below are static test-code observations, not pass/fail results.

## Configuration and runtime defaults

| Setting | Default / behavior |
|---|---|
| model | `bu-4.0` on Browser Use gateway when `llm=None` |
| browser | `Browser()` = Browser Use Cloud; `Browser('chromium')` = fresh local Chromium; `Browser('chrome')` = attach to user Chrome; `Browser(cdp_url=...)` = direct CDP |
| `run(max_steps, timeout)` | `max_steps=100`, `timeout=1700` seconds |
| cell timeout | `Session.cell_timeout_s=120` |
| delivery point | `timeout - max(0.1*timeout, min(30, 0.5*timeout))` → 90% budget, or up to 30 s reserve |
| image pruning | `PRUNE_IMAGES_AT=20`, `KEEP_IMAGE_RESULTS=2` |
| model request timeout | `MODEL_TIMEOUT_S=300`; streaming `STREAM_TIMEOUT_S=930`; gateway max output `16_384` |
| compaction | past 75% of char budget or 65% of context window; `max_chars` default 4 chars/token, 240k if window unknown |
| telemetry | `ANONYMIZED_TELEMETRY` defaults to true; config dir defaults to `~/.config/browseruse` |
| workspace | explicit path or temp dir `browser-use-*`, kept after run; caller deletes |
| cloud | one Browser Use key covers gateway model + cloud browser + CLI + MCP; `BROWSER_USE_API_KEY` wins over stored key |
| auth | interactive desktop -> browser login; SSH/Linux no display + tty -> device code; CI/notebook/pipe -> `MissingAPIKeyError`; auto-login once per process |
| secrets | values under 4 chars redacted only as whole words |

Evidence: `pyproject.toml`; `README.md:134,166,232`; `agent/service.py:244-358,452-461`; `agent/loop.py:86-121,157-163`; `config.py:41-58`; `auth.py:76-222`; `browser/browser.py:519-530,899-1034`.

## Dependency boundary

Runtime dependencies are pinned exactly:
`browser-harness==0.1.13`, `pydantic==2.14.3`, `pydantic-ai-slim[openai,anthropic,google]==2.66.0`, `google-genai==2.34.0`, `genai-prices==0.2.1`, `idna==3.20`, `pillow==12.3.0`, `python-dotenv==1.2.4`, `uuid7==0.1.0`, `mcp==2.3.0`.

`browser-harness` is the CDP daemon/helper layer; its source is **outside this repository** and was not deep-audited this run. This snapshot treats browser-harness as an external dependency boundary, not as verified internal code.

## Delta from prior archaeology

- Prior: 0.13.10 @ `d8110c5`, DOM-oriented action loop, 25 Facts / 22 EK / 10 KO.
- Current: 1.0.8rc1 @ `e682e0d`, Pydantic AI loop, persistent Python REPL, raw CDP, browser-harness helpers, cloud-first default.
- `MIGRATING.md` explicitly removes 0.x planner, DOM engine tuning, history replay, pause/resume/stop, per-step callbacks, `save_as_playwright_script`, 0.x MCP tool names, and Laminar self-tracing.
- Prior conclusions about DOM selection, per-step action schema, watchdog event bus, and three-layer security are **not automatically valid** for 1.0. They must be re-derived or downgraded to “0.x historical facts”.

## Snapshot limitations

- Static snapshot only; no install, no test run, no browser launch.
- `browser-harness` internals not read.
- Cloud behavior, live view, CAPTCHA solver, proxy, and SSH device-code flow were not exercised.
- Release candidate status: `1.0.8rc1` is an RC tag, not a stable GA release.
- `README.md` benchmark references were not converted into numeric claims; no vendor-reported score is treated as fact.

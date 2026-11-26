# Snapshot Artifact — ARCH-2026-11-27-001

- **run_id**: `ARCH-2026-11-27-001`
- **job_id**: `KA-2026-11-27-001`
- **project**: `browser-use`
- **mode**: `refresh` (baseline `ARCH-2026-09-22-001`, version `0.13.10`)
- **repository**: `https://github.com/browser-use/browser-use.git`
- **target ref**: tag `1.0.8`
- **commit**: `753d629d76eb2b670f4cf467f3355bbe750f6845`
- **commit date**: `2026-11-21 19:30:22 -0800`
- **commit subject**: `1.0.8: release the 1.0.8rc1 code (#6504)`
- **author**: Gregor Žunič
- **clone**: `git clone --depth 1 --branch 1.0.8`; `is-shallow-repository=true`; detached HEAD; `git describe --tags --always` = `1.0.8`
- **snapshot timestamp**: 2026-11-27 (Asia/Shanghai)
- **local path**: `archaeology-jobs/ARCH-2026-11-27-001/repo`
- **read-only status**: `git status --porcelain` empty after dependency sync and tests; no target-repo writes.

## 1. Repository size and composition

| Metric | Value | Method |
|---|---:|---|
| Tracked files | 146 | `git ls-files \| wc -l` |
| Python lines (tracked `.py`) | 35,428 | `git ls-files '*.py' \| xargs wc -l` |
| Test lines (`tests/**/*.py`) | 18,492 | `git ls-files 'tests/**/*.py' \| xargs wc -l` |
| Test files (`tests/**/*.py`) | 25 | `git ls-files 'tests/**/*.py' \| wc -l` |
| Test functions in `tests/ci` | 429 | `grep -R '^def test_' tests/ci \| wc -l` |
| Working tree size excluding `.venv` | 4.8 MB | `du -sh` on clean clone |

The repository is small enough for a single-day deep pass: 16,936 non-test Python lines, with a test suite larger than half the product code.

## 2. Package identity and declared runtime

`pyproject.toml`:

- name `browser-use`; version `1.0.8`; license MIT; Python `>=3.11,<4.0`.
- description: “Browser agent on Pydantic AI: one persistent Python REPL on raw CDP, cloud or local Chrome”.
- console scripts: `browser-use` / `browseruse` → `browser_use.cli:main`; `browser-use-tui` → `browser_use.cli:browser_use_tui_main`.
- pytest: `timeout=300`, `asyncio_mode=auto`, marker `chrome`, `testpaths=tests`.
- `[tool.uv] exclude-newer = "2026-11-09T09:48:30Z"`.
- runtime pins include `browser-harness==0.1.13`, `pydantic==2.14.3`, `pydantic-ai-slim[openai,anthropic,google]==2.66.0`, `google-genai==2.34.0`, `mcp==2.3.0`, `pillow==12.3.0`, `python-dotenv==1.2.4`, `uuid7==0.1.0`.
- dev pins include `pytest==9.0.2`, `ruff==0.14.14`, `pyright==1.1.408`.

Local verification: `uv sync --all-extras --frozen` completed in 40.9 s (uv 0.8.9, Python 3.12.11, Node v22.23.4).

## 3. Test execution

Command:

```
uv run pytest tests/ci -m 'not chrome' -q
```

Result:

```
964 passed, 3 skipped, 140 deselected, 1 warning in 328.03s
```

The single warning is a pydantic_ai unimplemented-event warning for `ResponseCompactionCompactingEvent`, not a failure. The `chrome` marker selects 140 tests that need a real Chrome; they were intentionally excluded from this snapshot run.

CI declarations:

- `.github/workflows/test.yaml` unit job: `ubuntu/macos/windows × py3.11`, plus ubuntu 3.12/3.13/3.14; `uv sync --locked --dev --all-extras`; `uv run --no-sync pytest tests -m "not chrome"`; 15-minute timeout.
- `.github/workflows/test.yaml` chrome job: ubuntu with real headless Chrome, scripted `FunctionModel`, no provider and no cloud; fails if `google-chrome` is absent; runs `pytest tests -m chrome -rs`.
- `.github/workflows/package.yaml`: `uv build`; installs the wheel on ubuntu/macos/windows × py3.11/3.13 from `$RUNNER_TEMP`; verifies CLI `--help`.
- `.github/workflows/docker.yml`: per-architecture build, `browser-use --version`, scripted `docker/smoke.py`; pushes only on release.
- `.github/workflows/lint.yml` and `install-script.yml` exist; `install-script.yml` runs `scripts/sync_browser_harness_skill.py --check`.

## 4. Project map (declared and code-traced)

Declared architecture in `AGENTS.md` (identical content in `CLAUDE.md`):

```
Agent(task, llm, browser).run()  -> browser_use/agent/service.py
  -> resolve_model(llm)          -> browser_use/llm/__init__.py
  -> Browser.start()             -> browser_use/browser/browser.py
  -> run(agent, task, Session)   -> browser_use/agent/loop.py
       python(code) -> Repl.call -> browser_use/repl/session.py
                    -> worker    -> browser_use/repl/worker.py
       mode='ultrafast' adds bu  -> browser_use/ax.py
  <- AgentHistoryList             -> browser_use/agent/views.py
```

Code confirms the declared path:

1. `Agent.run()` gates concurrency and browser ownership, then `_run()` builds the Pydantic AI agent, starts `Browser`, creates or reuses `Repl`, constructs `Session`, and calls `run()` (`browser_use/agent/service.py:452-565`).
2. `build_agent()` constructs a Pydantic AI `Agent` with `python` as a sequential tool and optional custom `Tools` (`browser_use/agent/loop.py:1120-1240`).
3. `python()` computes the remaining budget, calls `Repl.call('exec', code, timeout)`, redacts/truncates output, converts screenshots to `BinaryContent` (`browser_use/agent/loop.py:178-220`).
4. `Repl` spawns `python -P -m browser_use.repl.worker` in its own process group, with an allowlisted env and a loopback token handshake (`browser_use/repl/session.py:357-435`).
5. `worker.py` creates a persistent namespace containing `browser-harness` helpers plus `screenshot`, `checkpoint`, `type_secret`, `solve_captcha`, `report_issue`, and optionally `bu` (`browser_use/repl/worker.py:1-20`, `494-910`).
6. `Browser.start()` selects cloud, launched Chromium, attached Chrome, or `cdp_url`; creates a browser-harness daemon; returns env `BU_NAME`, `BU_OWN_BROWSER`, optional `BU_REMOTE_BROWSER` and `BU_CAPTCHA_SOLVER` (`browser_use/browser/browser.py:899-999`).
7. `AgentHistoryList` carries `status`, `text`, `output`, `steps`, `usage`, `cost_usd`, `partial`, `error`, `messages`, optional `success`, `output_schema`, `downloads`, `judgement` (`browser_use/agent/views.py:81-110`).

## 5. Core modules

| Area | Files | Role |
|---|---|---|
| Public facade / 0.x compatibility | `agent/service.py`, `agent/views.py`, `legacy.py` | `Agent`, `Browser`, `Tools`, `AgentHistoryList`; removed-name errors |
| Agent loop | `agent/loop.py` | Pydantic AI agent, python tool, delivery, budgets, compaction, streaming |
| Prompting | `agent/prompt.py` | Python/CDP system prompt, secrets/downloads/search/captcha prompt fragments |
| Judge | `agent/judge.py` | cached and standalone post-run judgement |
| REPL control plane | `repl/session.py` | worker lifecycle, token handshake, per-cell timeout, redaction, reattach |
| REPL execution plane | `repl/worker.py` | persistent namespace, browser helpers, secrets, checkpoint, process-group death |
| Browser lifecycle | `browser/browser.py`, `browser/local.py` | cloud/launch/attach, daemon, downloads, watchdog, local flags |
| Domain policy | `domains.py` | pattern matching, `Guard` Fetch interception, tab sweep |
| Auth | `auth.py` | env key, stored key, browser/device login, key hygiene |
| Custom tools | `tools.py` | function tools next to Python |
| MCP | `mcp/cli_mcp.py` | stdio server exposing `browser_exec`, `browser_screenshot`, `browser_login`, `browser_feedback` |
| CLI | `cli.py`, `skills/` | command surface, doctor, login, skill install, own Chromium fallback |
| Telemetry | `telemetry/` | PostHog events, device id, outbox, auth/run events |

## 6. Key data structures and state

- `Session` is the run’s control state: `timeout_s=1700`, `max_steps=1000`, `cell_timeout_s=120`, `vision=True`, `finalizing`, `checked_delivery`, `split_delivery_step`, `success: bool|None`, `finish_via`, `budget_hit`, `checkpoint`, `covered`, `usage_after`, `overflowed_step`, `provider_retried`, `unforced`, `request_params`, `summaries`, `rec` (`browser_use/agent/loop.py:126-155`).
- `AgentHistoryList` is the public result object; `success` is only the agent’s optional `finish(success=...)`; `judgement` is a separate `Judgement` (`browser_use/agent/views.py:81-110`).
- `Judgement` is a Pydantic model with `success`, `failure_reason`, `description`, `prompt_injection_detected`, `bot_block_or_captcha_detected`, `gotchas` (max 5) (`browser_use/agent/judge.py:40-70`).
- `Browser` holds `kind` in `cloud|chromium|chrome|cdp`, `keep_alive`, `live_view`, `captcha_solver`, `domains`, `_guard`, `_local`, `_watchdog`, download bookkeeping, and daemon identity (`browser_use/browser/browser.py:556-695`).
- `Repl` holds workspace, env, mode, process, Windows job object, loop, state directory, cell counter, lock, and secrets (`browser_use/repl/session.py:304-347`).
- `DomainRules` is frozen `allowed/prohibited/block_ips`; `Guard` holds page URLs, blocks, notices, replies, connection state (`browser_use/domains.py:139-175`, `300-340`).

## 7. Lifecycle

**Browser lifecycle**: `Browser()` is cloud by default; `Browser('chromium')` launches a fresh local Chrome with a temp profile; `Browser('chrome')` attaches to the user’s open Chrome through the shared daemon; `Browser('chrome', profile='Work')` launches a new Chrome on a temporary copy of that profile (`browser_use/browser/browser.py:1-9`). `Agent.run()` stops the browser unless `keep_alive=True`; cloud browsers are also stopped on process exit, SIGTERM, SIGHUP, or server timeout (`browser_use/browser/browser.py:8-10`).

**REPL lifecycle**: start → token-authenticated loopback connection → init secrets/search/downloads → cell loop → on timeout/worker exit, kill process group and restart; Python variables lost, files/checkpoints/browser state survive (`browser_use/repl/session.py:31`, `357-435`, `524-560`).

**Run lifecycle**: build/preflight → start browser → create/reuse REPL → run Pydantic AI loop → judge (after browser stop) → return result; `finally` stops the REPL/browser and records telemetry (`browser_use/agent/service.py:475-627`).

## 8. Configuration and dependencies

- `.env` is loaded into `os.environ` by `Agent`/`Browser` import (`browser_use/config.py:9-15`); the REPL worker never calls `load_env` and starts from an allowlisted parent env (`browser_use/repl/session.py:78-88`).
- `ANONYMIZED_TELEMETRY` defaults true; `BROWSER_USE_CONFIG_DIR` defaults to `~/.config/browseruse`; `is_in_container()` covers Docker, podman, Kubernetes, and `container` env (`browser_use/config.py:46-70`).
- Cloud API base `https://api.browser-use.com/api/v4`; cloud create timeout 120 s; local Chrome start wait 120 s; downloads 300 s (`browser_use/browser/browser.py:200-203`).
- Default model from `Agent` docstring is `ChatBrowserUse('bu-4.0')` (`browser_use/agent/service.py:221-243`); `build_agent`’s own default is `openai-responses:gpt-6-luna` (`browser_use/agent/loop.py:1110-1120`).
- Browser runtime dirs must be 0700 and not other-writable/owned; Unix socket path length is checked; multiple live debug dirs raise rather than attach an arbitrary Chrome (`browser_use/browser/browser.py:434-487`).

## 9. Authority and governance mechanisms (initial map)

1. **Run authority**: `Agent.run()` rejects overlapping runs on one Agent and one Browser (`browser_use/agent/service.py:452-470`).
2. **Browser authority**: kind selection is mutually exclusive; 0.x options raise with alternatives (`browser_use/browser/browser.py:519-654`).
3. **Execution authority**: model-written Python runs as the user in a child process; env allowlist blocks secret-looking names but is not a sandbox (`browser_use/repl/session.py:1-18`, `78-88`).
4. **Secret authority**: `type_secret` scopes values to the focused frame’s URL; TOTP seeds are converted to current codes; values are redacted in selected channels, not everywhere (`browser_use/repl/worker.py:840-897`, `browser_use/repl/session.py:135-200`).
5. **Navigation authority**: `DomainRules` + helper refusal + `Guard` + sweep; subresources/iframes/Python networking are out of scope (`browser_use/domains.py:1-24`, `300-340`).
6. **Delivery authority**: `finish`, `finish_from_repl`, and `on_text` are the only delivery outputs; `check_split_delivery` and `check_delivery` gate them (`browser_use/agent/loop.py:226-300`, `336-405`).
7. **Verdict authority**: agent’s `success` is separate from `judgement`; `is_successful()` returns the agent’s verdict, not the judge’s (`browser_use/agent/views.py:105-150`).
8. **Credential authority**: `BROWSER_USE_API_KEY` wins over stored key; automatic login only on an interactive main-thread run; stored key in browser-harness auth store (`browser_use/auth.py:1-3`, `202-221`).

## 10. What changed since the 0.13.10 baseline

The 1.0 line removes the DOM serializer, action schema, and event-bus watchdog layer and replaces them with a persistent Python REPL over raw CDP. The repository’s own `MIGRATING.md` states this directly, and the code corroborates it:

- no `browser_use/dom/serializer` or action registry remains in the tracked tree; `Tools` only registers function tools next to `python` (`browser_use/tools.py:1-30`).
- the model prompt instructs raw `js`, `cdp`, `click_at_xy`, `type_text`, `press_key`, `screenshot`, and `upload_file` (`browser_use/agent/prompt.py:7-60`).
- 0.x `*_watchdog.py` modules are absent; domain enforcement is now `Guard` + sweep (`browser_use/domains.py`).
- 0.x options such as `max_actions_per_step`, `enable_planning`, `loop_detection_*`, `message_compaction`, and `file_system_path` are removed or redirected (`browser_use/agent/service.py:81-180`).

The refresh must therefore treat the old 0.13.10 KO set as a **historical baseline**, not as inherited truth. Cross-architecture invariants require fresh 1.0.8 evidence.

## 11. Snapshot limitations

- Shallow clone: no full history, so failure/repair archaeology cannot rely on commit history; it uses code, docs, tests, and comments only.
- Chrome-marked tests (140) were not run; interactive Chrome, cloud browser, and real provider paths are not exercised locally.
- No cloud API key was used; cloud control plane is verified from code/tests/docs, not live API behavior.
- `MIGRATING.md` and `README.md` are design-intent evidence, not implementation facts; every claim below is triangulated with code or tests where possible.

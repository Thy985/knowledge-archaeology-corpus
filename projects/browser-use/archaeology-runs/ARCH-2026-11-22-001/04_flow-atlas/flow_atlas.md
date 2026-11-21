# 04 · Flow Atlas（七类流）— browser-use 1.0.8rc1

> Every edge below is derived from source or docs at `e682e0d`. Edge IDs are stable for this run. Conditions and exceptions are part of the edge, not footnotes.
>
> “Read, not executed”: no edge in this atlas is a runtime trace. Facts about ordering and conditions are static-code facts; runtime outcomes are marked as test contracts only where a test body was read.

## 1. Control Flow（控制流）

```
Agent.run(max_steps=100, timeout=1700)
  ├─ validate positive ints, one run per Agent, one Agent per Browser        [F-C-01]
  └─ Agent._run()
       ├─ _check_task()                                                       [F-C-02]
       ├─ raise_fd_limit()
       ├─ _build()
       │    ├─ resolve_model(llm)                                            [F-C-03]
       │    │    └─ missing key/extra -> raise (before browser)
       │    ├─ provider prices + gateway detection
       │    └─ build_agent(prompts, tools, output schema)
       ├─ RunRecorder + telemetry start
       ├─ preflight(provider/model/search)                                   [F-C-04]
       ├─ workspace + downloads dirs
       ├─ browser.start()                                                     [F-C-05]
       │    └─ config error -> raise; other start error -> status='error'
       ├─ Repl(...)                                                           [F-C-06]
       ├─ Session(...)
       └─ loop.run(agent, task, session)                                      [F-C-07]
            ├─ model request -> tool calls
            ├─ python(code) -> Repl.call('exec')                              [F-C-08]
            │    ├─ timeout -> close worker + partial output + CellTimeout
            │    └─ browser lost -> reattach -> maybe close Repl
            ├─ delivery call -> check_split_delivery -> check_delivery       [F-C-09]
            │    ├─ invalid before last request -> ModelRetry / repair turn
            │    ├─ invalid at last request -> warn + deliver
            │    └─ no delivery -> DELIVERY_REPAIR (once)
            ├─ budget reached -> DELIVER_NOW (delivery-only turn)
            └─ end -> status/success/partial
       ├─ _history = result
       ├─ final page screenshot (before browser stop)
       ├─ finally: _shutdown() -> Repl.close + Browser.stop                  [F-C-10]
       └─ finally after shutdown: _judge()                                  [F-C-11]
            ├─ judge_cached() -> fallback judge_run()
            └─ failure -> judgement=None (status unchanged)
```

### Control edges

| Edge | From → To | Condition / Exception | Source |
|---|---|---|---|
| F-C-01 | `Agent.run` → `_run` | rejects re-entry and a Browser owned by another active Agent | `agent/service.py:452-474` |
| F-C-02 | `_run` → `_check_task` | task must be non-empty `str` | `agent/service.py:475-520` |
| F-C-03 | `_build` → `resolve_model` | runs before `browser.start`; missing key/extra raises | `agent/service.py:380-450`; `llm/__init__.py:532-619` |
| F-C-04 | `_run` → preflight | official provider probe only; proxy/403/unknown do not block | `llm/__init__.py:461-515` |
| F-C-05 | `_run` → `Browser.start` | config error raises; 402/5xx/timeout becomes `status='error'`; `finally` stops | `agent/service.py:520-627` |
| F-C-06 | `_run` → `Repl` | workspace/env/secrets/mode/after_cell wired | `agent/service.py:537-550` |
| F-C-07 | `_run` → `loop.run` | wrapped in `asyncio.timeout(session.timeout_s)` | `agent/loop.py:1301-1361` |
| F-C-08 | model tool `python` → worker cell | sequential; `NO_TIME` if no budget; timeout kills worker | `agent/loop.py:175-225`; `repl/session.py:460-568` |
| F-C-09 | delivery tool → checks | split check once per request step (comment says once/run); delivery check once/run; last request warns instead of rejecting | `agent/loop.py:226-335` |
| F-C-10 | `_run` finally → shutdown | `Repl.close` then `Browser.stop(downloads_dir=...)`; final page captured before this | `agent/service.py:699-718` |
| F-C-11 | `_run` finally → `_judge` | after shutdown; only if judge + result.steps + agent; judge failure is non-fatal | `agent/service.py:628-657` |

### Terminal branches

| Branch | Trigger | Result | Source |
|---|---|---|---|
| Budget timeout | `asyncio.timeout` from `run` budget | `status='timeout'`; delivery attempt may still finish | `agent/loop.py:1301-1361` |
| Tool timeout | tool-level `TimeoutError` | `status='error'` | `agent/loop.py:1301-1361` |
| Max steps | `UsageLimitExceeded` | `status='max_steps'`; last request delivery-only | `agent/loop.py:1301-1361` |
| Provider/browser error | other exception after start | `status='error'`; does not raise to caller | `agent/loop.py:1301-1361`; `agent/service.py:520-627` |
| Reattach to a new browser | daemon changed browser | Repl closes, tabs/logins/variables lost | `repl/session.py:460-523` |
| Guard disconnect | CDP WS drop | fail-open until reconnect | `domains.py:308-390` |

**Key control observation**: there are four independent terminal policies (budget, tool timeout, max steps, provider error), and two independent delivery policies (budget delivery + repair). They must not be collapsed into one “timeout” concept.

## 2. State Flow（状态流）

```
Agent
  ├─ _running: bool
  ├─ _history: AgentHistoryList
  ├─ _agent: pydantic_ai.Agent | None
  ├─ _repl: Repl | None
  └─ _browser: Browser

Session (per run)
  ├─ messages: list[ModelMessage]
  ├─ checkpoint / covered / usage_after
  ├─ finalizing / checked_delivery / split_delivery_step
  ├─ success / finish_via / budget_hit / error_type
  └─ stats: RunStats

Repl (per run)
  ├─ proc: worker process (new session/process group)
  ├─ server/token: loopback auth
  ├─ cell: counter
  ├─ lock: one cell at a time
  └─ browser_gone: bool

Worker
  ├─ namespace: persistent Python vars/imports
  ├─ state_dir/cells/<n>.txt
  ├─ state_dir/partial.json
  ├─ state_dir/screenshots/
  └─ CellStats

Browser
  ├─ kind: cloud|chromium|chrome|cdp
  ├─ daemon / _guard / _local
  ├─ tabs file / _user_tabs
  └─ keep_alive

AgentHistoryList
  ├─ status / text / output / steps / usage / cost
  ├─ partial / error / messages
  ├─ success / judgement
```

### State transitions

| Transition | From → To | Condition | Source |
|---|---|---|---|
| F-S-01 | constructed → built | model resolved, prompts/tools built | `agent/service.py:380-450` |
| F-S-02 | built → browser started | preflight passed | `agent/service.py:501-536` |
| F-S-03 | browser started → REPL live | worker connected and token verified | `repl/session.py:357-442` |
| F-S-04 | cell running → cell done | output pumped, file read, images cleared | `repl/worker.py` cell loop |
| F-S-05 | cell timeout → new worker | old process group killed, partial file read | `repl/session.py:524-568` |
| F-S-06 | browser lost + reattach to same browser | Repl continues | `repl/session.py:460-523` |
| F-S-07 | browser lost + reattach to new browser | Repl closes; browser_gone=True | `repl/session.py:460-523` |
| F-S-08 | history full → compacted | 75% char or 65% window; projection shorter | `agent/loop.py:772-1020` |
| F-S-09 | summary empty / projection not shorter | compaction raises, full history retained | `agent/loop.py:1008-1019` |
| F-S-10 | run end → history saved | status/success/text/usage set | `agent/loop.py:1338-1361` |
| F-S-11 | history saved → shutdown | finally always; final page captured before stop | `agent/service.py:699-718` |
| F-S-12 | shutdown → judged | judge runs after browser stop; judge true + steps + agent | `agent/service.py:628-657` |
| F-S-13 | keep_alive close | Repl closed, Browser kept | `agent/service.py:709-718` |

**Durability tiers**:
- Durable: workspace files, checkpoints, partial, downloads, browser cookies/logins, compaction archive.
- Ephemeral: Python namespace, in-flight thread/subprocess, model message list, cloud browser while keep_alive.
- Recoverable: partial JSON is the only automatic fallback for a non-completed run.

## 3. Data Flow（数据流）

```
task(str)
  → prompt.py system prompt + local/cloud + captcha + search + domains
  → model request
  → model emits python(code) / finish / finish_from_repl / on_text
  → Repl.call('exec', code)
      → worker namespace
      → browser_harness helpers + cdp()
      → CDP / browser
      → cell stdout/stderr + optional screenshot
  → model context
  → delivery result (str or output_model_schema JSON)
  → AgentHistoryList
```

### Data edges

| Edge | Payload | Path | Condition | Source |
|---|---|---|---|---|
| F-D-01 | task text | Agent.task → system prompt | task must be non-empty | `agent/service.py:221-358`; `agent/prompt.py` |
| F-D-02 | model code | tool call → `Repl.call` | one cell at a time | `agent/loop.py:175-225` |
| F-D-03 | helper args / CDP | worker → harness → browser | `BU_REQUIRE_EXISTING_DAEMON=1` for naive CLI | `repl/session.py:357-442`; `repl/worker.py` |
| F-D-04 | cell output | worker → parent → model | clipped to `MAX_OUTPUT_CHARS`, full file path reported | `repl/worker.py` cell loop |
| F-D-05 | screenshot | worker → parent → model | JPEG q70; suppressed when `use_vision=False` | `repl/worker.py:728-741`; `agent/loop.py:175-225` |
| F-D-06 | secret | parent → worker control | named lookup; not env; scoped domain | `repl/session.py`; `repl/worker.py:807-900` |
| F-D-07 | checkpoint value | worker → state file → partial | JSON + redact + atomic write | `repl/worker.py:742-751` |
| F-D-08 | output/result | loop → `AgentHistoryList` | schema validation/coercion; str or JSON | `agent/loop.py:336-424` |
| F-D-09 | telemetry | service → PostHog | schema-minimized + scrub | `agent/service.py:742-1000`; `telemetry/*` |
| F-D-10 | archive | loop → disk | 0600, redacted, images omitted | `agent/loop.py:1010-1016` |

**Data boundary**: a secret can reach the browser as keystrokes and can appear in a screenshot; redaction covers text returns, not pixels. `open()` writes are outside the redacting pump. These are explicit channel gaps, not implementation accidents.

## 4. Evidence Flow（证据流）

```
page state
  → helper/CDP reads
  → cell stdout + screenshot
  → model context
  → checkpoint(partial=True)
  → result.text / result.messages
  → judge trace + screenshots
  → judgement (advisory)
```

### Evidence edges

| Edge | Evidence produced | Consumer | Condition | Source |
|---|---|---|---|---|
| F-E-01 | cell stdout/stderr | model + cell file | clipped; redacted pump | `repl/worker.py` |
| F-E-02 | screenshot JPEG | model + `state/screenshots` | image budget 20/2; no pixel redaction | `repl/worker.py:728-741`; `agent/loop.py:1055-1120` |
| F-E-03 | checkpoint value | disk + partial | redacted, atomic, opt-in | `repl/worker.py:742-751` |
| F-E-04 | compaction summary | next prompt | generated, fallible; user messages pinned | `agent/loop.py:976-1020` |
| F-E-05 | covered messages archive | disk | 0600, redacted, images omitted | `agent/loop.py:1010-1016` |
| F-E-06 | trace + screenshots | judge | bounds 120k/4; redacted | `agent/judge.py:100-142` |
| F-E-07 | judgement | `AgentHistoryList` | advisory; failure -> None | `agent/judge.py:143-218`; `agent/service.py:628-657` |
| F-E-08 | result.messages | caller | excluded from telemetry | `agent/service.py:876-1000` |
| F-E-09 | final page screenshot | judge | captured before browser stop; 10 s timeout; never fails run | `agent/service.py:628-657`; `agent/judge.py:63` |

**Evidence gaps**:
1. Screenshots are not redacted; a visible secret is evidence and leak at once.
2. Cell output is redacted on the return path, but arbitrary files written by the worker are not.
3. The judge sees model trace and screenshots, not an external action ledger; it cannot prove a side effect happened if the trace is incomplete.
4. Compaction archive preserves covered messages, but retention depends on the workspace.

## 5. Authority Flow（权威流）

```
user task / explicit config
  → Agent constructor validation
  → model/key/provider resolution
  → Browser mode selection
      ├─ cloud: Guard + helper checks + sweep
      ├─ chromium: Guard + helper checks + sweep
      ├─ chrome+profile: Guard + helper checks + sweep
      ├─ attached chrome: helper checks + agent-tab sweep only
      └─ cdp: external browser; local Guard only if wired
  → DomainRules
      → worker helper check (navigate/createTarget)
      → Guard Fetch (main-frame Document)
      → after-cell sweep
  → secret scope
      → type_secret domain check
      → recipient frame-tree confirmation
      → fail closed
  → MCP path (separate)
      ├─ in-process exec
      ├─ no sensitive_data/type_secret
      ├─ no worker env allowlist
      └─ cloud admin calls refused when domains configured
```

### Authority edges

| Edge | Authority granted / denied | Precedence | Source |
|---|---|---|---|
| F-A-01 | constructor accepts/rejects parameters | earliest | `agent/service.py:244-358` |
| F-A-02 | model/key/provider | before browser | `agent/service.py:380-450`; `llm/__init__.py` |
| F-A-03 | Browser mode decides which layers exist | before run | `browser/browser.py:519-760` |
| F-A-04 | DomainRules pattern match | before Guard | `domains.py:198-243` |
| F-A-05 | Guard main-frame Document | after helper, before page | `domains.py:281-441` |
| F-A-06 | secret domain scope + recipient | at typing time | `repl/worker.py:807-900` |
| F-A-07 | MCP execution authority | separate plane | `mcp/cli_mcp.py:473-1027` |
| F-A-08 | telemetry send authority | after run | `agent/service.py:742-1000`; `config.py` |
| F-A-09 | model prompt policy | advisory only | `agent/prompt.py`; `agent/judge.py` |

**Authority precedence rule observed**: explicit constructor/browser config outranks environment variables and prompts; `BROWSER_USE_ALLOWED_DOMAINS` in the environment is rejected with `RuntimeError` and must be passed as a parameter. Prompt statements are not authority; only code checks are.

## 6. Memory Flow（记忆流）

```
Session.messages
  ├─ system prompt + task
  ├─ assistant tool calls
  ├─ tool results (stdout/screenshots)
  └─ delivery / retries
        │
        ├─ threshold -> compaction projection
        │      ├─ summary checkpoint
        │      ├─ last two assistant/tool groups
        │      ├─ user messages verbatim
        │      └─ covered archive on disk
        │
        └─ run end -> result.messages

Workspace
  ├─ files created by cells
  ├─ downloads/
  ├─ .browser-use/context/*.json
  └─ state/partial.json

Worker
  └─ Python variables/imports (lost on restart)
```

### Memory edges

| Edge | Memory | Lifetime | Condition | Source |
|---|---|---|---|---|
| F-M-01 | message history | run | grows until compaction | `agent/loop.py:127-156` |
| F-M-02 | summary checkpoint | run/prompt | past 75%/65%; projection shorter | `agent/loop.py:772-1020` |
| F-M-03 | covered archive | workspace | written 0600 | `agent/loop.py:1010-1016` |
| F-M-04 | checkpoint/partial | workspace | explicit call | `repl/worker.py:742-751` |
| F-M-05 | REPL namespace | worker process | lost on restart | `repl/session.py:443-568` |
| F-M-06 | browser tabs/cookies | browser | lost if reattach to new browser or browser stop | `repl/session.py:460-523`; `browser/browser.py:701-760` |
| F-M-07 | follow-up task | same Agent | `add_new_task` appends, `_continue=True` | `agent/service.py:367-379` |
| F-M-08 | compaction archive images | disk | replaced by placeholder | `agent/loop.py:1010-1016` |

**Memory distinction**: the model's “memory” is the projected message list; the system's “memory” is the workspace + archive + browser state. They can diverge after compaction. A summary is not the original evidence.

## 7. Policy Flow（策略流）

```
system prompt (prompt.py)
  ├─ browser/cloud/local mode text
  ├─ captcha / no-captcha text
  ├─ search allowed / prohibited
  ├─ domain restrictions
  ├─ secrets rules
  ├─ downloads
  └─ checkpoint / delivery guidance
        │
        ├─ advisory to model
        │
code policy
  ├─ DomainRules + Guard + sweep
  ├─ type_secret + recipient
  ├─ worker env allowlist
  ├─ redaction channels
  ├─ telemetry scrub
  └─ MCP denials
```

### Policy edges

| Edge | Policy | Enforcement | Source |
|---|---|---|---|
| F-P-01 | mode-specific prompt (cloud/local) | advisory | `agent/prompt.py:61-75`; `agent/service.py:380-450` |
| F-P-02 | search vs domains conflict | constructor raises if search + allowed domains | `agent/service.py:380-450` |
| F-P-03 | domain restrictions | code layers; not a sandbox | `domains.py`; `MIGRATING.md:180-249` |
| F-P-04 | secrets | control channel + redaction; not isolation | `repl/session.py`; `repl/worker.py`; `MIGRATING.md:335-404` |
| F-P-05 | delivery format | code checks; best-effort at last request | `agent/loop.py:226-335` |
| F-P-06 | judge evidence handling | prompt + bounds + redaction | `agent/judge.py` |
| F-P-07 | telemetry minimization | schema + scrub + disable switch | `agent/service.py:742-1000`; `telemetry/stats.py` |
| F-P-08 | MCP cloud admin denial | explicit refusal when domains exist | `mcp/cli_mcp.py:463-472` |

**Policy gap**: prompt policy and code policy are not the same. A model can ignore the prompt; it cannot ignore a Guard block (while connected). The most important audit question is “which statements are enforced by code, and at which edge?”

## Flow-to-KO traceability

| Flow | Edges | KO |
|---|---|---|
| Control | F-C-01 至 F-C-11 | KO-01, KO-04, KO-05 |
| State | F-S-01 至 F-S-13 | KO-07, KO-10 |
| Data | F-D-01 至 F-D-10 | KO-01, KO-03 |
| Evidence | F-E-01 至 F-E-09 | KO-05, KO-06, KO-07 |
| Authority | F-A-01 至 F-A-09 | KO-02, KO-03, KO-08 |
| Memory | F-M-01 至 F-M-08 | KO-07, KO-11 |
| Policy | F-P-01 至 F-P-08 | KO-02, KO-03, KO-08, KO-09 |

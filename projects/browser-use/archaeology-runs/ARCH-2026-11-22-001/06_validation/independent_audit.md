# Independent Audit — ARCH-2026-11-22-001

> Method: a separate read-only reconstruction pass over the repository at `e682e0d`, without treating the Package as a source of facts. The auditor re-read `agent/service.py`, `agent/loop.py`, `agent/views.py`, `repl/session.py`, `repl/worker.py`, `browser/browser.py`, `domains.py`, `mcp/cli_mcp.py`, `llm/__init__.py`, `auth.py`, `pyproject.toml`, `LICENSE`, and the named test files. The auditor did not edit the Package; Reconciliation applied the accepted corrections afterward.
>
> What the auditor did not do: install dependencies, run pytest, launch Chrome, use an LLM/cloud key, or audit `browser-harness 0.1.13` internals.

## Verdict summary

| Verdict | Count |
|---|---|
| CONFIRMED | 18 |
| PARTIALLY_CONFIRMED | 4 |
| CONTRADICTED | 2 |
| DOWNGRADED | 0 |
| OVER_GENERALIZED | 0 |
| MISSING | 3 |
| NEEDS_HUMAN_REVIEW | 1 |
| **Total checks** | **28** |

The two contradictions are ordering/semantics errors, not factual fabrication. Both were corrected in Reconciliation. No KO was promoted during audit.

## Check matrix

| ID | Check | Verdict | Evidence / reason |
|---|---|---|---|
| A-01 | Agent lifecycle order: loop → history/final page → shutdown → judge | **CONTRADICTED** (corrected) | Package initially said judge before shutdown. `_run` returns after history/final page; `finally` calls `_shutdown`, and `_judge` runs after shutdown inside the shutdown `finally`. `agent/service.py:475-718` |
| A-02 | Model/key/provider preflight before browser.start | CONFIRMED | `_build` calls `resolve_model`; `_run` preflight before `browser.start`; `provider_preflight` official endpoints only. `service.py:380-540`; `llm/__init__.py:461-515` |
| A-03 | One active run per Agent, one active Agent per Browser | CONFIRMED | explicit `RuntimeError` checks. `service.py:452-474` |
| A-04 | Worker is a new process group; timeout kills and restarts it | CONFIRMED | `Popen(..., start_new_session=True)` on POSIX; `_call` close + partial read. `repl/session.py:357-568` |
| A-05 | Worker env allowlist and credential-name stripping | CONFIRMED | `ENV_ALLOW`, prefixes, `ENV_DENY`, `SECRET_ENV`. `repl/session.py:55-115` |
| A-06 | Secret redaction is channel-level, not isolation | CONFIRMED | module docstring, `secret_forms`, `redact_forms`, `open()` not intercepted; docs say not a sandbox. `repl/session.py:1-180`; `repl/worker.py` |
| A-07 | Guard is main-frame Document, iframes pass, reset to about:blank | CONFIRMED | `Fetch.enable` Document; `frame in self.pages`; `_reset`. `domains.py:281-441` |
| A-08 | Guard start failure raises; later disconnect is fail-open | CONFIRMED | `start()` raises; `_run` reconnect loop; docstring says “Chrome stops intercepting until it reconnects”. `domains.py:308-390` |
| A-09 | Attached `Browser('chrome')` has no Guard and sweeps only agent tabs | CONFIRMED | `_start` early return for `chrome` without profile; `_attached`; `_sweep(only=...)`. `browser.py:899-1010,1042-1073`; `domains.py:456-475` |
| A-10 | MCP `browser_exec` runs in the server process, not the REPL worker | CONFIRMED | `_execute` uses `exec(code, ns)`; `_run_cell` thread; no Repl. `mcp/cli_mcp.py:675-930` |
| A-11 | MCP cloud admin calls refused when domains configured | CONFIRMED | `_refuse_cloud` for `CLOUD_ADMIN` when `self._browser.domains`; cloud browser has no Guard. `mcp/cli_mcp.py:463-472,774-810` |
| A-12 | Delivery budget turn + one repair | CONFIRMED | `delivery_at`, `DELIVER_NOW`, `DELIVERY_REPAIR`; tests. `loop.py:86-98,157-174`; `test_loop.py` |
| A-13 | Last request accepts a delivery needing repair, with warning | CONFIRMED | `ctx.run_step > s.max_steps` branch; test `test_a_delivery_needing_repair_on_the_last_request_is_delivered`. `loop.py:252-335`; `test_loop.py:1194` |
| A-14 | Split delivery semantics | **CONTRADICTED** (corrected) | Package said once/run. Code comment says once/run, but `split_delivery_step` is step-scoped and only skips the same `ctx.run_step`; a later step can be refused again. No second-step test found. `loop.py:139,226-251` |
| A-15 | Judge is non-fatal and does not change status | CONFIRMED | `_judge` returns on failure; `result.judgement=None`; judge tokens/cost added. `service.py:628-657`; `judge.py:143-218` |
| A-16 | Compaction threshold, projection, archive, failure fallback | CONFIRMED | 75%/65%, pinned users, 0600 archive, rejects non-shrinking projection, keeps full history on failure. `loop.py:772-1020`; `test_compaction.py` |
| A-17 | Status, success, judge are independent | CONFIRMED | `AgentHistoryList.__post_init__` clears success unless completed; `is_successful` returns agent verdict. `views.py:84-151`; `loop.py:1301-1361` |
| A-18 | Telemetry is schema-minimized and scrubbed, but not zero-collection | PARTIALLY_CONFIRMED | `_config`/`_capture`/`scrub` confirm minimization; result text, host/path, error text can still leave. No runtime PostHog capture was performed. `service.py:742-1000`; `stats.py:175-557` |
| A-19 | Prior Corpus license metadata is wrong (Apache-2.0 vs MIT) | CONFIRMED | `LICENSE` MIT at HEAD and `d8110c5`; `pyproject.toml` MIT. Prior metadata is external and says Apache-2.0. |
| A-20 | Guard start failure applies to every mode with domains | PARTIALLY_CONFIRMED (corrected) | Attached `Browser('chrome')` returns before Guard creation; the guarantee applies only when a Guard is instantiated. `browser.py:899-1000` |
| A-21 | Guard does not see `Runtime.evaluate` navigation | PARTIALLY_CONFIRMED (corrected) | Guard’s request/target events do cover JS location changes and `Target.createTarget`; the real gap is attached Chrome, which has no Guard. EK-13 boundary was clarified. |
| A-22 | “One key covers model, cloud browser, CLI, and MCP” | PARTIALLY_CONFIRMED | Docs state this; code confirms `BROWSER_USE_API_KEY` fallback and stored harness auth, but not every CLI/MCP path was exercised. Treat as product intent, not a proof. `auth.py:167-222`; README |
| A-23 | Tests were executed and passed | MISSING | `pytest` collection failed: `ModuleNotFoundError: pydantic_ai`. No pass/fail/coverage number exists. |
| A-24 | `browser-harness 0.1.13` helper enforcement | MISSING | Dependency not installed or read. Layer 1 coverage is an external contract. |
| A-25 | `1.0.8rc1` vs any `1.0.8` GA | MISSING | No external release/tag metadata checked. HEAD is an rc. |
| A-26 | Security threat model of secrets/Guard is fully validated | NEEDS_HUMAN_REVIEW | Code and docs establish residual channels, but a human security review is needed for threat modeling, especially screenshots, `open()` writes, and attached Chrome. |
| A-27 | `Browser('chrome')` attach vs `Browser('chrome', profile=...)` are distinct | CONFIRMED | Attach returns early and uses the shared daemon; profile mode copies a profile and starts an owned Chromium with Guard. `browser.py:899-1000`; `local.py:260-332` |
| A-28 | MCP path lacks `sensitive_data`, `type_secret`, worker env allowlist, and process group | CONFIRMED | MCP executes in-process and never constructs `Repl`; its namespace has no secret injection. `mcp/cli_mcp.py:675-930` |

## Three strong confirmations

1. **Preflight ordering**: model/key/provider checks genuinely precede browser start, and a start failure becomes `status='error'` rather than an uncaught exception. This supports KO-04.
2. **Layered domain enforcement with a named exception**: Guard is a second CDP connection using `Fetch.enable` on main-frame documents; attached Chrome returns before Guard creation and sweeps only agent tabs. This supports KO-02 and prevents the false claim “all modes are guarded”.
3. **Compaction is evidence-preserving**: the summary is projected, but covered messages are written to a redacted archive and the projection is rejected if it is not smaller. This supports KO-07 and the “not a deletion” distinction.

## Three errors found

1. **Judge order (A-01)**: the Package placed judge before shutdown. Actual code captures the final page before stop, stops the browser, then judges. Impact: a reader could mis-model when the browser is alive and when the judge request is sent. Corrected.
2. **Split-delivery scope (A-14)**: the Package repeated the source comment “once per run”, while the implementation’s guard is step-scoped and can fire again on a later step. Impact: overstated the one-shot nature. Corrected, and flagged as a comment/code mismatch.
3. **Guard start guarantee (A-20)**: “if the Guard cannot start, Browser start raises” omitted the attached-Chrome early return. Impact: overbroad enforcement claim. Corrected to “when a Guard is requested/instantiated”.

## Omissions / blind spots

- `browser_use/browser/local.py` profile-copy and watchdog internals were not read end-to-end; only `require_debug_port`, `chrome_args`, and the key lifecycle were rechecked.
- `browser_use/mcp/cli_mcp.py` `_Cell` thread lifecycle and `browser_screenshot` path were read at entry points, not every branch.
- `browser_use/telemetry/recorder.py` and feedback consent semantics were not read end-to-end.
- `tests/ci/test_browser.py`, `test_agent.py`, `test_local_browser.py` assertions were not read line-by-line; only test names and selected tests were used.
- `browser-harness 0.1.13` internals were not audited.
- No runtime evidence exists for Chrome, cloud, CAPTCHA, proxy, live view, MCP transport, or model calls.
- README benchmark claims were not converted into numeric claims.

## Counterexamples that survive the audit

| Claim | Counterexample / limit | Status |
|---|---|---|
| “Delivery is checked” | last request accepts a placeholder or missing file after a warning | confirmed code path; not a delivery guarantee |
| “Secrets are safe” | worker can read parent env/files; screenshots and `open()` writes are outside redaction | confirmed residual channels; not a sandbox |
| “Domains are enforced” | Guard disconnect is fail-open; attached Chrome has no Guard; MCP cloud has no Guard | confirmed mode/exception matrix |
| “Completed means success” | `completed` with `success=False`; `success=None` if no verdict | confirmed |
| “Compaction preserves evidence” | archive exists only when compaction succeeds; retention is workspace-bound | confirmed limit |
| “One key covers everything” | provider-specific keys and proxy/unknown clients are outside preflight; docs are not runtime proof | partially confirmed |

## Epistemic audit

| Attack | Result |
|---|---|
| Single case → Pattern | KO-01/02/03/04/05/06/07/08/09 are explicitly scoped to browser-use 1.0.8rc1 and its modes; no global “all browser agents” claim. |
| Pattern → L4 | KO-10 and KO-11 are `Hypothesis`, not `Principle`. They are models, not facts. |
| Project experience → General Principle | KO-12 is L5 Methodology with `Cross-project validation pending`; it is not a validated Principle. |
| ADR/docs → implementation fact | Two mismatches found: judge order in the Package (not docs), and split comment vs code. Docs were not used as implementation proof. |
| Flow edge reality | 68 edges; key conditional edges (Guard fail-open, attached Chrome, MCP cloud, delivery last-request) were rechecked against code. |
| Bypass/override/exception | Attached Chrome, Guard disconnect, MCP in-process execution, last-request delivery, and worker residual channels are all recorded. |
| Epistemic status confusion | No Fact promoted to Pattern without a scope; no L5 written as Principle. |

## Benchmark case for the next round

**Case B-01 — “enforcement matrix” task**: run the same benign task on (a) browser-use 1.0 `Browser('chromium')`, (b) browser-use attached `Browser('chrome')`, (c) `browser-harness` direct, and (d) Playwright MCP, with one allowed domain, one prohibited domain, and a scoped secret. Record: which layer blocked a redirect, whether a JS `location` change was blocked, whether a screenshot/`open()` write exposed the secret, and whether a last-request bad delivery was accepted. This would test KO-02/KO-03 without relying on vendor claims.

## Audit conclusion

`PARTIAL` package quality gate: the package is structurally complete and its main claims are code-grounded, but it cannot be `PASS` because (1) no tests ran, (2) two ordering/scope errors required correction, (3) dependency and runtime paths remain unaudited. The correct disposition is “publish with explicit limitations and a human-review item”, not “all green”.

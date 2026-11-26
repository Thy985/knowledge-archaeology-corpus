# 03 Knowledge Layer — browser-use 1.0.8

The ladder used here is strict: L1 Fact, L2 Observation, L3 Pattern, L4 Cognitive Model, L5 Principle. A statement is promoted only when its aggregation rule is satisfied and a counterexample search has been run. No L5 principle is promoted from a single project; the strongest cross-project candidates are kept in `05_candidates` as hypotheses.

## L1 — Facts (selected, not exhaustive)

L1 facts are the EK entries. Representative facts used below:

- F1: the model-facing built-ins are `python`, `finish`, `finish_from_repl`; there is no DOM/action schema (EK-03).
- F2: the REPL worker is a separate process with an allowlisted env, a loopback token, and kill/restart on timeout (EK-04, EK-05, EK-36).
- F3: the browser is owned by a browser-harness daemon that can outlive the REPL (EK-06).
- F4: the delivery turn starts at 90% of the budget; `max_steps` is a request limit (EK-14, EK-17).
- F5: `success` is agent-authored; `judgement` is a separate request from the same model; `is_successful()` reads only `success` (EK-25, EK-26, EK-29).
- F6: redaction covers selected output channels, not the worker’s filesystem/process access or screenshots (EK-37, EK-39).
- F7: domain rules block main-frame documents and sweep tabs, but not iframes/subresources/Python networking (EK-41, EK-42).
- F8: local Chrome debug ports are loopback and unauthenticated; profile copies carry cookies (EK-44, EK-45).
- F9: context is managed by server-side compaction or a client-side summary checkpoint, plus a 20/2 image prune (EK-20, EK-30, EK-31).
- F10: the default browser is cloud and the default model is `bu-4.0`; one API key covers both (EK-10, EK-46, EK-47).

## L2 — Observations

- O1: the 1.0 rewrite removes the intermediate representation and with it the action-level validation surface. The replacement is not a stricter schema; it is a process plus a browser plus a budget.
- O2: the system’s safety language is unusually precise about what is *not* protected. `MIGRATING.md` says redaction is not a sandbox, domain rules are not a sandbox, and local attachment is an ambient authority.
- O3: the verdict layer is bifurcated: the agent’s own `success` drives the convenience API, while the judge is a second model request over the same trace. The judge is evidence about the trace, not an independent execution.
- O4: failure handling is designed around preserving deliverable state, not preserving Python state: the worker can die, variables are lost, but files/checkpoints/browser survive.
- O5: the compatibility layer is extensive but shallow: names and constructors are preserved; control flow is not.
- O6: the product default is a managed cloud runtime, while local Chrome is an explicit opt-in with a documented local-control-plane exposure.

## L3 — Patterns

### KO-01 — Code-as-action runtime
**Layer**: L3 Pattern  
**Statement**: In 1.0.8, the action space is model-authored Python executed in a persistent REPL over raw CDP; the DOM serializer, typed action schema, and per-action registry are gone. The model’s leverage comes from code, and its observability comes from what it prints/screenshots, not from a structured page model.  
**Aggregation rule**: R2 convergence (independent code paths: prompt, tool registration, REPL, browser helpers, tests all lack an action schema).  
**Derivation facts**: EK-01, EK-02, EK-03, EK-04, EK-05, EK-06, EK-07, EK-49, EK-52.  
**Evidence**: `agent/prompt.py`; `agent/loop.py:178-220`, `1122-1180`; `repl/worker.py:1-20`; `tools.py:1-30`; absence of serializer/action modules in tracked tree.  
**Confidence**: High for 1.0.8.  
**Boundary/counterexample**: the public facade still exposes 0.x `Tools` and `ActionResult`, so an observer could mistake compatibility names for a surviving action layer. The actual built-ins are only `python`/`finish`/`finish_from_repl`.  
**Epistemic note**: Pattern, not a claim that code-as-action is universally better.

### KO-02 — Three-plane execution split
**Layer**: L3 Pattern  
**Statement**: The runtime separates (a) a parent control plane that holds keys, budgets, and delivery; (b) a worker execution plane that runs model code and can be killed/restarted; and (c) a browser-harness daemon that owns the browser and can outlive either. Each plane has different failure and authority semantics.  
**Aggregation rule**: R1 causal chain (parent creates worker → worker calls daemon → each plane has a distinct lifecycle).  
**Derivation facts**: EK-04, EK-05, EK-06, EK-11, EK-12, EK-36, EK-39.  
**Evidence**: `repl/session.py:304-560`; `repl/worker.py:1142-1200`; `browser/browser.py:899-999`; `local.py:548-640`.  
**Confidence**: High.  
**Boundary**: “execution split” is not “security isolation”: the worker shares the user’s UID and can reach the parent’s environment (EK-39). The split buys failure containment and variable isolation, not confidentiality from the model.  
**Counterexample search**: a cell that writes to the parent’s files or reads `/proc/<ppid>/environ` succeeds; therefore the plane boundary is not a confidentiality boundary.

### KO-03 — Budget-to-delivery ladder
**Layer**: L3 Pattern  
**Statement**: The run degrades through a fixed ladder rather than a single timeout: cell deadline → delivery deadline (90% of budget) → delivery-only turn → forced delivery tool → partial checkpoint/last text fallback. Each rung preserves a different kind of state (files/checkpoints, then transcript text).  
**Aggregation rule**: R2 convergence (the deadline is computed in `Session`, enforced in `python`, enforced again in `BrowserLoop`, and consumed in `run`’s fallback).  
**Derivation facts**: EK-13, EK-14, EK-15, EK-16, EK-17, EK-22, EK-33, EK-34.  
**Evidence**: `agent/loop.py:128-220`, `426-520`, `1338-1365`; `tests/ci/test_loop.py:71-130`, `502-520`.  
**Confidence**: High.  
**Boundary**: the ladder governs the agent loop, not browser start/stop; cloud creation and downloads sit outside it. A cell in an uninterruptible syscall can still outlive its nominal deadline.  
**Counterexample**: a run with `timeout=25` still gets a delivery turn because `delivery_at` leaves at least half the budget; this is a deliberate exception, not a violation.

### KO-04 — Delivery is a validated contract, not a text answer
**Layer**: L3 Pattern  
**Statement**: A run’s result is only delivered through `finish`, `finish_from_repl`, or `on_text`; split delivery is rejected once, placeholder/empty JSON is repaired once, and missing workspace files referenced by the answer are repaired once. The last request warns and delivers rather than looping.  
**Aggregation rule**: R1 causal chain (finalizing → delivery tool choice → check_split → check_delivery → result).  
**Derivation facts**: EK-16, EK-22, EK-23, EK-24, EK-25.  
**Evidence**: `agent/loop.py:226-405`, `426-520`; `tests/ci/test_loop.py:385-501`.  
**Confidence**: High.  
**Boundary**: the checks are structural/lexical, not semantic; a well-formed but wrong answer passes. The contract raises the floor, not the ceiling.  
**Counterexample**: a JSON object with a non-empty but fabricated value passes `check_delivery`; nothing verifies it against execution evidence.

### KO-05 — Two verdicts, one authority
**Layer**: L3 Pattern  
**Statement**: The agent’s `finish(success=...)` verdict and the judge’s `Judgement` are parallel channels. `is_successful()` returns the agent’s verdict; the judge does not override it, and judge failure leaves the run unchanged. The judge is a second request from the same model over the same trace, often sharing the actor prefix for prompt cache, so it is a second opinion, not independent verification.  
**Aggregation rule**: R3 invariant (the separation holds across `judge=True/False`, judge failure, cached/standalone paths, and `status != completed`).  
**Derivation facts**: EK-25, EK-26, EK-27, EK-28, EK-29.  
**Evidence**: `agent/views.py:105-150`; `agent/judge.py:200-276`; `agent/service.py:628-656`; `tests/ci/test_judge.py:61-120`.  
**Confidence**: High for the code behavior.  
**Boundary**: “not independent” is a statement about model/context/prefix, not about the judge’s usefulness. A same-model judge can still catch inconsistencies the actor ignored.  
**Counterexample**: a judge can return `success=False` while `result.success=True`; both are exposed, and `is_successful()` still returns True. This is exactly the authority gap.

### KO-06 — Redaction is not isolation
**Layer**: L3 Pattern  
**Statement**: Secret handling has a real routing mechanism (`type_secret`, domain-scoped values, TOTP conversion) and a broad output redactor, but the worker still runs as the user and can read files, process environments, screenshots, and page values. Redaction is a display/output boundary, not a confidentiality boundary.  
**Aggregation rule**: R3 invariant (holds across cell output, checkpoints, custom tool results, final result fields, and transcript redaction).  
**Derivation facts**: EK-35, EK-36, EK-37, EK-38, EK-39, EK-40.  
**Evidence**: `repl/session.py:42-88`, `135-200`; `repl/worker.py:840-897`, `1016-1034`; `MIGRATING.md` trust model; `tests/ci/test_repl.py:91-128`.  
**Confidence**: High.  
**Boundary**: the worker env allowlist does reduce accidental leakage through inherited variables; it does not stop deliberate reads. Screenshots and `js()` are explicitly unredacted.  
**Counterexample**: a model cell can read `~/.config/browser-harness/auth.json` or `/proc/<ppid>/environ` and print it through a transform the redactor does not recognize; the output filter is not a sandbox.

### KO-07 — Navigation allowlist is not network isolation
**Layer**: L3 Pattern  
**Statement**: `allowed_domains`/`prohibited_domains`/`block_ip_addresses` are enforced at three points — helper refusal, main-frame `Fetch` interception, and after-cell tab sweep — but subresources, iframes, speculation/prerender/bfcache paths, and the REPL’s own Python HTTP are outside the guard. The policy is a navigation guard for an aligned model, not a network sandbox.  
**Aggregation rule**: R3 invariant (the scope statement is repeated in code comments, `MIGRATING.md`, and the tests’ distinction between browser and Python networking).  
**Derivation facts**: EK-41, EK-42, EK-43, EK-39.  
**Evidence**: `domains.py:1-24`, `281-455`; `browser/browser.py:1044-1072`; `tests/ci/test_domains.py:549-730`.  
**Confidence**: High.  
**Boundary**: the sweep catches tabs that land on disallowed pages even when the Guard missed the request; it does not undo data already fetched by a subresource.  
**Counterexample**: page JavaScript can `fetch()` an attacker host inside an allowed main frame; the Guard does not see it. The model’s own `requests.get()` is not a browser navigation at all.

### KO-08 — Local control plane is an authority grant
**Layer**: L3 Pattern  
**Statement**: Local browser modes create an unauthenticated loopback control plane. A launched Chromium/profile copy exposes a DevTools WebSocket to any local account for the browser’s lifetime; attaching to the user’s Chrome extends that to the user’s logged-in session. The code hardens the daemon directory and refuses port guessing, but does not add authentication to the endpoint.  
**Aggregation rule**: R3 invariant (holds for `chromium`, `chrome`, and profile-copy modes; the difference is which cookies/session are exposed, not whether authentication exists).  
**Derivation facts**: EK-07, EK-09, EK-44, EK-45, EK-46, EK-48.  
**Evidence**: `browser/browser.py:1-9`, `420-487`; `local.py:66-156`, `332-470`; `MIGRATING.md`; `tests/ci/test_browser.py:82-120`.  
**Confidence**: High for code behavior; the exact Chrome approval UX varies by platform/version.  
**Boundary**: the runtime-dir 0700 check prevents a different local user from planting a daemon socket; it does not prevent that user from connecting to the Chrome debug port.  
**Counterexample**: a second local account can connect to `ws://127.0.0.1:<port>/devtools/browser/...` and read cookies; the code has no token check at that endpoint.

### KO-09 — Two context-degradation paths
**Layer**: L3 Pattern  
**Statement**: Context pressure is handled either server-side (`LazyCompaction` at `min(150k, 65% window)`) or client-side (`Compaction` at 75% chars/65% tokens, keeping the last two groups, summarizing older history into a checkpoint). Images are pruned in one batch at 20 results, keeping 2. The paths have different evidence-retention properties.  
**Aggregation rule**: R1 causal chain (threshold detection → compaction → checkpoint/usage accounting → next request).  
**Derivation facts**: EK-20, EK-30, EK-31, EK-32, EK-34.  
**Evidence**: `agent/loop.py:719-900`, `1055-1100`; `tests/ci/test_loop.py:118-130`.  
**Confidence**: High.  
**Boundary**: server-side compaction is provider-managed and opaque; client-side compaction is inspectable but lossy. Neither is a semantic memory guarantee.  
**Counterexample**: a summary can drop a detail that later delivery needs; the checkpoint archive retains evidence but does not re-inject it automatically.

### KO-10 — Managed cloud is the product’s default runtime
**Layer**: L3 Pattern  
**Statement**: The default constructor is `Browser()` = cloud, the default model is `bu-4.0`, and one Browser Use API key covers both. Local Chrome is an explicit alternative with a documented local-control-plane warning. The architecture is therefore optimized for a managed runtime first, local execution second.  
**Aggregation rule**: R4 scope generalization within the project (code defaults, README positioning, billing/lifecycle code, auth precedence, and tests all point to the same default).  
**Derivation facts**: EK-01, EK-07, EK-10, EK-46, EK-47, EK-51.  
**Evidence**: `browser/browser.py:1000-1125`; `llm/__init__.py:33`; `README.md`; `auth.py:76-150`.  
**Confidence**: Medium-high.  
**Boundary**: this is a product-architecture pattern, not a judgment about which runtime is better. The cloud default also concentrates authority in a bearer API key.  
**Counterexample**: a user who always passes `Browser('chromium')` and `llm=...` never touches the cloud path; the default is a distribution choice, not a hard dependency.

## L4 — Cognitive Models

### KO-11 — Action space determines governance surface
**Layer**: L4 Cognitive Model  
**Statement**: When an agent’s action space is typed (DOM elements + named actions), governance can live at the action-schema layer: validate each action, allow/deny each action, observe each step. When the action space is code, that layer disappears; governance must move to the execution plane (process/user/filesystem), the browser plane (attach mode, CDP endpoint, domain policy), and the budget/delivery plane. A team that keeps looking for an “action allowlist” in a code-as-action agent will miss the actual authority boundaries.  
**Aggregation rule**: R3 invariant across the project’s independent controls: the absence of an action schema (EK-03), the worker’s OS-level authority (EK-36/EK-39), the browser’s attach authority (EK-44), domain policy’s main-frame scope (EK-42), and the budget/delivery gate (EK-14/EK-16).  
**Derivation facts**: EK-03, EK-04, EK-05, EK-14, EK-16, EK-36, EK-39, EK-42, EK-44.  
**Evidence**: as above; the model explains why the security documentation spends its words on process/filesystem/attach/domain scope rather than on action validation.  
**Confidence**: Medium. This is a model inferred from one architecture transition, not a tested universal.  
**Boundary**: it does not say code-as-action is less safe; it says the control surface is different. A code-executing agent can be safer if the execution plane is sandboxed (container/VM), which this project explicitly recommends for untrusted tasks.  
**Counterexample**: a code-as-action agent that runs in a locked-down container with no host mounts and a network policy can have a stronger boundary than a DOM-action agent running in the user’s browser. The model is about *where* governance lives, not its strength.

### KO-12 — Boundary-confusion model
**Layer**: L4 Cognitive Model  
**Statement**: Agent runtimes often use one word — “sandbox”, “allowlist”, “guard”, “redaction” — for controls with different threat models. A useful design review separates four boundaries: (1) confidentiality (can the model read the secret?), (2) execution containment (can the model affect the host?), (3) navigation/network policy (where can the browser go?), and (4) authority grant (whose session/credentials does the browser carry?). browser-use 1.0.8 is explicit that redaction is not (1), the worker is not (2), domain rules are not (3), and local attach is a (4).  
**Aggregation rule**: R3 invariant across the project’s security controls; each control is documented with an explicit “not a sandbox” scope, and the four categories are independently evidenced.  
**Derivation facts**: EK-36, EK-37, EK-39, EK-40, EK-41, EK-42, EK-44, EK-45, EK-46.  
**Evidence**: `MIGRATING.md` trust model and domain/attach sections; `repl/session.py:1-18`; `domains.py:1-24`; `browser/local.py:104-145`.  
**Confidence**: Medium-high as a diagnostic model; cross-project validation pending.  
**Boundary**: the four categories are not mutually exclusive; a container can provide (1) and (2), while domain rules provide a weak (3). The model is a checklist, not a formal taxonomy.  
**Counterexample**: an OS-level sandbox can make redaction unnecessary for confidentiality, but not for accidental leakage into logs/telemetry; the categories interact.

## L5 — Principles

**No L5 principle is promoted in this run.** The two L4 models are strong within-project explanations, but the corpus contains only one 1.0.8 refresh and one historical 0.13.10 baseline of the same project. Promoting “action space determines governance surface” or “boundary confusion is universal” to L5 would require at least one independent project with a contrasting action space (for example a typed-action agent with a real sandbox) and a counterexample search across that project. Those are recorded as cross-project hypotheses in `05_candidates`, not as principles.

## KO summary

| KO | Layer | Aggregation | Core claim |
|---|---|---|---|
| KO-01 | L3 | R2 | Code is the action space; no DOM/action schema |
| KO-02 | L3 | R1 | Parent / worker / daemon planes have distinct lifecycles |
| KO-03 | L3 | R2 | Budget degrades through a delivery ladder |
| KO-04 | L3 | R1 | Delivery is a validated contract |
| KO-05 | L3 | R3 | Agent verdict and judge verdict are separate; judge is not independent |
| KO-06 | L3 | R3 | Redaction is not isolation |
| KO-07 | L3 | R3 | Navigation allowlist is not network isolation |
| KO-08 | L3 | R3 | Local debug port is an ambient authority grant |
| KO-09 | L3 | R1 | Two context-degradation paths with different retention |
| KO-10 | L3 | R4 | Managed cloud is the default runtime |
| KO-11 | L4 | R3 | Action space determines where governance lives |
| KO-12 | L4 | R3 | Four distinct boundaries are often conflated |

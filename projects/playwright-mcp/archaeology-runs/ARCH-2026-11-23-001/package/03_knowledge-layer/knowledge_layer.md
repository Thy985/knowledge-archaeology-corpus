# 03 · Knowledge Layer — L1 → L5

The ladder is applied strictly. A claim is not promoted because it sounds elegant; it is promoted only when the evidence and the aggregation rule support it. `L3` entries are **templates**, `L4` entries are **provisional models**, and `L5` is **one scoped principle** with explicit cross-project validation pending.

## L1 — Facts (directly observable)

| ID | Statement | Evidence | Provenance | Flow |
|---|---|---|---|---|
| KO-L1-01 | This repository is a release wrapper, not the MCP server implementation | `AGENTS.md:3-20`; `index.js:17`; `cli.js:17-37` | `[REPO]` | Control |
| KO-L1-02 | Defaults: chromium/chrome, Linux-no-DISPLAY headless, 1280×720, AutomationControlled disabled flag | `mcp/config.ts:226-274` | `[DEP]` | Control/Policy |
| KO-L1-03 | Default profile is persistent and keyed by browser token + hash of client cwd | `mcp/browserFactory.ts:193-239`; README:478-488 | `[DEP]` + `[REPO] docs` | Memory |
| KO-L1-04 | HTTP transport has no MCP auth; Host allowlist is DNS-rebinding protection | `utils/mcp/http.ts:57-85`; README:681-685 | `[DEP]` | Authority |
| KO-L1-05 | Default no-`--caps` tool set is exactly 25 tools, including `browser_run_code_unsafe`, `browser_evaluate`, `browser_network_request` | `tests/capabilities.spec.ts:19-48` | `[REPO]` test | Control/Authority |
| KO-L1-06 | Repository tests: 9 cases, 809 lines, packaged server only | `tests/*.spec.ts`; `AGENTS.md` | `[REPO]` | Evidence |
| KO-L1-07 | Docker default is headless chromium, non-root `node`, `--no-sandbox`, stdio, no port | `Dockerfile:12-103` | `[REPO]` | Policy |
| KO-L1-08 | Pinned implementation is `playwright-core@1.66.0-alpha-2026-11-18` @ `8124fc3` | `package.json`; `npm view gitHead` | `[REPO]` + npm | Control |

## L2 — Observations (patterns in code/config, intent not proven)

| ID | Statement | Evidence | Why not L1 | Flow |
|---|---|---|---|---|
| KO-L2-01 | The default control plane exposes both Node-process code execution and page-context arbitrary JS | `runCode.ts:37-96`; `evaluate.ts:32-84`; default test | two primitives, both default, but "intended risk" is not stated | Authority/Policy |
| KO-L2-02 | Default network inspection can return request/response headers and bodies, creating a credential-observation surface | `network.ts:30-102,140-174`; default test | behavior is observable; actual leakage depends on page and client | Data/Evidence |
| KO-L2-03 | `secrets` redaction covers serialized text and text files, but not binary payloads | `response.ts:123-129,174-235` | implementation behavior, not a guarantee | Evidence/Policy |
| KO-L2-04 | File, origin, sandbox and file-access guards are convenience defenses with explicit bypass flags | `context.ts:582-598`; README:713-726; Dockerfile | multiple guards share a "convenience" stance | Policy/Authority |
| KO-L2-05 | `browser_get_config` is gated behind `config` capability and is not in the default set | `config.ts:20-33`; `tools.ts:83-94` | gating is fact; why it is excluded is not documented | Authority/Data |
| KO-L2-06 | CLI `--caps` only accepts `vision`, `pdf`, `devtools`, while config type and env allow more capabilities | `program.ts:42`; `config.ts:387,440`; `config.d.ts` | mismatch is observable; intent unclear | Control/Policy |

## L3 — Patterns (reusable implementation templates)

### KO-L3-01 · Core-always-on + capability opt-in

**Statement.** A tool server can keep a small "core" surface always enabled and put higher-risk or specialized tools behind explicit capability gates; however, if the core itself contains code execution, the gate does not reduce the default risk.

**Evidence.** `backend/tools.ts:83-94`; `tests/capabilities.spec.ts:19-48`; `mcp/program.ts:42`; `config.d.ts`.

**Aggregation rule.** R2: multiple mechanisms (filter predicate, CLI enum, config type, test snapshot) converge on one exposure model.

**Reusable template.** Define a `capability` field per tool; always-on prefix (`core*`); opt-in set; filter before `ListTools`; test the exact default set. The template must also classify "core" by blast radius, not by product importance.

**Counterexample / limit.** If `core` includes `run_code_unsafe`, "core" is a product tier, not a safety tier. This is exactly what happens here.

**Flow traceability.** Control F1 edges E1-E4; Authority F5 edges A1-A3.

---

### KO-L3-02 · Accessibility snapshot + ref indirection

**Statement.** Instead of pixel coordinates, the agent receives a structured aria snapshot; actions target a short-lived `ref` (or selector/locator), and the server resolves the ref against the live page at call time.

**Evidence.** `backend/snapshot.ts:35-55`; `backend/tab.ts:495-548,592-619`; `tests/click.spec.ts:19-48`; README:3,15-17.

**Aggregation rule.** R1: snapshot producer, ref resolver, action tools and tests form one mechanism family.

**Reusable template.** `ariaSnapshot(mode:'ai')` → stable textual tree with refs → `target` schema accepts `ref|selector|locator` → resolve at execution → return a fresh snapshot. It reduces coordinate ambiguity but increases token volume and requires re-snapshot after DOM mutation.

**Counterexample / limit.** Stale refs fail with an explicit error; a changed DOM requires a new snapshot. Coordinate tools still exist behind `--caps=vision`, and `browser_take_screenshot` is default. "No vision" means "not required", not "unavailable".

**Flow traceability.** Data F3 edges D1-D5; Control F1 edges E5-E6.

---

### KO-L3-03 · Workspace-derived persistent profile

**Statement.** A browser-control server can make login state continuous across sessions by deriving a persistent profile directory from the client's workspace root, rather than using a random temp directory.

**Evidence.** `mcp/browserFactory.ts:193-239`; README:472-491.

**Aggregation rule.** R1: `createUserDataDir`, `launchPersistentContext`, README profile docs all describe the same mechanism.

**Reusable template.** `${cache}/<product>/<browserToken>-<hash(cwd)>` → `launchPersistentContext` → one profile per workspace; document that concurrent clients with the same workspace conflict; offer `--isolated` or distinct `--user-data-dir` for parallelism.

**Counterexample / limit.** The profile is not encrypted; all logged-in state persists on disk. `isolated` + `userDataDir` is rejected. The profile lock is retried 5×1s, then errors.

**Flow traceability.** Memory F6 edges M1-M3; State F2 edges S1-S4.

---

### KO-L3-04 · Modal-state exclusivity gate

**Statement.** When a page interrupt (dialog or file chooser) is open, every normal tab tool is rejected until the designated clearing tool handles it; the gate is implemented as a per-call precondition, not as a prompt instruction.

**Evidence.** `backend/tool.ts:70-88`; `backend/tab.ts:224-237`; `browser_handle_dialog` / `browser_file_upload`.

**Aggregation rule.** R2: modal state set, tool wrapper precondition, and clearing tools converge.

**Reusable template.** `modalStates()` returns a set; each tool declares `clearsModalState`; `defineTabTool` errors if a modal exists and the tool does not clear it; action waits race against modal states.

**Counterexample / limit.** The gate is page-level, not user-level. It prevents accidental actions behind a modal; it does not authenticate the caller.

**Flow traceability.** Policy F7 edges P1-P3; State F2 edges S5-S6.

---

### KO-L3-05 · Attached-vs-owned browser lifecycle

**Statement.** The server distinguishes browsers it owns (launch/launchPersistent) from browsers it attaches to (CDP/extension/remote); ownership determines idle timeout, close semantics, and security posture.

**Evidence.** `mcp/browserFactory.ts:66-96,118-146,193-244`; `mcp/cdpRelay.ts`; `mcp/extensionContextFactory.ts`.

**Aggregation rule.** R2: factory branches, ownership field, idle timer condition, extension relay.

**Reusable template.** `ownership: 'own'|'attached'`; own browsers get launch options, idle timeout and close; attached browsers get connect options, no default idle timeout, and must not kill the user's browser.

**Counterexample / limit.** Attached mode inherits the user's real profile and cookies, so isolation is weaker by design. Extension mode introduces a local relay and token path.

**Flow traceability.** State F2; Authority F5; Memory F6.

---

### KO-L3-06 · Thin wrapper + pinned upstream bundle

**Statement.** A release repository can own only CLI/types/tests/Docker while all implementation comes from a pinned dependency; a roll script updates the pin and generated docs.

**Evidence.** `AGENTS.md:3-20`; `roll.js:7-84`; `package.json:40-43`; `src/README.md`.

**Aggregation rule.** R1: ownership doc, wrapper imports, roll script, placeholder source.

**Reusable template.** Keep a public wrapper repo; pin an exact upstream build; copy public config types; generate docs from the bundle; test the packaged artifact; never claim implementation facts from the wrapper alone.

**Counterexample / limit.** The pin can lag upstream; alpha versions can change behavior; fixes must go upstream and roll in.

**Flow traceability.** Control F1; Evidence F4.

## L4 — Cognitive models (provisional)

### KO-L4-01 · Default-open control plane

**Type.** Cognitive Model / Hypothesis.

**Statement.** When a browser-control server exposes arbitrary code execution, page JS, and credential-reading network tools in its default surface, and its remote transport has no authentication, the effective trust boundary is the **client and transport**, not the tool list or in-tool guards. Capability gating reduces accidental exposure, not adversarial risk.

**Evidence chain.** EK-12 (no auth) + EK-14 (25 default tools) + EK-15 (RCE) + EK-16 (page JS) + EK-17 (network headers) + EK-24 (file guard bypass) + EK-25 (redaction convenience).

**Why L4 and not L5.** It is a compact model that explains several mechanisms at once, but it is derived from one project. It needs at least one more browser-control project (browser-use, browser-harness) and a runtime test before becoming a principle.

**Counterexample.** A deployment can wrap the server in an authenticated reverse proxy and restrict `--caps`; then the effective boundary moves to the proxy. The model predicts risk only when no such external boundary exists.

**Flow traceability.** Authority F5, Policy F7, Control F1.

---

### KO-L4-02 · Continuity-by-default, isolation-by-exception

**Type.** Cognitive Model / Hypothesis.

**Statement.** Browser-control products often optimize for session continuity (persistent profile, attached browser) and make isolation an explicit opt-in; this improves real-task success but turns "whose cookies am I using?" into a configuration question. Safety therefore depends on the operator choosing isolation, not on the default.

**Evidence chain.** EK-06 (chrome default) + EK-07 (persistent profile) + EK-08 (attach precedence) + EK-09 (isolated opt-in).

**Why L4 and not L5.** Supported by this project's defaults and docs, but not yet compared with browser-use/browser-harness defaults.

**Counterexample.** A user always launches with `--isolated` and a fresh `--user-data-dir`; then continuity is not the effective behavior. The model is about defaults, not possibilities.

**Flow traceability.** Memory F6, State F2.

## L5 — Principle (scoped, one only)

### KO-L5-01 · Convenience guards are not security boundaries

**Type.** Principle.

**Statement.** In agent/browser tooling, in-tool guards (secret redaction, origin filters, workspace file roots, host allowlists used as auth) must be treated as **accident prevention**, not as a security boundary; enforce trust at the transport, client identity, and process/network isolation layers.

**Evidence.**

- README: "Playwright MCP is **not** a security boundary" (`README.md:873-875`).
- `secrets` and `redactUrlParams` are explicitly "a convenience and not a security feature" (`README.md:713-726`).
- `allowUnrestrictedFileAccess` is documented as convenience (config comment).
- `allowedOrigins` "does not serve as a security boundary and does not affect redirects" (`README.md:409-410`).
- `origin='code'` bypasses file guard (`context.ts:582-598`).
- HTTP has no auth (`http.ts`).

**Scope.** Browser-control servers and agent tools with arbitrary page/JS/network access. Cross-project validation pending; do not extend it to systems with a documented, tested security boundary.

**Operational form.**

1. Put authentication/authorization at the transport or a trusted proxy.
2. Treat default code-execution tools as RCE and gate them outside the model's control.
3. Run the browser in a separate process/container/account when handling untrusted pages.
4. Do not rely on redaction, origin filters or file roots to stop a determined caller.

**Counterexample / limit.** If a product has a real OS sandbox and a verified capability model, some in-tool guards can be part of defense-in-depth. This principle says "do not treat them as the boundary", not "they are useless".

**Flow traceability.** Authority F5, Policy F7.

## Promotion audit

| Candidate | From | To | Verdict | Reason |
|---|---|---|---|---|
| Default 25 tools | L1 | L1 | keep | exact test fact |
| RCE default | L1/L2 | L4 | keep at L2 + model in L4-01 | exploitability not demonstrated |
| A11y snapshot | L3 | L4 | **not promoted** | single-project implementation template |
| Persistent profile | L3 | L4 | **not promoted** | single-project default; needs cross-project comparison |
| "No vision" | claim | L3 | **downgraded** | screenshot default + vision opt-in; "not required" only |
| Extension token auth | observation | L2 | **downgraded** | relay does not compare token server-side; extension side unverified |
| Convenience guards | L2 | L5 | promoted with scope | explicitly documented + multiple bypasses; still scoped |

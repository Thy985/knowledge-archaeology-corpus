# Independent Audit — playwright-mcp @ 2337aff

## Method

Independent read-only reconstruction executed **before** the formal package was written. The auditor did not use the formal package as a source of truth. It re-read:

- target repository `repo/` (44 files, HEAD `2337aff`, tag `v0.0.86`)
- pinned implementation dependency at upstream commit `8124fc3` (npm `gitHead` of `playwright-core@1.66.0-alpha-2026-11-18`), specifically `packages/playwright-core/src/tools/{mcp,backend,utils}`
- compiled `dep/playwright-core/lib/coreBundle.js` as a cross-check
- tests, CI, README, Dockerfile, `AGENTS.md`, `SECURITY.md`, `CONTRIBUTING.md`

No code was executed; no browser was launched; no test was run.

## Independent findings

| ID | Finding | Evidence | Epistemic status |
|---|---|---|---|
| A-01 | This repository is a thin wrapper/release repo; MCP implementation is upstream `packages/playwright-core/src/tools` | `AGENTS.md:3-20`; `index.js:17`; `cli.js:17-37`; `src/README.md` placeholder | **Fact** |
| A-02 | There is no build step; `npm run build` is `echo OK` | `package.json:24` | **Fact** |
| A-03 | Runtime pins `playwright`/`playwright-core` to `1.66.0-alpha-2026-11-18`; upstream commit is `8124fc3` | `package.json:40-43`; `npm view ... gitHead` | **Fact** |
| A-04 | Default tool surface (no `--caps`) is exactly 25 tools | `tests/capabilities.spec.ts:19-48` | **Fact** |
| A-05 | `browser_run_code_unsafe` is in the default set and is described as RCE-equivalent; it executes through Node `vm` | `tests/capabilities.spec.ts:42`; `backend/runCode.ts:37-44,79-89`; README:1134-1136 | **Fact** |
| A-06 | `browser_evaluate` is in the default set and executes arbitrary page-context JavaScript | `tests/capabilities.spec.ts:27`; `backend/evaluate.ts:32-64` | **Fact** |
| A-07 | `browser_network_request` is core/default and can return request headers and response headers/bodies | `backend/network.ts:30-34,74-87,140-174`; `tests/capabilities.spec.ts:38-39` | **Fact** |
| A-08 | Default browser is `chromium` with channel `chrome`; Linux without `DISPLAY` becomes headless; headless viewport 1280×720 | `mcp/config.ts:226-265`; README:470-486 | **Fact** |
| A-09 | Default profile is persistent and derived from browser token + hash of client cwd, not a temp dir | `mcp/browserFactory.ts:193-239`; README:478-488 | **Fact** |
| A-10 | Browser factory precedence: remote → CDP → isolated → extension → persistent | `mcp/browserFactory.ts:66-87` | **Fact** |
| A-11 | `isolated` and `userDataDir` are mutually exclusive | `mcp/config.ts:245-246` | **Fact** |
| A-12 | HTTP transport has no authentication; it has a Host header allowlist for DNS-rebinding protection | `utils/mcp/http.ts:57-85`; README:681-685 | **Fact** |
| A-13 | `--host 0.0.0.0` still advertises `localhost` and defaults the allowlist to localhost/loopback/bind host; arbitrary external Host headers are rejected unless configured | `utils/mcp/http.ts:48-53,63-70`; `mcp/program.ts:57` | **Fact** |
| A-14 | README's Docker section says the server "can be reached by any MCP client", while the same code path applies the Host allowlist | README:896-904; `utils/mcp/http.ts:63-70` | **Contradiction / documentation risk** |
| A-15 | `browser_get_config` is `config` capability, not default; it returns the full resolved config as JSON | `backend/config.ts:20-33`; `backend/tools.ts:83-94` | **Fact** |
| A-16 | Cookie/storage/route tools are `storage`/`network` capability, not default; CLI `--caps` only documents/validates vision/pdf/devtools | `backend/cookies.ts:21-22`; `backend/storage.ts:21-22`; `backend/route.ts:87-88`; `mcp/program.ts:42` | **Fact** |
| A-17 | `secrets` redaction is applied to all serialized text and text files, but not to binary output | `backend/response.ts:123-129,174-235`; `backend/context.ts:529-538` | **Fact** |
| A-18 | File guard is explicitly a convenience defense; `origin='code'` bypasses it | `backend/context.ts:582-598`; README config comment 300-310 | **Fact** |
| A-19 | `--extension` attaches to a real Chrome/Edge via a local WebSocket relay; one extension and one Playwright CDP client | `mcp/cdpRelay.ts:230-280`; `mcp/extensionContextFactory.ts:31-51` | **Fact** |
| A-20 | Extension token is passed in the connect URL; the relay itself does not compare the token | `mcp/cdpRelay.ts:120,158-168,172-193,230-245` | **Observation (server-side validation absent)** |
| A-21 | Modal state is a gate: dialog/file-chooser blocks all tab tools except the designated clearing tool | `backend/tool.ts:70-88`; `backend/tab.ts:224-237` | **Fact** |
| A-22 | Snapshot uses `page.ariaSnapshot({ mode: 'ai' })`; refs are resolved by `page.getByRef`, otherwise selector/locator parsing | `backend/tab.ts:495-548,592-619`; `backend/snapshot.ts:35-55` | **Fact** |
| A-23 | Actions default to returning a new snapshot; `snapshot.mode=none` suppresses it | `backend/response.ts:161-164`; `mcp/config.ts:100-112` | **Fact** |
| A-24 | Tab request history is capped at 200 earlier-navigation requests | `backend/tab.ts:83-84,273-278` | **Fact** |
| A-25 | Downloads are saved automatically to an output file with sanitized name; output write uses `origin: 'code'` | `backend/tab.ts:239-252` | **Fact** |
| A-26 | Crash recovery closes the crashed page, clears current tab, creates a new one, logs `about:blank` reset | `backend/context.ts:225-238` | **Fact** |
| A-27 | Owned headless browsers get a 1-hour idle timeout by default; headed/attached get none | `mcp/browserFactory.ts:61-64`; `backend/idleTimer.ts:17` | **Fact** |
| A-28 | Exit watchdog forcibly exits after 15 seconds if graceful close hangs | `mcp/watchdog.ts:21-37` | **Fact** |
| A-29 | Test suite has 9 tests, all against the packaged server, not upstream `tests/mcp` | `tests/*.spec.ts`; `AGENTS.md:31-37`; `CONTRIBUTING.md:39-55` | **Fact** |
| A-30 | No test execution or browser runtime evidence exists in this run | absence of `npm ci`/`npm test` artifacts | **Missing evidence** |

## Attack the inference chain

| Attack | Result | Verdict |
|---|---|---|
| Single case → Pattern: "a11y snapshot" from one tool | Rejected as a pattern claim; the repo has a whole snapshot/ref pipeline plus coordinate tools opt-in. Pattern is limited to "a11y-first default with vision opt-in". | **PARTIALLY_CONFIRMED** |
| Pattern → L4: "default-open control plane" | Supported by default RCE tool + no HTTP auth + persistent profile, but only one project. Must remain L4 **Hypothesis**, not a law. | **DOWNGRADED** |
| Project experience → universal Principle | Rejected. "Convenience guard is not a security boundary" is documented by this project but still needs cross-project validation before L5 generalization; keep L5 scoped. | **DOWNGRADED** |
| ADR → implementation fact | No ADR found. `AGENTS.md` is a contributor/ownership contract, not evidence of runtime behavior. Implementation claims are sourced to code, not prose. | **CONFIRMED** |
| Flow edge reality | Control flow, state flow, data flow and policy flow edges are backed by symbol/file/condition. The "extension token authentication" edge is **not** backed by server-side validation. | **MISSING / PARTIALLY_CONFIRMED** |
| Bypass / override / fallback paths | Found: `origin='code'` bypass; `allowUnrestrictedFileAccess`; `--allowed-hosts '*'`; `--no-sandbox`; `--isolated`; CDP/remote/extension attach modes; `snapshot.mode=none`; `imageResponses=omit/only`; `--caps` opt-ins; `browser_run_code_unsafe`. | **CONFIRMED** |
| Epistemic status confusion | Formal package must not label "RCE-equivalent" as proven exploit; it is a self-description plus a `vm` implementation. Keep as Fact (description/code) + Hypothesis (escape ease). | **NEEDS_HUMAN_REVIEW** |

## Three strongest independent successes

1. **Boundary detection**: correctly identified the target as a thin wrapper and moved implementation evidence to the pinned upstream commit, preventing a false "44-file repo = MCP server" conclusion.
2. **Default-surface reconstruction**: independently derived the exact 25-tool default set from the test, then confirmed the RCE/evaluate/network primitives in source. This is the highest-value safety fact.
3. **Ownership/state model**: reconstructed persistent-profile default, factory precedence, isolated/userDataDir exclusion, idle timer and extension attach path from code, not README claims.

## Three independent errors / corrections

1. **Overstated Docker reachability**: initial reading of README suggested "any MCP client" over the network. Code's Host allowlist means arbitrary external `Host` headers are rejected unless `--allowed-hosts` is widened. Verdict: **CONTRADICTED**; documentation must be read with the host check.
2. **Overstated extension token security**: initial reading treated `PLAYWRIGHT_MCP_EXTENSION_TOKEN` as relay authentication. The relay only passes it in the URL; no server-side comparison was found. Verdict: **DOWNGRADED**.
3. **Overstated "no vision"**: README says no vision models, but the default surface includes `browser_take_screenshot`, and `--caps=vision` adds coordinate mouse tools. "No vision by default" is correct; "vision impossible" is false. Verdict: **OVER_GENERALIZED** → scope tightened.

## Omissions found

- No upstream `tests/mcp` execution; only packaged-server tests (9 cases) are in scope.
- No browser binary / Chrome channel validation; `--browser chrome` may fail on a host without Chrome.
- No proof that `vm` can be escaped; RCE-equivalent is the project's own description and the code exposes powerful globals, but exploitability was not demonstrated.
- No runtime measurement of snapshot token cost or latency; no benchmark comparison with browser-use / WebArena.
- No verification that `@playwright/mcp@0.0.86` on npm exactly matches this git tree; the npm package was not installed.

## Benchmark case (for later comparison)

**Case B-01 — Default-open browser control plane.**

Question: given a browser-automation MCP server, what is the minimum evidence needed to decide whether it is safe to expose to an untrusted client?

Checklist derived independently:
1. transport authn/authz (not just Host allowlist)
2. default tool list and whether it contains code execution / arbitrary page JS / credential-reading primitives
3. default profile persistence and workspace derivation
4. attach modes (CDP/extension/remote) and their trust implications
5. file/network guard semantics and explicit bypass flags
6. whether redaction is a security boundary or convenience
7. test/runtime evidence for the packaged artifact

This case can be applied to `browser-use`, `browser-harness`, `webwright`, and future Computer Use projects. It is a **candidate benchmark**, not a validated universal rubric.

# 05 · Candidates

Candidates are unresolved claims. They are not promoted into Knowledge Objects. Each has an explicit next test and an owner boundary.

| ID | Candidate | Evidence | Status | Next test |
|---|---|---|---|---|
| C-01 | Is `browser_run_code_unsafe` in the default surface an intentional product decision or an oversight? | `tests/capabilities.spec.ts:42`; `backend/runCode.ts:37-44`; README:1134-1136; no SECURITY threat model | **OPEN** | Ask upstream / inspect release notes; compare with `--sandbox`/Docker defaults. Do not label as vulnerability. |
| C-02 | Can Node `vm` be escaped in practice from `browser_run_code_unsafe`? | `vm.createContext` + exposed `Buffer`, `fetch`, `process` not directly passed but host objects reachable; description says RCE-equivalent | **NEEDS_HUMAN_REVIEW** | Controlled local proof only with explicit authorization; not attempted. Treat as RCE-equivalent regardless. |
| C-03 | Does the Docker long-lived command actually accept "any MCP client" as README claims? | README:896-904 vs `http.ts:48-53,63-70`; default allowedHosts = localhost/loopback/bind host | **CONTRADICTED** | Launch container with `--host 0.0.0.0 --port 8931`, send external `Host: <ip>:8931`, expect 403; then set `--allowed-hosts '*'`. |
| C-04 | Is `PLAYWRIGHT_MCP_EXTENSION_TOKEN` validated by the Chrome extension, or only passed through? | `cdpRelay.ts:120,158-193`; relay `_handleExtensionConnection` has no token compare; extension source not cloned | **PARTIALLY_CONFIRMED / OPEN** | Clone `packages/extension/src`, find token compare; or run extension with wrong token and observe connect. |
| C-05 | Does `browser_get_config` expose `cdpHeaders`, proxy credentials or storage state beyond the `secrets` map? | `backend/config.ts:20-33` returns full config; `response.ts` redacts only literal secrets and URL params | **OPEN** | Enable `config` capability, set dummy `cdpHeaders` + proxy auth + storage state, call tool, inspect output. |
| C-06 | When `allowedOrigins` and `blockedOrigins` both match, which wins? Does redirect bypass origin policy? | `context.ts:436-449` registers allowed then blocked; README says blocked wins; docs say redirects unaffected | **OPEN** | Run route test with same origin in both lists; test a 302 to blocked origin and observe request. |
| C-07 | Why does CLI `--caps` accept only `vision/pdf/devtools` while config type/env allow `network/storage/testing/config`? | `program.ts:42`; `config.ts:387,440`; `config.d.ts` | **OPEN — likely design/docs mismatch** | Compare CLI `--caps=network` behavior (expect parser error) vs config file `capabilities:['network']` (expect tools exposed). |
| C-08 | Is "no vision models required" consistent with a default `browser_take_screenshot` and opt-in coordinate tools? | README:15-17; default set includes screenshot; `--caps=vision` adds mouse tools | **DOWNGRADED to scope: "vision not required by default"** | Check screenshot output mode and imageResponses behavior; do not claim vision-impossible. |
| C-09 | Does attached-browser mode (CDP/extension/remote) have different close/dispose semantics that could kill the user's browser? | `browserFactory.ts:66-96`; `ownership` field; `context.dispose` closes isolated contexts only; `browser.close()` path not fully traced | **OPEN** | Inspect `program.ts` shutdown and `BrowserBackend.dispose` for `browser.close()` vs `browser.disconnect()` per ownership. |
| C-10 | Cross-project hypothesis: browser-use 1.0's REPL/CDP route is more token-efficient but less inspectable than Playwright MCP's a11y snapshot route. | Prior Corpus run `ARCH-2026-11-22-001`; this project's README:5-11; no benchmark run | **HYPOTHESIS / DEFERRED** | Run a fixed 10-task benchmark on both; measure tokens, steps, success, recovery. Not today. |
| C-11 | Does npm `@playwright/mcp@0.0.86` exactly match this git tree? | `package.json` is source; npm tarball not installed; `server.json` version aligned | **OPEN** | `npm pack @playwright/mcp@0.0.86`, compare file hashes/contents. |
| C-12 | Are the 9 packaged-server tests sufficient to catch backend regressions? | Tests cover tool list, click/navigate, CLI, library; no run_code, network, extension, HTTP auth, file guard, crash | **MISSING COVERAGE** | Map each EK to a test; add upstream `tests/mcp` coverage evidence. |
| C-13 | Does `--allowed-hosts '*'` plus `--host 0.0.0.0` create an unauthenticated remote browser-control endpoint? | `http.ts:64-70`; no auth; Docker README exposes 0.0.0.0 | **OBSERVATION → L4-01 support** | Runtime test in isolated container; do not scan external networks. |

## Candidate promotion rules

- C-03 is a documentation contradiction, not a code vulnerability; it can become a **Fact** if reproduced.
- C-04 stays a candidate until the extension source is read; do not call it "token auth".
- C-10 is a cross-project hypothesis; it must not be written as a validated Principle.
- C-13 supports KO-L4-01 but does not prove exploitability; keep it as an observation.

# 02 · Engineering Knowledge — EK Graph

## Graph contract

- 30 Engineering Knowledge objects (`EK-01`…`EK-30`), each tied to file:line evidence.
- Edge types: `supports` (evidence → claim), `enables` (mechanism → capability), `constrains` (policy → behavior), `conflicts` (docs vs code), `derived_from` (EK → upstream dependency), `tested_by`.
- EKs are never deleted when a Knowledge Object is formed. KOs reference EKs in `03_knowledge-layer`.
- `confidence`: high = direct code/test; medium = code + docs; low = single observation or doc-only.

## EK table

| ID | Type | Claim | Evidence | Confidence | Edges |
|---|---|---|---|---|---|
| EK-01 | Boundary | This repo is a thin wrapper; implementation is upstream `packages/playwright-core/src/tools` | `AGENTS.md:3-20`; `index.js:17`; `cli.js:17-37`; `src/README.md` | high | supports KO-L1-01; derived_from EK-03 |
| EK-02 | Build | No build step; `npm run build` is `echo OK`; runtime uses published `coreBundle` | `package.json:24`; `index.js:17` | high | supports KO-L1-01 |
| EK-03 | Dependency | `playwright`/`playwright-core` pinned to `1.66.0-alpha-2026-11-18`; npm gitHead `8124fc3` | `package.json:40-43`; `npm view ... gitHead` | high | enables EK-01; supports KO-L1-01 |
| EK-04 | Release | `roll.js` bumps Playwright packages, copies `config.d.ts`, regenerates README, updates Docker Node version | `roll.js:7-84`; `AGENTS.md:22-30` | high | enables EK-03; supports KO-L3-06 |
| EK-05 | Config | Config precedence is defaults → file → env → CLI | `mcp/config.ts:137-154` | high | enables EK-06, EK-07, EK-08 |
| EK-06 | Defaults | Default browser is chromium, channel chrome, Linux/no DISPLAY headless, 1280×720 headless viewport, AutomationControlled flag | `mcp/config.ts:226-274` | high | supports KO-L1-02; enables F1 |
| EK-07 | State | Persistent profile path derives from browser token + sha256(client cwd); default is persistent, not temp | `mcp/browserFactory.ts:193-239`; README:478-488 | high | supports KO-L1-03; enables F6 |
| EK-08 | Lifecycle | Browser factory precedence: remote → CDP → isolated → extension → persistent | `mcp/browserFactory.ts:66-87` | high | enables F2; supports KO-L3-05 |
| EK-09 | Constraint | `isolated` and `userDataDir` cannot be combined; `isolated` stays in memory | `mcp/config.ts:245-246`; README:493-498 | high | constrains EK-07; supports KO-L4-02 |
| EK-10 | Attach | `--extension` attaches to a real Chrome/Edge via a local WebSocket relay and `chrome.debugger` bridge; one extension + one CDP client | `mcp/cdpRelay.ts:230-280`; `mcp/extensionContextFactory.ts:31-51` | high | enables F5; supports KO-L3-05 |
| EK-11 | Transport | No port → stdio; port → HTTP (SSE + Streamable HTTP); default host localhost | `utils/mcp/http.ts:37-55`; `utils/mcp/server.ts:189-211` | high | enables F1 |
| EK-12 | Auth | HTTP has no MCP authentication; Host allowlist is DNS-rebinding protection; `*` disables it | `utils/mcp/http.ts:57-85`; README:681-685 | high | supports KO-L1-04; constrains F5 |
| EK-13 | Exposure | `filteredTools` keeps `core*` always on, adds explicit capabilities, removes `skillOnly` | `backend/tools.ts:83-94` | high | supports KO-L3-01; enables F5 |
| EK-14 | Default surface | Default no-`--caps` tool list is exactly 25 tools | `tests/capabilities.spec.ts:19-48` | high | tested_by EK-27; supports KO-L1-05 |
| EK-15 | RCE | `browser_run_code_unsafe` is core/default; runs code in Node `vm`, exposes Buffer/fetch/etc.; description says RCE-equivalent | `backend/runCode.ts:37-96`; README:1134-1136 | high | supports KO-L2-01; constrains F5 |
| EK-16 | Page JS | `browser_evaluate` is core/default; executes `eval` in page/element context | `backend/evaluate.ts:32-84` | high | supports KO-L2-01 |
| EK-17 | Network | `browser_network_request` is core/default; can return request/response headers and bodies | `backend/network.ts:30-102,140-174` | high | supports KO-L2-02 |
| EK-18 | Modal gate | An open dialog/file chooser blocks all tab tools except `browser_handle_dialog` / `browser_file_upload` | `backend/tool.ts:70-88`; `backend/tab.ts:224-237` | high | supports KO-L3-04; enables F7 |
| EK-19 | Snapshot | `browser_snapshot` uses `page.ariaSnapshot({mode:'ai'})`; refs resolve via `page.getByRef`, else selector/locator | `backend/tab.ts:495-548,592-619`; `backend/snapshot.ts:35-55` | high | supports KO-L3-02; enables F3 |
| EK-20 | Action feedback | Actions default to injecting a fresh snapshot; `snapshot.mode=none` suppresses | `backend/response.ts:161-164`; `mcp/config.ts:100-112` | high | enables F3 |
| EK-21 | Crash recovery | A crashed current tab is closed, current pointer cleared, new tab created and logs `about:blank` | `backend/context.ts:225-238` | high | enables F2 |
| EK-22 | Request memory | Tab keeps request entries; navigation trims earlier entries to 200 | `backend/tab.ts:83-84,273-278` | high | enables F4 |
| EK-23 | Downloads | Downloads auto-save to output file with ordinal naming, bypassing workspace guard via `origin:'code'` | `backend/tab.ts:239-252`; `backend/context.ts:558-571` | high | supports KO-L2-04 |
| EK-24 | File guard | `checkFile` restricts to output/workspace; `origin='code'`, `allowUnrestrictedFileAccess`, `skillMode` bypass | `backend/context.ts:582-598` | high | supports KO-L2-04; constrains F5 |
| EK-25 | Redaction | `secrets` replace literal values in serialized text and text files; URL params can be redacted; not a security feature | `backend/response.ts:123-129,174-235`; README:713-726 | high | supports KO-L2-03 |
| EK-26 | Config exposure | `browser_get_config` is `config` capability, not default; returns `JSON.stringify(context.config)` | `backend/config.ts:20-33`; `backend/tools.ts:83-94` | high | supports KO-L2-05 |
| EK-27 | Tests | 9 tests exercise packaged server only; no upstream `tests/mcp` in this repo | `tests/*.spec.ts`; `AGENTS.md:31-37`; `CONTRIBUTING.md:39-55` | high | tested_by EK-14; limits validation |
| EK-28 | CI | Lint regenerates README and fails on diff; test matrix 3 OS; Docker test + Trivy secret scan | `.github/workflows/ci.yml:1-84` | high | supports EK-04 |
| EK-29 | Container | Docker: Ubuntu noble, non-root `node`, chromium only, `--no-sandbox`, default stdio ENTRYPOINT | `Dockerfile:12-103` | high | supports KO-L2-04 |
| EK-30 | Extension token | `PLAYWRIGHT_MCP_EXTENSION_TOKEN` is passed in the connect URL; relay does not compare it server-side | `mcp/cdpRelay.ts:120,158-193,230-245` | medium | supports C-04; constrains F5 |

## Edge summary

- `supports` edges: 18
- `enables` edges: 9
- `constrains` edges: 5
- `conflicts` edges: 1 (EK-12 vs README Docker claim; see C-03)
- `derived_from` edges: 1 (EK-01 → EK-03)
- `tested_by` edges: 2 (EK-14 → EK-27; EK-04 → EK-28)

## Aggregation clusters (R1–R4)

| Cluster | EKs | Rule | Result |
|---|---|---|---|
| C-A Default-open surface | EK-13, EK-14, EK-15, EK-16, EK-17 | R2 (multi-mechanism, same behavior) | KO-L1-05, KO-L2-01/02, KO-L3-01 |
| C-B Continuity by default | EK-05, EK-06, EK-07, EK-08, EK-09 | R1 (same mechanism family) | KO-L1-03, KO-L4-02 |
| C-C A11y action loop | EK-19, EK-20, EK-14 | R1 | KO-L3-02 |
| C-D Guard convenience | EK-23, EK-24, EK-25, EK-29 | R3 (same semantic role: convenience guard) | KO-L2-03/04, KO-L5-01 |
| C-E Interrupt gate | EK-18, EK-21, EK-27 | R2 | KO-L3-04 |
| C-F Attach/authority | EK-10, EK-11, EK-12, EK-30 | R4 (same flow: authority) | KO-L4-01, C-04 |

## Non-reduction notes

- EK-01 and EK-03 stay separate: "thin wrapper" and "pinned dependency" are different facts; one can change without the other.
- EK-15 and EK-16 remain separate: Node-process code execution and page-context JS have different blast radii.
- EK-25 is not promoted to a security guarantee; the README explicitly calls it convenience.
- EK-30 is medium confidence because server-side validation could exist in the extension code, which was not cloned; the relay itself has no comparison.

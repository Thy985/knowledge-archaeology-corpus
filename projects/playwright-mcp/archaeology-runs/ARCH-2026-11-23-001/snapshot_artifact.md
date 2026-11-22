# Repository Snapshot — playwright-mcp @ v0.0.86

## Snapshot identity

| Field | Value |
|---|---|
| repository | `https://github.com/microsoft/playwright-mcp.git` |
| local path | `/home/user/.doubao/agent_mode/workspace/archaeology-jobs/ARCH-2026-11-23-001/repo` |
| clone command | `git clone --depth 50 https://github.com/microsoft/playwright-mcp.git` |
| HEAD | `2337aff0213d28db41d3e3daf48cee19b3d0bd82` |
| branch | `main` |
| tag at HEAD | `v0.0.86` |
| commit time | `2026-11-18T10:49:18-08:00` (2026-11-19 02:49 +0800) |
| commit subject | `chore: skip fork PRs in the review bot workflow (#1875)` |
| package version | `0.0.86` (`@playwright/mcp`) |
| license | Apache-2.0 |
| repository size | 3.1 MB |
| tracked files | 44 |
| total tracked lines | 6,103 |
| tests | 809 lines / 9 test cases |
| Node engine | `>=20` |
| clone depth | 50 commits |
| read-only | yes; no build or test execution performed |

## Repository role: thin release wrapper, not the MCP implementation

`AGENTS.md:3-20` states the ownership boundary explicitly:

- This repository owns `cli.js`, `index.js`, type declarations, README, Dockerfile, and tests of the packaged server.
- The MCP server and browser tools live in `microsoft/playwright`, under `packages/playwright-core/src/tools/`:
  - `mcp/` — server, config, CLI options
  - `backend/` — tool implementations
- `roll.js` copies `config.d.ts` from upstream; `npm run lint` regenerates README tool/option sections.
- `package.json` pins an alpha of `playwright` / `playwright-core`; the pin is older than upstream `main` tip.

This is the single most important boundary fact. A 44-file reading of this repo alone would miss almost every core mechanism.

**Pinned implementation dependency:**

| Field | Value |
|---|---|
| package | `playwright-core@1.66.0-alpha-2026-11-18` |
| npm `gitHead` | `8124fc39233c4c349b9dbebd4abc4103738eae81` |
| sparse checkout | `/home/user/.doubao/agent_mode/workspace/archaeology-jobs/ARCH-2026-11-23-001/upstream` |
| upstream tool source | `packages/playwright-core/src/tools/{mcp,backend,utils}` |
| upstream tool source size | 93 `.ts` files / 16,568 lines (whole tools tree); `mcp` + `backend` = 8,868 lines |
| compiled artifact used for cross-check | `dep/playwright-core/lib/coreBundle.js` |

Dependency facts below are labelled **[DEP]**; repository facts are labelled **[REPO]**. The package must not present `[DEP]` as this repo's own source.

## Top-level inventory [REPO]

```
cli.js                 # executable entry; delegates to coreBundle tools
index.js               # library entry; exports createConnection
index.d.ts             # public type
config.d.ts            # copy of upstream MCP config type (315 lines)
package.json           # version/deps/scripts/bin
package-lock.json      # 97 locked packages, lockfileVersion 3
README.md              # generated tool/option tables + setup + security note
AGENTS.md              # ownership, upstream fix path, roll, tests, commit rules
CONTRIBUTING.md        # issue-first contribution policy; points to monorepo
SECURITY.md            # MSRC coordinated disclosure boilerplate
Dockerfile             # Ubuntu noble, node user, chromium, tini
server.json            # MCP Registry metadata; stdio only
playwright.config.ts   # chrome project; optional chromium-docker project
roll.js                # bump pinned Playwright + copy config + regenerate README/Docker node
update-readme.js       # generates README tools/options from coreBundle
src/README.md          # placeholder; no TypeScript source here
tests/                 # 809 lines, 9 tests
utils/docker/          # build/publish scripts
.github/workflows/     # CI, publish, agent bot workflows
.azure-pipelines/      # Docker publish pipeline
.claude/skills/        # release runbook
```

## Entrypoints and build [REPO]

`cli.js`:

```js
const { program } = require('playwright-core/lib/utilsBundle');
const { tools, libCli } = require('playwright-core/lib/coreBundle');

if (process.argv.includes('install-browser')) {
  // map install-browser -> playwright install
}
const packageJSON = require('./package.json');
tools.decorateMCPCommand(program, packageJSON.version);
void program.parseAsync(process.argv);
```

`index.js`:

```js
const { tools } = require('playwright-core/lib/coreBundle');
module.exports = { createConnection: tools.createConnection };
```

There is **no build step**: `npm run build` is literally `echo OK` (`package.json:24`). Runtime is the pinned `playwright-core` bundle; the repo ships only wrappers and metadata.

## Core implementation map [DEP]

| Module | Path | Role |
|---|---|---|
| CLI decoration / lifecycle | `mcp/program.ts` | 203 lines; all CLI options; browser factory; shared-browser lifecycle; start server |
| Config merge | `mcp/config.ts` | 679 lines; defaults → config file → env → CLI; path resolution; validation |
| Server / MCP protocol | `utils/mcp/server.ts` | 259 lines; `ListTools`, `CallTool`, lazy backend init, roots, heartbeat |
| HTTP transport | `utils/mcp/http.ts` | 161 lines; SSE + Streamable HTTP; Host allowlist; no auth |
| Browser factory | `mcp/browserFactory.ts` | 283 lines; remote/CDP/isolated/extension/persistent precedence |
| Backend façade | `backend/browserBackend.ts` | 153 lines; one Context per client; `_meta` handling; dispose |
| Tool registry/filter | `backend/tools.ts` | 94 lines; capability filtering; `skillOnly` removal |
| Context | `backend/context.ts` | 598 lines; tab set, isolated contexts, routes, files, redaction |
| Tab | `backend/tab.ts` | 753 lines; event log, downloads, modal state, snapshot, navigation, refs |
| Response | `backend/response.ts` | 481 lines; sections, snapshot injection, redaction, output budget |
| Snapshot | `backend/snapshot.ts` | 237 lines; aria snapshot tool + element schema |
| Run code | `backend/runCode.ts` | 102 lines; `vm` execution, RCE-equivalent |
| Evaluate | `backend/evaluate.ts` | 102 lines; page/element `eval` |
| Network | `backend/network.ts` | 292 lines; request list/details, headers, bodies |
| Files | `backend/files.ts` | 103 lines; upload/drop through workspace guard |
| Extension relay | `mcp/cdpRelay.ts` | 313 lines; WebSocket relay, one extension + one CDP client |
| Browser model | `mcp/browserModel.ts` | 356 lines; `chrome.debugger` ↔ CDP translation |
| Watchdog | `mcp/watchdog.ts` | 38 lines; stdin/SIGINT/SIGTERM → 15s force exit |

## Core data structures / state [DEP]

- `Context`: default browser context; a map of isolated contexts keyed by name; `_tabs`, `_currentTab`, `_routes`, video/recording/tracing state; output file set; pending unhandled rejections.
- `Tab`: page, isolated-context name, request ring (`kMaxEarlierNavigationRequests = 200`), downloads, modal states, recent events, console log, WebMCP listing, timeouts.
- `Response`: result/error/code arrays, snapshot mode/root/depth/boxes, image results, written files, JSON/raw flags.
- `BrowserBackend`: config, tools, one `Context`, session log, idle timer, disconnected flag.
- `Config`: browser launch/context/CDP/extension; server host/port/allowedHosts; capabilities; secrets/redaction; snapshot; network origins; timeouts; output; codegen.

## Request lifecycle [DEP]

1. `cli.js` decorates the command and parses argv.
2. `program.ts` resolves config, filters tools, creates a `BrowserBackend` per client, starts stdio or HTTP.
3. MCP `CallTool` lazily initializes backend, then `BrowserBackend.callTool` parses Zod schema and builds `Response`.
4. `defineTabTool` resolves the current tab and enforces modal exclusivity.
5. Tool handler acts through Playwright; `Response.serialize()` appends page state/snapshot/events, applies output budget and secret redaction.
6. `BrowserBackend` disposes on close, browser disconnect, idle timeout, or a `setClose` tool.

## Configuration surface [DEP]

- Defaults: `codegen=typescript`; action 5s, navigation 60s, expect 5s, settle 500ms; browserName `chromium`; channel `chrome`; Linux without `DISPLAY` → headless; headless viewport 1280×720; headed viewport null.
- Chromium always gets `--disable-blink-features=AutomationControlled` unless user already supplied a matching arg.
- `isolated` and `userDataDir` are mutually exclusive.
- Default profile: `${cacheDir}/ms-playwright-mcp/mcp-${browserToken}-${sha256(client cwd).slice(0,7)}` — persistent and workspace-derived, not a temp directory.
- `createBrowser` precedence: `remoteEndpoint` → `cdpEndpoint` → `isolated` → `extension` → persistent.
- Idle timeout: only self-owned headless browsers get 1 hour by default; headed and attached browsers get none.
- `--extension` is attached mode: it spawns/opens Chrome, starts a local WebSocket relay, and bridges `chrome.debugger` to CDP.

## Permission / governance mechanisms [REPO + DEP]

- `SECURITY.md` is MSRC boilerplate; it is not a project threat model.
- README `Security`: "Playwright MCP is **not** a security boundary."
- `allowedOrigins` / `blockedOrigins`: request interception, documented as not a security boundary and not effective for redirects.
- `secrets`: response text replacement; documented as convenience, not security.
- `allowUnrestrictedFileAccess`: file guard convenience, not a secure boundary.
- No MCP-layer authentication in the HTTP transport; only a `Host` header allowlist for DNS-rebinding protection.
- Default tool set includes `browser_run_code_unsafe` (description: "RCE-equivalent") and `browser_evaluate`.

## Test system [REPO]

- `tests/fixtures.ts`: stdio MCP client spawns `cli.js`, optional roots, config file, Docker mode; `toHaveResponse()` parses sections.
- 9 tests: default tool list (exact set of 25), pdf capability, vision capability, legacy `--vision`, `install-browser --help`, README env-var/runtime consistency, navigate snapshot, click by ref, CommonJS library import.
- CI: lint (README regeneration + `git diff --exit-code`), test matrix Ubuntu/macOS/Windows, Docker test project + Trivy secret scan.
- No test execution in this run: `npm ci` and `npm test` were not run; no pass rate/coverage claim is made.

## External dependencies / supply chain [REPO]

- Runtime: `playwright` and `playwright-core` pinned to the same alpha.
- Dev: `@modelcontextprotocol/sdk ^1.25.2`, `@playwright/test` pinned to the same alpha, `@types/node ^24.3.0`.
- `package-lock.json`: 97 packages, lockfileVersion 3.
- Docker: Ubuntu noble, Node 26.13.1 (updated by `roll.js`), runs as non-root `node`, chromium only, `--no-sandbox`, no port in default ENTRYPOINT.
- npm publish uses OIDC and `npm run ctest`; release also publishes to the MCP Registry and validates `server.json` version alignment.

## Snapshot limitations

- Shallow clone, depth 50: history-dependent claims are out of scope.
- The implementation dependency is identified by npm `gitHead`, not by a fresh build from source.
- No browser binary / Chrome channel was validated locally; no runtime test was executed.
- The package deliberately separates `[REPO]` from `[DEP]` facts.

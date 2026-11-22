# 01 · Project Layer — playwright-mcp

## 1. Project boundary

`microsoft/playwright-mcp` is a **release wrapper**, not the MCP implementation repository.

Evidence:

- `AGENTS.md:3-20`: "This repo ships the npm package `@playwright/mcp`. The MCP server itself is in `microsoft/playwright`, in `packages/playwright-core/src/tools/`: `mcp/` has the server, its config and its command line options, and `backend/` has the tools."
- `src/README.md` is a placeholder; there is no TypeScript source under `src/`.
- `index.js:17` and `cli.js:17-37` import `playwright-core/lib/coreBundle`.
- `package.json:24`: `"build": "echo OK"`.

The archaeology therefore uses two evidence planes:

| Plane | Path | What it can prove |
|---|---|---|
| Repository plane `[REPO]` | `repo/` @ `2337aff` | release wrapper, public surface, tests, Docker, CI, docs |
| Implementation plane `[DEP]` | upstream `8124fc3`, `packages/playwright-core/src/tools/` | server, backend, tools, browser lifecycle |
| Compiled cross-check `[DEP]` | `dep/playwright-core/lib/coreBundle.js` | confirms the published bundle contains the same implementation sections |

## 2. Inventory

44 tracked files, 6,103 lines, 3.1 MB. Top-level groups:

| Group | Files | Role |
|---|---|---|
| Entrypoints | `cli.js`, `index.js`, `index.d.ts` | executable and library entry |
| Types/config | `config.d.ts`, `package.json`, `package-lock.json` | public config type, package metadata, 97 locked packages |
| Documentation | `README.md`, `AGENTS.md`, `CONTRIBUTING.md`, `SECURITY.md` | user docs, ownership contract, contribution policy, MSRC policy |
| Packaging | `Dockerfile`, `server.json`, `.npmignore` | container, MCP Registry metadata |
| Tests | `tests/*.spec.ts`, `tests/fixtures.ts`, `tests/testserver/` | 9 packaged-server tests |
| Build/release | `roll.js`, `update-readme.js`, `.github/workflows/*`, `.azure-pipelines/*`, `.claude/skills/release.md` | version roll, README generation, CI/publish |
| Utilities | `utils/docker/*` | Docker build/publish scripts |

## 3. Entrypoints

`cli.js`:

- Maps `install-browser` to Playwright's `install`.
- Decorates a Commander program with `tools.decorateMCPCommand`.
- Parses argv; actual server startup is upstream.

`index.js`:

- Exports only `createConnection` from `playwright-core/lib/coreBundle`.

`package.json`:

- `bin.playwright-mcp = cli.js`
- `exports["."] = { types: index.d.ts, default: index.js }`
- scripts: `lint` (regenerate README), `test` (`playwright test`), `ctest` (chrome only), `build` (`echo OK`), `roll`.

## 4. Core modules (implementation plane)

| Module | Approx lines | Responsibility | Key symbols |
|---|---:|---|---|
| `mcp/program.ts` | 203 | CLI options, config resolution, shared browser, server start | `decorateMCPCommand`, `factory.create` |
| `mcp/config.ts` | 679 | defaults, env/CLI/file merge, path resolution, browser validation | `resolveCLIConfigForMCP`, `validateBrowserConfig` |
| `mcp/browserFactory.ts` | 283 | remote/CDP/isolated/extension/persistent browser creation | `createBrowser`, `createPersistentBrowser`, `createIdleTimer` |
| `utils/mcp/server.ts` | 259 | MCP server, tool list, call dispatch, roots | `createServer`, `initializeServer`, `allRootPaths` |
| `utils/mcp/http.ts` | 161 | SSE and Streamable HTTP, host allowlist | `startMcpHttpServer`, `installHttpTransport` |
| `backend/browserBackend.ts` | 153 | per-client backend, lazy init, tool dispatch, dispose | `BrowserBackend.callTool`, `initialize`, `dispose` |
| `backend/tools.ts` | 94 | tool registry and capability filter | `browserTools`, `filteredTools` |
| `backend/context.ts` | 598 | tab/context set, routes, file guards, redaction | `Context.ensureTab`, `checkUrlAllowed`, `checkFile` |
| `backend/tab.ts` | 753 | event log, requests, downloads, modal state, snapshots, refs | `Tab.targetLocators`, `captureSnapshot` |
| `backend/response.ts` | 481 | section building, snapshot injection, output budget, redaction | `Response.serialize`, `_build` |
| `backend/runCode.ts` | 102 | arbitrary code in Node `vm` | `runCode` |
| `backend/evaluate.ts` | 102 | arbitrary page-context JS | `evaluate` |
| `backend/network.ts` | 292 | request list/details, headers, bodies | `requests`, `request` |
| `backend/files.ts` | 103 | upload/drop through workspace guard | `uploadFile`, `drop` |
| `mcp/cdpRelay.ts` | 313 | local WebSocket relay for extension attach | `CDPRelayServer` |
| `mcp/browserModel.ts` | 356 | `chrome.debugger` ↔ CDP translation | `BrowserModel` |
| `mcp/watchdog.ts` | 38 | exit watchdog | `setupExitWatchdog` |

## 5. Core data structures and state

**`Context`** (`backend/context.ts:112-539`):

- `_defaultBrowserContext`: the context supplied by the browser factory.
- `_browserContexts: Map<string|undefined, Promise<BrowserContext>>`: default and named isolated contexts.
- `_tabs: Tab[]`, `_currentTab`: single active tab pointer; `ensureTab()` creates one if absent.
- `_routes: RouteEntry[]`: user route mocks, re-applied to each new context.
- `_video`, `_recording`, `_tracing`: optional artifact lifecycles.
- `_outputFiles`: ordinal collision tracking.
- `_pendingUnhandledRejections` and listeners: surfaces async listener errors into tool responses.
- `redactSecrets()`: literal secret replacement + URL-param regex.

**`Tab`** (`backend/tab.ts:106-624`):

- `page`, `isolatedContext`, `crashed`.
- `_requests: RequestEntry[]`, `_lastRequestIndex`, `_navigationRequestIndex`, `_mainDocumentStatus`.
- `_downloads`, `_modalStates`, `_recentEventEntries`, `_consoleLog`, `_webmcpTools`.
- timeouts: `action`, `navigation`, `expect`.
- `targetLocators()`: ref/selector/locator resolution.

**`Response`** (`backend/response.ts:52-235`):

- `_results`, `_errors`, `_code`, `_imageResults`.
- snapshot flags: `none|full|explicit`, filename, root, depth, boxes.
- `_clientWorkspace`, `_json`, `_raw`.
- `_writeFile` redacts text; `serialize` redacts all serialized text.

**`BrowserBackend`** (`backend/browserBackend.ts:38-147`):

- one `Context` per client; `_tools` is the filtered list; `_sessionLog` optional.
- `_disconnected` / `_disposed`; `callTool` builds a `Response` and normalizes errors.

## 6. Lifecycle

1. **Process start** (`cli.js`): parse argv, call upstream `decorateMCPCommand`.
2. **Config resolution** (`mcp/config.ts:137-154`): defaults → file → env → CLI; validate browser; set headless default; validate output dir.
3. **Server start** (`utils/mcp/server.ts:189-211`): no port → `StdioServerTransport`; port → HTTP server + host allowlist.
4. **Lazy backend init** (`server.ts:85-114`): first `CallTool` triggers `initializeServer`; roots are read from client capabilities, default cwd.
5. **Browser creation** (`browserFactory.ts:66-96`): choose remote/CDP/isolated/extension/persistent; bind endpoint; create idle timer.
6. **Call** (`browserBackend.ts:89-145`): parse Zod, build `Response`, run tool, drain async errors, serialize, log.
7. **Dispose** (`browserBackend.ts:81-87`): dispose context + callback; triggered by close, browser disconnect, idle timeout, or a close tool.
8. **Exit** (`watchdog.ts:21-37`): stdin close/SIGINT/SIGTERM → graceful close; 15s force exit.

## 7. Configuration and defaults

- **Precedence**: `defaultConfig` → config file (`--config` / `PLAYWRIGHT_MCP_CONFIG`) → env overrides → CLI overrides (`config.ts:137-145`).
- **Defaults**: codegen typescript; action 5000ms; navigation 60000ms; expect 5000ms; settle 500ms.
- **Browser**: `browserName` default `chromium`; if no channel, channel `chrome`; Linux no `DISPLAY` → headless; headless viewport 1280×720; headed viewport null; Chromium gets `--disable-blink-features=AutomationControlled`.
- **Mutual exclusion**: `isolated` + `userDataDir` throws.
- **Profile**: `createUserDataDir` uses `defaultCacheDirectory()/ms-playwright-mcp`, directory `mcp-${browserToken}-${sha256(clientInfo.cwd).slice(0,7)}`.
- **Idle**: `defaultIdleTimeout = 3,600,000ms`; only `ownership === 'own' && headless` by default.
- **Extension**: `channel` default chrome; requires installed Playwright Extension; uses `CDPRelayServer`.

## 8. Permission / governance mechanisms

| Mechanism | Implementation | Claim strength |
|---|---|---|
| Security policy | `SECURITY.md` MSRC reporting boilerplate | no project threat model |
| Product security stance | README: "not a security boundary" | explicit |
| HTTP host allowlist | `utils/mcp/http.ts:57-85` | DNS-rebinding protection only |
| MCP auth | none found | **no authentication layer** |
| Network origins | `Context._setupRequestInterception` | request-layer, not redirect/navigation policy |
| File roots | `checkFile` inside output/workspace | convenience; `origin='code'` bypass |
| Secrets | `redactSecrets` on serialized text | convenience, not isolation |
| Capabilities | `filteredTools` | exposure control, not authorization |
| Modal gate | `defineTabTool` | page-interrupt safety, not user auth |

## 9. Test system

- 9 tests in 5 spec files:
  - exact default tool set (25)
  - pdf/vision/legacy vision capabilities
  - `install-browser --help`
  - README env-var/runtime consistency
  - navigate snapshot
  - click by ref
  - CommonJS library import
- `tests/fixtures.ts`: stdio client, optional roots/config, Docker mode, section parser.
- CI: lint + `git diff --exit-code`; test matrix Ubuntu/macOS/Windows; Docker test + Trivy secret scan.
- Tests exercise the **packaged server**, not upstream `tests/mcp` (`AGENTS.md`, `CONTRIBUTING.md`).
- No `npm ci` / `npm test` was run; no pass rate or coverage is claimed.

## 10. Extension points / upgrade path

- **Fix location**: upstream `microsoft/playwright` (`packages/playwright-core/src/tools`), not this repo.
- **Roll**: `node roll.js` bumps `playwright`, `playwright-core`, `@playwright/test`, copies `config.d.ts`, regenerates README, updates Docker `NODE_VERSION`.
- **Release**: npm OIDC; `npm run ctest` must pass; MCP Registry `server.json` versions must match.
- **Test contract**: add cases to closest spec; run on every project.

## 11. Project-level risks visible from structure

1. **Double-layer evidence risk**: most implementation claims cannot be sourced to this repo; the dependency pin is the real source of truth.
2. **Generated documentation risk**: README tool/option tables are generated; they can drift from code only if `npm run lint` is skipped. CI checks drift.
3. **Single-maintainer platform risk**: all implementation is in a monorepo owned by Microsoft; this repo cannot fix backend bugs directly.
4. **Distribution mismatch risk**: npm package, MCP Registry metadata and Docker image are three artifacts with separate version checks.

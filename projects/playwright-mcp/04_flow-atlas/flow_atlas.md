# 04 · Flow Atlas — seven flows

Every edge names the symbol/file and the condition or state transition. Edges marked **[BYPASS]** are alternate/override paths found during the audit; they are not hypothetical.

## F1 · Control Flow

Purpose: turn CLI/stdin input into a browser action and a tool response.

| Edge | From → To | Symbol / file | Condition |
|---|---|---|---|
| F1-E1 | process argv → Commander program | `cli.js:17-37` | always |
| F1-E2 | program options → resolved config | `mcp/program.ts:34-202`; `mcp/config.ts:137-154` | CLI wins over env/file/defaults |
| F1-E3 | resolved config → browser | `mcp/browserFactory.ts:66-87` | first matching branch |
| F1-E4 | resolved config → filtered tool list | `backend/tools.ts:83-94` | `core*` or explicit capability; not `skillOnly` |
| F1-E5 | MCP `CallTool` → lazy backend init | `utils/mcp/server.ts:85-114` | first call only |
| F1-E6 | backend → tool handler | `backend/browserBackend.ts:89-145` | Zod parse passes |
| F1-E7 | tool handler → current tab | `backend/tool.ts:70-88`; `backend/context.ts:225-238` | modal gate passes; tab exists or is created |
| F1-E8 | handler → serialized response | `backend/response.ts:174-235` | always |
| F1-E9 | close/disconnect/idle → dispose | `backend/browserBackend.ts:81-87`; `browserFactory.ts:61-64` | close tool, browser disconnect, idle timeout |

**Control invariants.**

- No tool can run without `ensureTab()` except non-tab/config tools.
- Every `CallTool` error is converted to `isError:true` text; the process does not crash (`server.ts:105-114`).
- `browser_close` sets `response.isClose`; backend disposes at end of call.

## F2 · State Flow

Purpose: track browser, isolated contexts, tabs, and the current tab.

| Edge | From → To | Symbol / file | Condition / transition |
|---|---|---|---|
| F2-S1 | backend init → default Context | `browserBackend.ts:55-64` | once per client |
| F2-S2 | ensureTab → new tab | `context.ts:225-238` | no current tab or current crashed |
| F2-S3 | browser `page` event → Tab object | `context.ts:378-414` | new page; first page becomes current |
| F2-S4 | page close → tab removal | `context.ts:395-407` | current close selects neighbor |
| F2-S5 | crashed page → close/reset | `context.ts:225-238` | `tab.crashed`; log `about:blank` |
| F2-S6 | selectTab → bringToFront | `context.ts:208-223` | explicit tab selection |
| F2-S7 | closeTab → page close + URL return | `context.ts:240-247` | explicit close |
| F2-S8 | dispose → close isolated contexts + tabs | `context.ts:416-434` | client close / browser disconnect |

**State invariants.**

- `_currentTab` is a single pointer; `selectTab` is the only way to change it except page creation/closure.
- Isolated contexts are cached by name; default context key is `undefined`.
- Crash recovery is automatic and destructive: the crashed page is closed, not reused.

**[BYPASS] state paths.**

- `isolatedContext` name on `browser_tabs` creates a separate context; cookies are not shared across names.
- `--isolated` discards the persistent profile at close.
- `--extension`/`--cdp-endpoint`/`--remote-endpoint` attach to an existing browser; close semantics differ from owned browser.

## F3 · Data Flow

Purpose: move page structure into the model-readable response, and actions back into the page.

| Edge | From → To | Symbol / file | Condition |
|---|---|---|---|
| F3-D1 | live page → aria snapshot | `backend/tab.ts:495-548` | snapshot requested or action includes snapshot |
| F3-D2 | aria snapshot → text/JSON sections | `backend/snapshot.ts:35-55`; `response.ts:174-235` | `snapshot.mode != none` |
| F3-D3 | model `target` → ref/selector/locator | `backend/tab.ts:592-619` | `eN`/`fN eN` ref, else parse locator, else CSS |
| F3-D4 | action → fresh snapshot | `backend/response.ts:161-164` | tool calls `setIncludeSnapshot()` |
| F3-D5 | download → output file | `backend/tab.ts:239-252` | `download` event |
| F3-D6 | screenshot → image result | `backend/screenshot.ts` (tool), `response.ts:200-235` | screenshot tool; imageResponses policy |
| F3-D7 | file upload/drop → page | `backend/files.ts:22-61` | path or data; guard applies |

**Data invariants.**

- Ref resolution is **live**, not a stored DOM handle; stale refs error and demand a new snapshot.
- `boxes` are viewport-relative CSS pixels from `getBoundingClientRect`; not page coordinates.
- Download naming uses `ordinal:true` to avoid collisions.

**[BYPASS] data paths.**

- `--caps=vision` adds coordinate mouse tools; pixels can replace refs.
- `browser_take_screenshot` is default and returns an image.
- `browser_evaluate` can read arbitrary page data directly, bypassing the snapshot.

## F4 · Evidence Flow

Purpose: preserve what happened for the model and for later debugging.

| Edge | From → To | Symbol / file | Condition |
|---|---|---|---|
| F4-V1 | console/pageerror → `_consoleLog` | `backend/tab.ts:129-167` | listener active |
| F4-V2 | request/response/requestfailed → `_requests` | `backend/tab.ts:269-301` | listener active |
| F4-V3 | main-frame navigation response → `_mainDocumentStatus` | `backend/tab.ts:280-289` | non-redirect, main frame |
| F4-V4 | recent events/console → response sections | `response.ts:174-235` | `renderTabMarkdown` |
| F4-V5 | tool call/result → session log | `browserBackend.ts:89-145` | `saveSession` / session log enabled |
| F4-V6 | output files → disk with budget | `response.ts:200-235`; `context.ts:558-580` | file result / output size |

**Evidence invariants.**

- Request history before a navigation is trimmed to 200 entries.
- `toHaveResponse()` parses sections; tests rely on stable section names.
- Session log is optional; absence does not affect tool behavior.

**[LIMIT] evidence gaps.**

- No runtime test was executed; no trace/video artifact was produced.
- `browser_network_request` exposes headers/body only when the tool is called; it is not an automatic audit log.
- Console/error logs are per-tab and cleared on tab closure.

## F5 · Authority Flow

Purpose: decide who can do what, and where the trust boundary actually sits.

| Edge | From → To | Symbol / file | Condition |
|---|---|---|---|
| F5-A1 | MCP client roots → `clientInfo.cwd` | `utils/mcp/server.ts:128-148,214-237` | roots capability and not `ignoreRoots` |
| F5-A2 | HTTP `Host` header → allow/deny | `utils/mcp/http.ts:57-85` | not `*`; 403 if absent from list |
| F5-A3 | extension connect URL → relay | `mcp/cdpRelay.ts:120,158-193` | token optional; relay does not compare it |
| F5-A4 | resolved capabilities → `ListTools` | `backend/tools.ts:83-94` | filter before exposure |
| F5-A5 | file path → `checkFile` | `backend/context.ts:582-598` | origin != code; not unrestricted; not skillMode |
| F5-A6 | URL/origin → request interception | `backend/context.ts:436-449` | allowed/blocked origins configured |
| F5-A7 | response text → `redactSecrets` | `response.ts:123-129` | secret literal or URL param match |
| F5-A8 | launch flags → browser sandbox | `mcp/config.ts:226-274`; Dockerfile | channel/platform defaults; `--no-sandbox` override |

**Authority invariants.**

- There is **no MCP authentication** on HTTP; a Host check is not identity.
- `allowedHosts='*'` disables the only HTTP host control.
- `--caps` is an exposure filter, not an authorization check; a client that can call a tool can use it fully.
- The extension relay accepts a connection if the path is correct; the token is not verified at the relay.

**[BYPASS] authority paths.**

| Path | Effect | Evidence |
|---|---|---|
| `origin='code'` | skips workspace/output file check | `context.ts:582-598` |
| `allowUnrestrictedFileAccess` | skips file roots | same |
| `skillMode` | skips file roots | same |
| `--allowed-hosts '*'` | skips Host check | `http.ts:64-70` |
| `--no-sandbox` | disables OS sandbox | README:438; Dockerfile |
| `browser_run_code_unsafe` | arbitrary Node code | `runCode.ts` |
| `browser_evaluate` | arbitrary page JS | `evaluate.ts` |
| `browser_get_config` | full config dump when enabled | `config.ts:20-33` |

## F6 · Memory Flow

Purpose: preserve identity, session and history across calls and process restarts.

| Edge | From → To | Symbol / file | Condition |
|---|---|---|---|
| F6-M1 | cache dir + browser token + cwd hash → profile dir | `browserFactory.ts:193-239` | persistent mode |
| F6-M2 | profile dir → `launchPersistentContext` | `browserFactory.ts:193-244` | default branch |
| F6-M3 | storage state file → context | `config.ts:302-426`; `browserFactory.ts` | `--storage-state` / contextOptions |
| F6-M4 | page cookies/localStorage → storage tools | `cookies.ts`; `storage.ts` | `storage` capability opt-in |
| F6-M5 | session/trace/video → output | `browserBackend.ts`; `context.ts` | saveSession / tracing / video |
| F6-M6 | idle timer → dispose | `browserFactory.ts:61-64`; `idleTimer.ts` | own + headless + 1h default |

**Memory invariants.**

- Persistent profile is per workspace, not per session; concurrent clients with the same workspace conflict.
- `isolated` keeps profile in memory and loses state on close.
- `userDataDir` is an explicit persistent override; cannot combine with `isolated`.
- Cookies are domain-scoped by the browser; the MCP server does not re-scope them.

**[BYPASS] memory paths.**

- `--extension` uses the user's real Chrome profile, including existing logins.
- `--cdp-endpoint` / `--remote-endpoint` attach to a browser whose profile is outside this process.
- `browser_storage_state` can export cookies/localStorage to a file when `storage` capability is enabled.

## F7 · Policy Flow

Purpose: apply limits and safety policies to tool behavior.

| Edge | From → To | Symbol / file | Condition |
|---|---|---|---|
| F7-P1 | dialog/filechooser → block non-clearing tools | `backend/tool.ts:70-88` | modal state present |
| F7-P2 | `snapshot.mode` → include/suppress snapshot | `config.ts:100-112`; `response.ts:161-164` | full/none |
| F7-P3 | `imageResponses` → allow/omit/only images | `config.ts`; `response.ts` | image result |
| F7-P4 | `outputMaxSize` → evict old files | `context.ts`; `response.ts` | output write |
| F7-P5 | `allowedOrigins`/`blockedOrigins` → route abort/continue | `context.ts:436-449` | request interception |
| F7-P6 | `allowUnrestrictedFileAccess` → guard bypass | `context.ts:582-598` | true |
| F7-P7 | timeouts → action/navigation/expect/settle/idle | `config.ts:100-112`; `idleTimer.ts` | per call / default |
| F7-P8 | `secrets`/`redactUrlParams` → response redaction | `response.ts:123-129` | literal/param match |

**Policy invariants.**

- Modal gate is a hard precondition, not a hint.
- Network origin policy is request-layer, and the README says it does not affect redirects.
- Redaction is post-serialization and best-effort; binary output is not covered.
- `allowUnrestrictedFileAccess` is documented as convenience, not a boundary.

**[CONFLICT] policy vs doc.**

- README Docker says the server "can be reached by any MCP client", but the HTTP Host allowlist defaults to localhost/loopback/bind host. An external client using an IP `Host` gets 403 unless `--allowed-hosts` is widened. See C-03.

## Flow-level summary

| Flow | Edges | Strongest evidence | Weakest edge |
|---|---:|---|---|
| Control | 9 | `backend/browserBackend.ts`, `server.ts` | config precedence intent |
| State | 8 | `context.ts` crash/tab lifecycle | extension close semantics |
| Data | 7 | `tab.ts` snapshot/ref | screenshot image policy |
| Evidence | 6 | `tab.ts` request/console | no runtime trace |
| Authority | 8 | `http.ts`, `tools.ts`, `context.ts` | extension token validation |
| Memory | 6 | `browserFactory.ts` profile | storage tools untested |
| Policy | 8 | `tool.ts` modal, `context.ts` routes | allowed vs blocked precedence |

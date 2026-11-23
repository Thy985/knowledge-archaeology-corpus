# 01 · Project Layer — WebMCP repository map

> This layer answers “what the repository is and how it is organized”, not “what it teaches”.
> All facts are repository-local unless marked `[EXTERNAL]`.

## 1. Repository identity

| Field | Value | Source |
|---|---|---|
| Name | WebMCP | `index.bs:2` |
| Status | CG-DRAFT | `index.bs:3` |
| Group | W3C Web Machine Learning Community Group (`webml`) | `index.bs:4`, `w3c.json` |
| Editors | Brandon Walderman (Microsoft), Khushal Sagar (Google), Dominic Farolino (Google), Samuel Dallstream (Microsoft) | `index.bs:6–9` |
| License | W3C Software and Document License | `LICENSE.md:1–2` |
| Repository | `webmachinelearning/webmcp` | `index.bs:5` |
| Normative source | `index.bs` (Bikeshed) | `Makefile:1–13` |
| Published output | `index.html` on `gh-pages` | `.github/workflows/auto-publish.yml:1–16` |
| Test suite | `https://wpt.fyi/results/webmcp` (external) | `index.bs:17` |

## 2. Repository shape

```
webmcp/
├── index.bs                     # normative spec, 2,355 lines
├── README.md                    # explainer, use cases, design, best practices
├── continuations-explainer.md   # non-normative token proposal
├── declarative-api-explainer.md # non-normative form API proposal
├── docs/service-workers.md      # non-normative background worker proposal
├── implementation-status.md     # browser/agent support table
├── security-privacy-questionnaire.md
├── CONTRIBUTING.md
├── LICENSE.md
├── Makefile
├── w3c.json
├── .pr-preview.json
├── .github/workflows/auto-publish.yml
├── .github/dependabot.yml
├── .gitignore
└── assets/                      # logos for implementation-status
```

There is **no `src/`, no `package.json`, no test runner, no browser extension, and no server**. The repository does not implement WebMCP. It specifies and explains it.

## 3. Module map

### 3.1 Normative specification (`index.bs`)

| Module | Lines | Responsibility |
|---|---:|---|
| Metadata / bibliography | 1–158 | Spec identity, editors, references |
| Introduction / terminology | 160–176 | Agent, browser agent, AI platform |
| Supporting concepts | 176–323 | Model context, tool definition, annotations, error taxonomy |
| Pending executions | 323–470 | Browser-process ledger, cancel, unload cleanup, toolchange |
| API | 711–1420 | `Document.modelContext`, registration, discovery, execution, dictionaries |
| Tool interface | 1420–1580 | Snapshot object, weak source, origin, annotations |
| Events | 1580–1647 | `toolchange`, `toolactivated`, `toolcancel` |
| Permissions policy | 1647–1655 | `tools` feature |
| Agent interaction | 1653–1777 | Event loop, AI agent queue, observation |
| Security/privacy | 1777–2333 | Threat model, cross-origin gates, mitigations |
| Accessibility | 2333–2336 | Empty placeholder |

### 3.2 Explainer layer

| Document | Lines | What it adds beyond the spec |
|---|---:|---|
| `README.md` | 548 | Motivation, use cases, imperative API walkthrough, declarative future, iframe model, best practices, alternatives, open questions |
| `continuations-explainer.md` | 312 | Cross-document and cross-frame tool continuation tokens; not in `index.bs` |
| `declarative-api-explainer.md` | 213 | Form-based tool declaration, schema synthesis, `SubmitEvent.respondWith()`; not in `index.bs` |
| `docs/service-workers.md` | 274 | Background service-worker tools, JIT install, routing, session IDs, lethal trifecta; not in `index.bs` |

The explainers are **design intent and future work**, not normative behavior. Any claim from them is tagged `[DOC]` in this package, never `[SPEC]`.

### 3.3 Governance and build layer

| File | Responsibility |
|---|---|
| `.github/workflows/auto-publish.yml` | `w3c/spec-prod@v2`; build with `BUILD_FAIL_ON: warning`; deploy `gh-pages` |
| `Makefile` | `make` uses csswg API or local `bikeshed`; `make lint` runs `bikeshed --dry-run --die-when=late`; `make watch` |
| `.pr-preview.json` | Bikeshed PR preview, `force: 1` |
| `w3c.json` | CG report metadata, group `110166` |
| `.github/dependabot.yml` | Weekly grouped GitHub Actions updates |
| `CONTRIBUTING.md` | W3C CG CLA; substantive contributions require CG membership |

## 4. Core abstractions

### 4.1 `ModelContext`

`Document.modelContext` is a `[SecureContext, SameObject]` attribute (`index.bs:739–746`). Each `Document` gets a `ModelContext` with:

- a **tool map** keyed by unique internal `tool id`;
- a **local pending tool executions map** keyed by `uuid`, holding event-loop-local `AbortController`s (`index.bs:176–260`).

The tool map is document-scoped. There is no global registry and no cross-document name namespace.

### 4.2 `tool definition`

A `tool definition` contains:

- `tool id` — unique per registration, not reused on re-registration;
- `name` — 1–128 ASCII alphanumeric / `_` / `-` / `.`;
- `title` — `USVString` or null;
- `description` — non-empty string;
- `input schema` / `output schema` — serialized JSON string or empty string;
- `execute steps` — algorithm invoking the page callback;
- `annotations` — `readOnlyHint`, `untrustedContentHint`, `consequentialHint`, `debugging`;
- `exposed origins` — list of origins (`index.bs:190–250`).

### 4.3 `Tool`

`Tool` is a **snapshot** returned by `getTools()`. It exposes `name`, `title`, `description`, `inputSchema`, `outputSchema`, `source` (weak `Window`), `origin`, `annotations` (`index.bs:1420–1470`). Each `getTools()` call creates new `Tool` objects; two calls never return `===` objects (`index.bs:1427`).

### 4.4 `pending tool execution`

A `pending tool execution` holds `caller document`, `target document`, `tool name`, and `completion steps` (`index.bs:323–350`). It lives in the traversable navigable's map, which is mutated **in parallel** to simulate a browser-process ledger (`index.bs:340–350`).

## 5. Entry points and control surface

### 5.1 Page author

```js
await document.modelContext.registerTool({
  name: "add-todo",
  description: "Add a new item to the user's active todo list",
  inputSchema: { type: "object", properties: { text: { type: "string" } }, required: ["text"] },
  async execute({ text }) { ... }
}, { signal: controller.signal });
```

Registration is asynchronous but the tool enters the map before the returned promise resolves (`index.bs:811–950`).

### 5.2 In-page agent

```js
const tools = await document.modelContext.getTools();
const tool = tools.find(t => t.name === "add-todo");
const result = await document.modelContext.executeTool(tool, { text: "Buy groceries" });
```

`getTools()` enumerates the caller document's **inclusive descendant navigables** (`index.bs:995`). `executeTool()` requires the target to be in the **same traversable navigable** (`index.bs:1150–1160`).

### 5.3 Browser-integrated agent

The browser agent does not call `getTools()`. It uses an implementation-defined `observation` over a top-level traversable (`index.bs:1682–1760`). The observation contains a tool map keyed by `Document` unique ID, with origin and tool definitions. The spec explicitly does not prescribe the format exposed to the browser agent (`index.bs:1760`).

## 6. Lifecycle

### 6.1 Document lifecycle

```
Document created
  → associated ModelContext created
  → fully active
  → register/get/execute allowed (if tools policy allows)
  → inactive (BFCache or navigation)
  → tools unavailable; pending tasks may wait
  → restored / destroyed
```

- `registerTool` rejects if the document is not fully active (`index.bs:811–812`).
- `getTools` rejects if not fully active (`index.bs:959–962`).
- `executeTool` rejects if caller not fully active (`index.bs:1096–1100`).
- BFCache: tools remain in memory but are unavailable; on restoration they become available again (`security-privacy-questionnaire.md:91–92`).
- Questionnaire says pending invocations are abandoned on disconnect, but notes this is “not yet spec'd” (`security-privacy-questionnaire.md:100`).

### 6.2 Tool lifecycle

```
unregistered
  → registerTool() validates
  → registered in tool map
  → discovered by getTools() → Tool snapshot
  → executeTool() → pending execution
  → invoke execute callback
  → fulfilled/rejected → completion steps
  → Tool may be unregistered by AbortSignal
```

There is no `updateTool()` and no separate `unregisterTool()`. Unregistration is the abort steps of the registration `AbortSignal` (`index.bs:650–680`). Changing metadata requires re-registration; changing implementation behind a stable callback does not (`README.md:438–444`).

### 6.3 Execution lifecycle

```
executeTool()
  → validate caller, origin, tool id, exposure
  → create pending execution {caller, target, name, completion}
  → queue tool execute steps on target event loop
  → parse JSON input to object
  → create AbortController in target realm
  → invoke page execute callback
  → toolactivated event
  → await promise
  → serialize result / map error
  → completion removes pending entry, resolves/rejects caller promise
```

Cancellation path:

```
caller AbortSignal abort
  → reject caller promise with abort reason
  → in parallel cancel pending execution
  → remove traversable entry
  → queue target global task
  → remove local entry
  → abort target AbortController
  → fire toolcancel at target ModelContext
```

Target unload path: completion with `target document unloaded` → caller gets `AbortError` (`index.bs:410–430`, `:1195–1240`).

## 7. Configuration surface

### 7.1 Permissions Policy

`tools` is a policy-controlled feature with default allowlist `self` (`index.bs:1647–1655`). It gates `registerTool`, `getTools`, and `executeTool`. Cross-origin iframe needs `<iframe allow="tools">`; a site can disable with `Permissions-Policy: tools=()` (`README.md:323–333`, `index.bs:2268–2283`).

### 7.2 Cross-origin exposure

| Knob | Side | Default | Effect |
|---|---|---|---|
| `tools` policy | embedder | `self` | allows API in document |
| `exposedTo` | tool provider | empty | lists origins allowed to see/invoke |
| `fromOrigins` | tool consumer | empty | lists origins whose tools are queried |

All listed origins must be potentially trustworthy; otherwise `SecurityError` (`index.bs:872–885`, `:974–985`). `executeTool` rechecks exposure (`index.bs:1185–1192`).

### 7.3 AbortSignals

- Registration `signal`: unregisters tool when aborted (`index.bs:894–899`).
- Execution `signal`: rejects caller promise and cancels pending execution (`index.bs:1140–1150`).
- Callback `options.signal`: aborts when execution is cancelled (`index.bs:580–590`).

### 7.4 Not configurable

- No authentication.
- No per-tool user consent.
- No rate limit.
- No tool count limit.
- No maximum description/title length.
- No native input/output schema validation.
- No audit log.

These are absences, not bugs; they are the boundary the spec draws.

## 8. Permissions and governance mechanisms

| Mechanism | Layer | Enforcement |
|---|---|---|
| `SecureContext` | API exposure | browser |
| `tools` Permissions Policy | document/frame | browser |
| `exposedTo` | tool provider | browser |
| `fromOrigins` | tool consumer | browser |
| opaque origin rejection | execution | browser |
| `AbortSignal` | registration/execution | browser + page |
| `Tool` id lookup | TOCTOU | browser |
| platform-defined error mapping | cross-origin leak | browser |
| annotations | agent policy | **not enforced** |
| `consequentialHint` | user confirmation | **not enforced** |
| schema validation | input/output | **not implemented** |

The governance model is **browser-enforced structural isolation + agent-enforced semantic safety**. The spec does not collapse the two.

## 9. Test体系

- **No local tests**. `git ls-files` has no `test/` or `spec/` directory.
- **External WPT**: `https://wpt.fyi/results/webmcp` referenced in metadata (`index.bs:17`).
- **Build validation**: `w3c/spec-prod` with `BUILD_FAIL_ON: warning` and `make lint` (`bikeshed --dry-run --die-when=late`).
- **No behavioral tests** for registration, execution, cross-origin gates, BFCache, or cancellation.
- **Implementation status** is a document, not a test report (`implementation-status.md`).

Consequently, this package can validate **spec-text consistency and design intent**, but cannot validate runtime behavior.

## 10. External dependencies

| Dependency | Use | Source |
|---|---|---|
| HTML / DOM / WebIDL | platform primitives | `index.bs:85–110` |
| Secure Contexts | `SecureContext` | `index.bs:105` |
| JSON Schema 2020-12 | tool schemas | `index.bs:130–135`, `:1313–1325` |
| MCP specification | vocabulary/reference | `index.bs:120–126`, `:1760` |
| Spotlighting paper | injection mitigation | `index.bs:138–145`, `:2299–2305` |
| Chrome agent security docs | agent guardrails | `index.bs:146–158`, `:2315–2330` |
| Bikeshed / csswg API | build | `Makefile:1–13` |
| `w3c/spec-prod` | CI/deploy | `.github/workflows/auto-publish.yml:1–16` |

## 11. Repository boundary statement

This repository is **not** the implementation. It owns:

- the normative text;
- the explainers and open questions;
- the implementation-status table;
- the build and publication pipeline.

It does **not** own:

- Chrome/Edge/Brave/Firefox/Safari implementations;
- the WPT tests;
- the `@w3c/webmcp-polyfill` package;
- MCP transport or browser-agent runtime;
- any production deployment.

Any cross-boundary claim must be marked `[EXTERNAL]` or downgraded to `Observation`.

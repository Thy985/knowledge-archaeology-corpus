# Repository Snapshot Artifact — webmcp

- **run_id**: `ARCH-2026-11-24-001`
- **repository**: `https://github.com/webmachinelearning/webmcp.git`
- **commit**: `a768d04b4f9ac7f3d4357671db9686675d52c8e5`
- **branch**: `main`
- **commit date**: `2026-11-23 11:50:07 +0100` (2026-11-23 18:50 +0800)
- **commit subject**: `Link to the @w3c/webmcp-polyfill npm package (#387)`
- **tag / describe**: no tag; `a768d04`
- **clone**: `git clone --depth 50`
- **local path**: `/home/user/.doubao/agent_mode/workspace/archaeology-jobs/ARCH-2026-11-24-001/repo`
- **repository size**: 588K
- **tracked files**: 18
- **source lines**: 3,922 (`.bs` + `.md` + `.json` + `.yml`; excludes `.git`)

## File inventory

| File | Lines | Role |
|---|---:|---|
| `index.bs` | 2,355 | Normative Bikeshed spec source |
| `README.md` | 548 | Main explainer, use cases, design, best practices, alternatives, open questions |
| `continuations-explainer.md` | 312 | Non-normative continuation-token proposal |
| `docs/service-workers.md` | 274 | Non-normative background service-worker proposal |
| `declarative-api-explainer.md` | 213 | Non-normative form-based API proposal |
| `security-privacy-questionnaire.md` | 113 | W3C security/privacy self-review |
| `implementation-status.md` | 48 | Browser/agent implementation status |
| `CONTRIBUTING.md` | 20 | W3C CG CLA and contribution rules |
| `.github/workflows/auto-publish.yml` | 16 | Spec build + gh-pages deploy |
| `Makefile` | 13 | Bikeshed build/lint/watch |
| `.github/dependabot.yml` | 11 | Weekly GitHub Actions updates |
| `.pr-preview.json` | 6 | Bikeshed PR preview |
| `w3c.json` | 5 | CG report metadata |
| `LICENSE.md` | 1 | W3C Software and Document License |
| `.gitignore` | — | Build artifacts |
| `assets/*` | — | Logos used by implementation status |
| `.github/*` | — | CI |

There is **no implementation code, no local test suite, and no package manifest**. The repository is a specification/explainer repo. `index.bs` is the normative source; `index.html` is generated and published to `gh-pages`.

## Identity and status

- **Title**: WebMCP
- **Shortname**: `webmcp`
- **Level**: None
- **Status**: `CG-DRAFT`
- **Group**: `webml` (W3C Web Machine Learning Community Group)
- **Repository type**: `cg-report` (`w3c.json`)
- **License**: W3C Software and Document License
- **Editors**: Brandon Walderman (Microsoft), Khushal Sagar (Google), Dominic Farolino (Google), Samuel Dallstream (Microsoft)
- **Test Suite**: `https://wpt.fyi/results/webmcp` (external, not vendored)
- **Build**: `w3c/spec-prod@v2`, `GH_PAGES_BRANCH: gh-pages`, `BUILD_FAIL_ON: warning`
- **Local build**: `make` (csswg API or local `bikeshed`), `make lint`, `make watch`

## Project map

### 1. Normative layer (`index.bs`)

| Area | Lines | Content |
|---|---:|---|
| Metadata / bibliography | 1–158 | Spec status, editors, references (MCP, JSON Schema, Spotlighting, Chrome agent security) |
| Introduction / terminology | 160–176 | Agent, browser agent, AI platform |
| Supporting concepts | 176–323 | Model context, tool definition, annotations, error taxonomy, pending execution |
| Pending executions | 323–470 | Browser-process map, cancel, unload cleanup, toolchange notification |
| API | 711–1420 | `Document.modelContext`, `registerTool`, `getTools`, `executeTool`, dictionaries |
| Tool interface | 1420–1580 | Snapshot object, weak source, origin, annotations |
| Events | 1580–1647 | `toolchange`, `toolactivated`, `toolcancel` |
| Permissions policy | 1647–1655 | `tools` feature, default `self` |
| Agent interaction | 1653–1777 | Event loop, AI agent queue, non-normative observation |
| Security/privacy | 1777–2333 | Threat model, cross-origin gates, mitigations |
| Accessibility | 2333–2336 | Empty section |
| Acknowledgements | 2336–2355 | Contributors |

### 2. Explainer layer

- `README.md`: motivation, use cases, imperative API, lifecycle, declarative future, permissions/iframes, best practices, alternatives, security, open questions.
- `continuations-explainer.md`: cross-document/cross-frame tool continuation tokens.
- `declarative-api-explainer.md`: HTML form attributes, schema synthesis, `SubmitEvent.respondWith()`.
- `docs/service-workers.md`: background tools, JIT installation, routing, session IDs, lethal trifecta.

### 3. Governance layer

- W3C CG draft, not standards-track.
- `CONTRIBUTING.md` requires CG participation for substantive contributions.
- CI builds and deploys on every push to `main`; build fails on warning.
- Dependabot updates GitHub Actions weekly.
- No release tags; commit-based living document.

## Core data structures

| Structure | Defined at | Purpose |
|---|---|---|
| `ModelContext` | `index.bs:739–752` | Per-Document registry and event target |
| `model context` struct | `index.bs:176–188` | `tool map` + local pending executions map |
| `tool definition` struct | `index.bs:190–250` | Tool id, name, title, description, schemas, execute steps, annotations, exposed origins |
| `local pending tool execution` | `index.bs:252–260` | Event-loop-local `AbortController` |
| `pending tool execution` | `index.bs:323–350` | Caller/target documents, tool name, completion steps |
| `annotations` | `index.bs:262–278` | readOnly, untrustedContent, consequential, debugging hints |
| `Tool` | `index.bs:1420–1470` | Snapshot exposed to in-page callers |
| `ToolActivatedEvent` / `ToolCancelEvent` | `index.bs:1602–1647` | Tool name only |
| `observation` | `index.bs:1682–1705` | Browser-agent view: Document ID → origin + tools |

## Entry points

- **Human/spec reader**: `README.md`, generated `index.html`.
- **Normative reader**: `index.bs`.
- **In-page agent**: `document.modelContext.getTools()` / `executeTool()`.
- **Page author**: `document.modelContext.registerTool()`.
- **Built-in browser agent**: implementation-defined `perform an observation` over a top-level traversable.
- **Build**: `make` or `.github/workflows/auto-publish.yml`.

## State and lifecycle

- Document: created → fully active → inactive / BFCache → restored → detached / destroyed.
- Tool: unregistered → registered → discovered (`Tool` snapshot) → pending execution → activated → completed / rejected / cancelled.
- Registration: `AbortSignal` abort unregisters; no `updateTool()`.
- Execution: `AbortSignal` cancels waiting; target callback receives its own signal; unload cleanup removes pending entries.
- Navigation: `Tool.source` becomes null; execution rejects if target window is gone or document not fully active.

## Configuration and policy

- Permissions Policy `tools`, default allowlist `self`.
- `exposedTo`: provider-side cross-origin allowlist, secure origins only.
- `fromOrigins`: consumer-side cross-origin query allowlist, secure origins only.
- `AbortSignal` at registration and execution.
- No authentication, no per-tool user consent, no rate limit, no tool count limit.
- `SecureContext` required; opaque target origins rejected.

## Testing and evidence

- No local tests.
- External WPT suite is referenced but not cloned or executed.
- `make lint` is Bikeshed validation, not behavioral testing.
- CI validates spec build, not runtime behavior.
- Implementation status is a document, not a test result.

## External dependencies

- HTML / DOM / WebIDL / Secure Contexts definitions.
- JSON Schema 2020-12.
- MCP specification (non-normative reference).
- Spotlighting paper and Chrome agent-security guidance (security references).
- Bikeshed / csswg API for build.
- `w3c/spec-prod` GitHub Action.

## Snapshot conclusion

The repository is a **specification and design-debate artifact**, not a runnable implementation. Any claim about “WebMCP behavior” must distinguish:

1. **Normative spec text** (`index.bs`);
2. **Design intent** (`README.md`, explainers, questionnaire);
3. **Implementation status** (`implementation-status.md`);
4. **External runtime behavior** (not present in this repo; WPT and browser implementations are external).

This distinction is the central evidence boundary for the archaeology.

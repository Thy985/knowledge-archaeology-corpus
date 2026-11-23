# 02 · Engineering Knowledge — EK Graph

> 30 Engineering Knowledge nodes (EK-01…EK-30), each with ≥1 edge.
> This is the wide base. It is not a KO list and is not compressed away.
> `[SPEC]` = normative `index.bs`; `[DOC]` = README/explainer; `[META]` = repository/CI/status.

## 2.1 Graph summary

| Edge type | Count |
|---|---:|
| `mechanism` | 14 |
| `subsystem` | 8 |
| `causal` | 13 |
| `dependency` | 6 |
| `constraint` | 19 |
| `contrast` | 16 |
| **Total declared out-edges** | **76** |

Average out-degree: **2.53**. Isolated EK: **0**.

Six clusters:

1. **Registry & identity** (EK-01–05)
2. **Execution & cancellation** (EK-06–09)
3. **Origin & exposure** (EK-10–13, EK-20, EK-23–24)
4. **Semantic trust** (EK-14–19, EK-21)
5. **Future surfaces** (EK-25–27)
6. **Standardization & scope** (EK-28–30)

---

## EK-01 · Document-scoped tool registry

- **claim**: Every `Document` owns one `ModelContext`; tools live in a per-document tool map keyed by a unique internal `tool id`, not in a global namespace.
- **category**: ARCHITECTURE
- **abstraction**: L1
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:711–737`, `index.bs:176–190`, `index.bs:800–810`
- **detail**: `Document.modelContext` is `[SecureContext, SameObject]`; creating a `Document` creates its associated `ModelContext`; the internal `model context` struct contains `tool map` and `local pending tool executions map`. No cross-document registry is defined.
- **scope**: Applies to every document. Does not define a site-wide or origin-wide registry.
- **links**:
  - `subsystem` → EK-02 (same registry)
  - `subsystem` → EK-06 (same execution model)
  - `mechanism` → EK-04 (id-based identity)
- **evidence**: EV-001, EV-002

## EK-02 · Registration validation and name constraints

- **claim**: `registerTool()` rejects inactive documents, disallowed `tools` policy, duplicate names in the same document, empty/invalid names, and empty descriptions; names are 1–128 chars from `[A-Za-z0-9_.-]`.
- **category**: ENGINEERING
- **abstraction**: L1
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:811–850`, `index.bs:220–240`, `index.bs:1263–1280`
- **detail**: The method checks fully-active, `tools` permission, duplicate name, name syntax/length, non-empty description, and schema serializability. It does **not** validate the schema against JSON Schema; it only serializes it.
- **scope**: Per-document. Cross-document duplicate names are allowed.
- **links**:
  - `mechanism` → EK-03 (serialization can re-enter script)
  - `constraint` → EK-10 (policy is a precondition)
  - `causal` → EK-04 (successful registration creates the id)
- **evidence**: EV-003, EV-004

## EK-03 · Reentrancy revalidation after schema serialization

- **claim**: Because `inputSchema`/`outputSchema` serialization can run page script (`toJSON`), `registerTool()` rechecks fully-active and duplicate-name **after** serialization, not only before.
- **category**: ENGINEERING
- **abstraction**: L1
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:843–885`; commit `3279798`
- **detail**: The spec explicitly notes a `toJSON()` method can remove an iframe or register another tool with the same name. The second fully-active check and second duplicate-name check close that reentrancy window.
- **scope**: Any schema object supplied by page code. Does not cover arbitrary getters outside WebIDL conversion; those are platform-level concerns.
- **links**:
  - `causal` → EK-04 (revalidation protects the registration identity)
  - `dependency` → EK-01 (depends on document-scoped map)
- **evidence**: EV-005, EV-006

## EK-04 · Tool id indirection prevents re-registration hijack

- **claim**: A `Tool` carries an internal `tool id`; `executeTool()` looks up by id, not name, so unregister+re-register under the same name cannot silently execute the new tool with old arguments.
- **category**: ENGINEERING
- **abstraction**: L1
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:463–560`, `index.bs:1420–1470`; commit `14b6d53`
- **detail**: `tool not found` is returned if the id is absent. The spec note says this protects a race between `executeTool()` checking existence and the tool being unregistered/re-registered.
- **scope**: All imperative tools. Does not apply to declarative forms (not in spec).
- **links**:
  - `constraint` → EK-05 (snapshot identity constrains execution)
  - `causal` → EK-06 (id is the key into pending execution)
  - `mechanism` → EK-01 (registry identity)
- **evidence**: EV-007, EV-008

## EK-05 · Tool snapshot and weak source

- **claim**: `getTools()` returns new `Tool` snapshots; `source` is a weak `Window`; after navigation or destruction, `source` is null/GC and execution rejects.
- **category**: ARCHITECTURE
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:1420–1535`
- **detail**: Each `getTools()` call creates new objects, never `===` across calls. `Tool.source` returns null if the window is GC'd or the document is not fully active; a `WindowProxy` obtained earlier keeps following the browsing context, but the getter itself refuses to mislead.
- **scope**: In-page API. Built-in agent uses an observation snapshot instead.
- **links**:
  - `dependency` → EK-06 (execution depends on target liveness)
  - `contrast` → EK-04 (snapshot vs registration identity)
- **evidence**: EV-009, EV-010

## EK-06 · Browser-process execution ledger

- **claim**: Pending executions live in a traversable-navigable map mutated in parallel, while each `ModelContext` keeps an event-loop-local `AbortController` map; the split models a browser process that outlives renderer loops.
- **category**: RUNTIME
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:323–395`, `index.bs:1195–1255`
- **detail**: The map is keyed by `uuid` and stores caller/target document, tool name, completion steps. The local map stores event-loop-only `AbortController`s. `executeTool()` creates the pending entry before queueing the target task.
- **scope**: Same traversable navigable. Cross-tab/window execution is not supported.
- **links**:
  - `causal` → EK-08 (ledger enables cancellation)
  - `dependency` → EK-07 (completion uses error taxonomy)
  - `subsystem` → EK-01 (same ModelContext)
- **evidence**: EV-011, EV-012

## EK-07 · Platform-defined error funnel

- **claim**: The tool-execution error table has five platform-defined failure reasons mapped to four exception names (`NotFoundError`, `DataError`, `OperationError`, `AbortError`); API preconditions additionally expose `InvalidStateError`, `NotAllowedError`, `NotSupportedError`, `SecurityError`, and `TypeError`. Raw page exceptions/stack traces are not leaked, and a rejected callback is reported as `OperationError` plus an optional console warning.
- **category**: ENGINEERING
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:280–320`, `index.bs:600–640`, `index.bs:1096–1114`, `index.bs:1225–1245`
- **detail**: `tool not found`→`NotFoundError`, `invalid input`→`DataError`, `tool rejected`→`OperationError`, `unserializable result`→`OperationError`, `target document unloaded`→`AbortError`. Precondition failures (`InvalidStateError`, `NotAllowedError`, `NotSupportedError`, `SecurityError`, `TypeError`) are thrown before the pending execution. Author-defined failures can still be encoded in the response JSON.
- **scope**: Cross-realm execution. Same-origin callers also receive the mapped exception, not the raw one.
- **links**:
  - `mechanism` → EK-08 (cancellation also uses mapped errors)
  - `constraint` → EK-20 (protects same-origin realm details)
- **evidence**: EV-013, EV-014

## EK-08 · Cooperative cancellation and unload cleanup

- **claim**: Caller abort rejects the promise and cancels the pending entry; target callback gets an aborted signal and `toolcancel`; target unload completes with `AbortError`; caller unload cancels; same-document unload removes the entry.
- **category**: RUNTIME
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:356–440`, `index.bs:1140–1150`
- **detail**: Cancellation only stops waiting and aborts the signal; it does not undo side effects already performed by page code. The spec says the callback must cooperate.
- **scope**: Imperative execution. Continuation tokens are a separate proposal.
- **links**:
  - `causal` → EK-09 (cancel fires event)
  - `mechanism` → EK-06 (ledger removal)
  - `contrast` → EK-22 (BFCache vs destroy)
- **evidence**: EV-015, EV-016

## EK-09 · Event timing: activated after invoke, cancel on abort

- **claim**: `toolactivated` fires after the execute callback is invoked and before its promise settles; `toolcancel` fires when a pending execution is cancelled, not when the tool is merely unregistered.
- **category**: ENGINEERING
- **abstraction**: L1
- **value**: B
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:638`, `index.bs:1580–1647`
- **detail**: `ToolActivatedEvent` and `ToolCancelEvent` carry only `toolName`. No tool id, origin, input, or result is included.
- **scope**: Imperative tools. Declarative event semantics are still open (`declarative-api-explainer.md:155–200`).
- **links**:
  - `subsystem` → EK-08 (same execution lifecycle)
  - `causal` → EK-14 (events are part of the page's observable surface)
- **evidence**: EV-017

## EK-10 · Permissions Policy `tools` is the document-level gate

- **claim**: `tools` is a policy-controlled feature with default allowlist `self`; it gates `registerTool`, `getTools`, and `executeTool`, so a cross-origin iframe must be delegated `allow="tools"`.
- **category**: PERMISSION
- **abstraction**: L1
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:1647–1655`, `index.bs:811–830`, `index.bs:959–970`, `index.bs:1096–1110`
- **detail**: A site can disable with `Permissions-Policy: tools=()`, which applies to all descendants including same-origin frames (`index.bs:2268–2283`).
- **scope**: All WebMCP APIs. Does not gate built-in agent observation (see EK-13).
- **links**:
  - `constraint` → EK-11 (policy is gate 1 of 3)
  - `subsystem` → EK-12 (both shape discovery)
- **evidence**: EV-018, EV-019

## EK-11 · Two-sided cross-origin opt-in

- **claim**: Cross-origin access requires three independent gates: embedder `tools` policy, provider `exposedTo`, consumer `fromOrigins`; same-origin is the default. `executeTool` rechecks exposure and returns `NotFoundError` for both missing and not-exposed.
- **category**: PERMISSION
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:872–885`, `index.bs:974–985`, `index.bs:1185–1192`, `index.bs:2159–2210`
- **detail**: `tool is exposed to an origin` returns true for same-origin or an origin in `exposedTo`. `getTools` first filters by same-origin/`fromOrigins`, then by exposure. All listed origins must be potentially trustworthy.
- **scope**: In-page agents in the same traversable navigable. Built-in agent path is separate (EK-13).
- **links**:
  - `mechanism` → EK-10 (policy gate)
  - `constraint` → EK-20 (execution realm boundary)
  - `contrast` → EK-13 (observation path)
- **evidence**: EV-020, EV-021, EV-022

## EK-12 · Descendant-only enumeration asymmetry

- **claim**: `getTools()` enumerates the caller document's inclusive descendant navigables only; a parent can discover/execute a child's exposed tool, but a child cannot discover/execute a parent's tool, and siblings do not see each other.
- **category**: ARCHITECTURE
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:995`, `index.bs:1150–1160`; commit `0c3e712`
- **detail**: The algorithm starts from `toolRequestor`'s active document and walks its inclusive descendants. It never walks ancestors or sibling trees.
- **scope**: In-page API. Cross-top-level documents are rejected (EK-24).
- **links**:
  - `constraint` → EK-11 (direction of exposure)
  - `contrast` → EK-13 (browser agent sees from top-level)
  - `subsystem` → EK-24 (same traversable constraint)
- **evidence**: EV-023, EV-024

## EK-13 · Built-in agent observation is a separate, implementation-defined path

- **claim**: The browser agent does not use `getTools()`; it uses a non-normative `perform an observation` over a top-level traversable, collecting all tools from all descendant documents with origin, without applying `fromOrigins`/`exposedTo`.
- **category**: AGENT
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact (about the spec text) / implementation behavior is Observation
- **source**: `[SPEC] index.bs:1682–1770`
- **detail**: The observation is keyed by `Document` unique ID; each `observed tool collection` has `origin` and `tools`. The algorithm loops `flat descendants` and reads each tool map directly. It says exposure to the browser agent is implementation-defined; the spec does not prescribe MCP format.
- **scope**: Non-normative guidance. Actual browser filtering is unknown.
- **links**:
  - `contrast` → EK-11, EK-12 (different authority path)
  - `dependency` → EK-30 (format is not prescribed)
- **evidence**: EV-025, EV-026

## EK-14 · Tool metadata is untrusted input

- **claim**: Tool names, descriptions, schemas, annotations, and results come from the page; an agent must treat them as untrusted, because descriptions can carry instructions and annotations can be omitted or misrepresented.
- **category**: AGENT
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact (design stance) / runtime compliance unknown
- **source**: `[SPEC] index.bs:1777–1840`, `index.bs:1890–2010`; `[DOC] README.md:337`, `README.md:520`
- **detail**: The security section says “a malicious site can omit or misrepresent those hints” and “tool descriptions and return values could be treated as trusted context by agents”. The spec does not sanitize descriptions.
- **scope**: All page-declared tools. Does not apply to tools from a signed/trusted manifest (not specified here).
- **links**:
  - `constraint` → EK-15 (annotations inherit untrusted status)
  - `constraint` → EK-16 (protocol cannot fix it)
  - `mechanism` → EK-17 (injection surface)
- **evidence**: EV-027, EV-028

## EK-15 · Advisory annotations, default false

- **claim**: `readOnlyHint`, `untrustedContentHint`, `consequentialHint`, `debugging` are booleans defaulting false; they are signals for the agent, not enforcement, and `consequentialHint` may trigger confirmation but is not required.
- **category**: DESIGN
- **abstraction**: L1
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:1263–1355`, `index.bs:2268–2330`
- **detail**: `untrustedContentHint` is about output trust; there is no `untrustedMetadataHint`. `readOnlyHint` may cause an agent to skip confirmation (questionnaire Q12). All are author-supplied.
- **scope**: Imperative tools. Declarative annotations are TBD.
- **links**:
  - `mechanism` → EK-14 (metadata trust)
  - `contrast` → EK-16 (hint vs policy)
  - `causal` → EK-18 (misrepresentation)
- **evidence**: EV-029, EV-030

## EK-16 · Protocol, not policy

- **claim**: The spec explicitly cannot define precise mitigation strategies that agents or user agents must provide; it defines structural boundaries and recommendations, leaving semantic safety, auth, and consent to agent/UA policy.
- **category**: DESIGN
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:1798–1830`, `index.bs:2268–2330`
- **detail**: “This document cannot define precise mitigation strategies…” It lists agent-level guardrails as recommendations and links to Chrome guidance. It does not normatively require confirmation, authentication, or rate limits.
- **scope**: Whole spec. This is a deliberate division of responsibility, not an omission.
- **links**:
  - `constraint` → EK-14, EK-15 (semantic safety outside protocol)
  - `causal` → EK-19 (privacy leakage is an agent-policy problem)
- **evidence**: EV-031, EV-032

## EK-17 · Prompt injection has three page-side surfaces

- **claim**: Injection can enter through tool metadata/descriptions, tool outputs, or the tool implementation itself; each has a different threat actor and asset at risk.
- **category**: FAILURE
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact (threat model) / exploitability Observation
- **source**: `[SPEC] index.bs:1838–2006`
- **detail**: Metadata attack example embeds “SYSTEM INSTRUCTION” in `description`; output attack embeds instructions in reviews/posts; implementation attack targets high-value tools like password reset. The spec notes WebMCP does not inherently expand attack surface because UI already exposes functionality, but the code path differs.
- **scope**: Page-declared tools. Does not quantify success rates.
- **links**:
  - `mechanism` → EK-14 (metadata is the channel)
  - `causal` → EK-18 (misrepresentation)
  - `subsystem` → EK-19 (privacy)
- **evidence**: EV-033, EV-034

## EK-18 · Misrepresentation of intent is structurally unverifiable

- **claim**: There is no guarantee that a tool's declared intent matches its behavior; an agent cannot verify effects before execution, and natural-language descriptions are ambiguous.
- **category**: FAILURE
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact (design gap) / exploitability Observation
- **source**: `[SPEC] index.bs:2006–2073`
- **detail**: The `finalizeCart` example declares “Finalizes the current shopping cart” but actually triggers a purchase. The spec lists no verification mechanism, no behavioral contracts, and “agent trust assumptions”.
- **scope**: Any tool whose description is not a formal contract. A signed manifest with verified code could reduce it, but is not specified.
- **links**:
  - `mechanism` → EK-17 (semantic attack)
  - `constraint` → EK-15 (hint cannot prove intent)
  - `causal` → EK-19 (privacy/abuse)
- **evidence**: EV-035, EV-036

## EK-19 · Over-parameterization creates a privacy pipeline

- **claim**: A site can request excessive personal parameters; a helpful agent may fill them from cross-site context, creating silent profiling, cross-site tracking, and discrimination risk without explicit consent.
- **category**: PERMISSION
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact (risk analysis) / runtime frequency unknown
- **source**: `[SPEC] index.bs:2073–2143`
- **detail**: The malicious example requests age, pregnancy, location, height, skin tone, previous purchases under plausible descriptions. The spec calls it a “personalization-to-fingerprinting pipeline”.
- **scope**: Agents with personalization/cross-site context. A stateless agent with no personal data may not be able to fill them.
- **links**:
  - `mechanism` → EK-17 (semantic manipulation)
  - `constraint` → EK-18 (intent ambiguity)
  - `causal` → EK-16 (policy responsibility)
- **evidence**: EV-037, EV-038

## EK-20 · Same-origin execution realm and serialized result

- **claim**: The `execute` callback runs in the registering document's realm with its DOM, storage, and ambient credentials; cross-origin callers receive only the serialized JSON result, not object references.
- **category**: RUNTIME
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:2143–2260`, `index.bs:1815`, `index.bs:2014`
- **detail**: The security section says a cross-origin document that can discover/invoke a tool crosses a privilege boundary. Execution is in the owner realm. `executeTool` returns `DOMString`; result is serialized in the target realm.
- **scope**: Imperative tools. Does not apply to server-side MCP.
- **links**:
  - `constraint` → EK-11 (exposure gates protect this boundary)
  - `dependency` → EK-06 (execution ledger)
  - `contrast` → EK-23 (opaque origin exception)
- **evidence**: EV-039, EV-040

## EK-21 · Private browsing is delegated to the user agent

- **claim**: The spec does not define private-browsing behavior; it says user agents are responsible for safely exposing private-mode activity to agents, and warns about cross-boundary leakage.
- **category**: PERMISSION
- **abstraction**: L2
- **value**: B
- **epistemic_status**: Fact (gap) / implementation Observation
- **source**: `[SPEC] index.bs:2261–2268`; `security-privacy-questionnaire.md:78–83`
- **detail**: “Users agents are responsible for ensuring that their respective private browsing modes are safely exposed to agents.” No normative algorithm.
- **scope**: Browsers with private modes. Not a WebMCP-specific mechanism.
- **links**:
  - `contrast` → EK-16 (policy delegation)
  - `constraint` → EK-20 (ambient credentials boundary)
- **evidence**: EV-041

## EK-22 · BFCache keeps registrations but blocks execution

- **claim**: A BFCached document keeps its registered tools in memory, but tools cannot be registered, retrieved, or invoked while non-fully-active; on restoration they become available. Pending invocation behavior is not fully spec'd.
- **category**: RUNTIME
- **abstraction**: L1
- **value**: B
- **epistemic_status**: Fact (questionnaire) / spec gap
- **source**: `security-privacy-questionnaire.md:91–100`, `index.bs:1248`
- **detail**: The questionnaire says BFCached tools remain in memory and unavailable; Q19 says disconnected pending invocations are abandoned but “not yet spec'd”. The spec queues the target task and says it runs when the document becomes active again.
- **scope**: BFCache-capable browsers. Not tested in this repo.
- **links**:
  - `contrast` → EK-08 (destroy vs BFCache)
  - `constraint` → EK-06 (pending ledger)
  - `causal` → EK-05 (source liveness)
- **evidence**: EV-042, EV-043

## EK-23 · Opaque origins cannot execute tools

- **claim**: `executeTool()` rejects a tool whose `origin` is opaque with `NotSupportedError`; sandboxed/data-URL documents cannot participate as tool targets.
- **category**: PERMISSION
- **abstraction**: L1
- **value**: B
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:1110–1114`
- **detail**: `Issue(#349)` tracks allowing opaque-origin tools; it is not allowed today.
- **scope**: Execution only. Registration may still occur in an opaque-origin document, but execution is rejected.
- **links**:
  - `constraint` → EK-11 (origin trust)
  - `contrast` → EK-20 (normal origin vs opaque)
- **evidence**: EV-044

## EK-24 · No cross-top-level-document execution

- **claim**: `executeTool()` requires target and caller to be in the same traversable navigable; cross-tab/window execution is rejected. Continuation tokens propose a same-tab workaround, but are not in the spec.
- **category**: RUNTIME
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:1150–1160`, `index.bs:1155`; `[DOC] continuations-explainer.md:1–35`
- **detail**: The spec queues `InvalidStateError` if traversables differ. `Issue(#227)` considers cross-top-level support. Continuations are same-tab, same-origin, one redemption, 30s in Chromium.
- **scope**: In-page execution. Built-in agent observation is cross-document within a top-level traversable.
- **links**:
  - `constraint` → EK-12 (same tree)
  - `contrast` → EK-25 (continuation workaround)
- **evidence**: EV-045, EV-046

## EK-25 · Continuation token proposal

- **claim**: A tool can request one v4 UUID token; it can be redeemed once in a same-origin document in the same tab within a limited time (Chromium 30s); the original callback result is ignored and the resumed callback result becomes the tool result.
- **category**: DESIGN
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Hypothesis (proposal, not spec)
- **source**: `[DOC] continuations-explainer.md:1–312`
- **detail**: Token creation via `ToolExecuteCallbackOptions.invocation.requestToken()`; redemption via `ModelContext.resumeTool(token, callback)`. Cancellation does not revoke an unredeemed token; result is discarded. No chained continuations; no cross-origin; no other tabs.
- **scope**: Same-origin, same-tab, one continuation. Not normative.
- **links**:
  - `contrast` → EK-24 (workaround for cross-document)
  - `mechanism` → EK-27 (similar background/session problem)
- **evidence**: EV-047, EV-048

## EK-26 · Declarative API is deferred, not specified

- **claim**: A form-based declarative API (`toolname`, `tooldescription`, `toolautosubmit`, `toolparamdescription`) is explained but not in the spec; it was removed from `index.bs` on 2026-10-08. Schema synthesis and cross-document response remain TBD.
- **category**: DECISION
- **abstraction**: L2
- **value**: B
- **epistemic_status**: Fact (removed) / Hypothesis (future)
- **source**: `[DOC] declarative-api-explainer.md:1–213`; commit `14ae813`
- **detail**: The explainer proposes `SubmitEvent.agentInvoked`, `respondWith()`, `:tool-form-active`, `:tool-submit-active`, and `application/ld+json` response extraction. `Input schema synthesis` is TODO. The spec currently defines only imperative tools.
- **scope**: Future work. Do not treat as current behavior.
- **links**:
  - `contrast` → EK-01, EK-02 (imperative registry vs form synthesis)
  - `mechanism` → EK-27 (another surface proposal)
- **evidence**: EV-049, EV-050

## EK-27 · Service worker background proposal

- **claim**: A separate explainer proposes service-worker WebMCP tools with JIT installation, origin/scope tool ownership, session IDs, and routing ambiguity between tabs and workers; it explicitly names the “Lethal Trifecta” and recommends single-origin session limits.
- **category**: DESIGN
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Hypothesis (proposal)
- **source**: `[DOC] docs/service-workers.md:1–274`
- **detail**: Tools are scoped to worker origin/scope; `self.modelContext.registerTool()`; `options.sessionId` for per-conversation state; `placeOrder()` can open a secure payment window and communicate via `postMessage`. Discovery layer and JIT install are TBD.
- **scope**: Background/off-tab tools. Not normative.
- **links**:
  - `contrast` → EK-13 (page observation vs background worker)
  - `mechanism` → EK-25 (session/continuation state)
  - `subsystem` → EK-24 (routing boundary)
- **evidence**: EV-051, EV-052

## EK-28 · Implementation-first standardization

- **claim**: The spec is CG-DRAFT while Chrome 149 and Edge 150 run origin trials, ChatGPT Desktop supports it, Brave 1.102.2 is expected 2027-01-05, Meta Ray-Ban is “coming soon”, and Firefox/Safari only have standards positions.
- **category**: DECISION
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact (status document) / runtime Observation
- **source**: `[META] implementation-status.md:1–48`, `index.bs:3`
- **detail**: Chrome and Edge origin trials; Brave release expected January 5, 2027; ChatGPT Desktop listed; Meta Ray-Ban Display “coming soon”; Firefox Bugzilla 2018306; Safari standards-position 670. This is adoption ahead of standards-track.
- **scope**: Browser/agent implementations listed. Does not prove feature completeness or security.
- **links**:
  - `contrast` → EK-29 (spec process vs implementation)
  - `constraint` → EK-30 (semantic scope still unsettled)
- **evidence**: EV-053, EV-054

## EK-29 · Spec CI with no local behavioral tests

- **claim**: CI builds the spec with `w3c/spec-prod` and fails on warnings, but the repository has no local behavioral tests; the WPT suite is external and not executed here.
- **category**: TESTING
- **abstraction**: L1
- **value**: B
- **epistemic_status**: Fact
- **source**: `.github/workflows/auto-publish.yml:1–16`, `Makefile:1–13`, `index.bs:17`
- **detail**: `BUILD_FAIL_ON: warning`; `make lint` is Bikeshed; `Test Suite: https://wpt.fyi/results/webmcp` is a link, not a vendored test. There is no `npm test` or browser test.
- **scope**: This repository. External WPT may test behavior.
- **links**:
  - `constraint` → EK-28 (CI governs publication, not behavior)
  - `contrast` → EK-30 (text validation vs semantic validation)
- **evidence**: EV-055, EV-056

## EK-30 · The name overstates protocol alignment

- **claim**: “WebMCP” is a page-side tool registry API; the spec explicitly does not prescribe the format exposed to the browser agent and says browsers may use MCP or proprietary function calling.
- **category**: DECISION
- **abstraction**: L2
- **value**: A
- **epistemic_status**: Fact
- **source**: `[SPEC] index.bs:1760`, `[DOC] README.md:464–484`
- **detail**: The spec says “Despite the name Web*MCP*, this specification does not prescribe the format in which tools are exposed to the browser agent.” README alternatives say direct MCP adoption was rejected because MCP lacks origins, permissions, DOM, tab lifecycle, and is evolving.
- **scope**: Whole design. MCP is a vocabulary/reference, not the wire protocol.
- **links**:
  - `contrast` → EK-13 (browser agent path)
  - `mechanism` → EK-16 (protocol vs policy)
  - `constraint` → EK-28 (implementation divergence)
- **evidence**: EV-057, EV-058

---

## 2.2 Edge list (selected, all declared)

| From | To | Type | Note |
|---|---|---|---|
| EK-01 | EK-02 | subsystem | same registry |
| EK-01 | EK-06 | subsystem | same execution model |
| EK-01 | EK-04 | mechanism | id identity |
| EK-02 | EK-03 | mechanism | schema serialization reentrancy |
| EK-02 | EK-10 | constraint | policy precondition |
| EK-02 | EK-04 | causal | registration creates id |
| EK-03 | EK-04 | causal | revalidation protects identity |
| EK-03 | EK-01 | dependency | document-scoped map |
| EK-04 | EK-05 | constraint | snapshot identity |
| EK-04 | EK-06 | causal | id key into execution |
| EK-05 | EK-06 | dependency | target liveness |
| EK-05 | EK-04 | contrast | snapshot vs registration |
| EK-06 | EK-08 | causal | ledger enables cancel |
| EK-06 | EK-07 | dependency | completion uses errors |
| EK-07 | EK-08 | mechanism | mapped errors |
| EK-07 | EK-20 | constraint | realm leak prevention |
| EK-08 | EK-09 | causal | cancel event |
| EK-08 | EK-06 | mechanism | ledger removal |
| EK-08 | EK-22 | contrast | destroy vs BFCache |
| EK-09 | EK-08 | subsystem | execution lifecycle |
| EK-09 | EK-14 | causal | observable surface |
| EK-10 | EK-11 | constraint | gate 1 |
| EK-10 | EK-12 | subsystem | discovery shape |
| EK-11 | EK-20 | constraint | execution boundary |
| EK-11 | EK-13 | contrast | observation path |
| EK-12 | EK-11 | constraint | exposure direction |
| EK-12 | EK-13 | contrast | top-level observation |
| EK-12 | EK-24 | subsystem | same traversable |
| EK-13 | EK-30 | dependency | format not prescribed |
| EK-14 | EK-15 | constraint | annotations untrusted |
| EK-14 | EK-16 | constraint | protocol cannot fix |
| EK-14 | EK-17 | mechanism | injection surface |
| EK-15 | EK-16 | contrast | hint vs policy |
| EK-15 | EK-18 | causal | misrepresentation |
| EK-16 | EK-14 | constraint | semantic safety outside |
| EK-16 | EK-19 | causal | privacy policy |
| EK-17 | EK-18 | causal | semantic attack |
| EK-17 | EK-19 | subsystem | privacy |
| EK-18 | EK-15 | constraint | hint cannot prove intent |
| EK-18 | EK-19 | causal | abuse |
| EK-19 | EK-16 | causal | agent policy |
| EK-20 | EK-11 | constraint | gates protect realm |
| EK-20 | EK-06 | dependency | execution ledger |
| EK-20 | EK-23 | contrast | opaque exception |
| EK-21 | EK-16 | contrast | delegated policy |
| EK-21 | EK-20 | constraint | ambient credentials |
| EK-22 | EK-08 | contrast | destroy vs BFCache |
| EK-22 | EK-06 | constraint | pending ledger |
| EK-22 | EK-05 | causal | source liveness |
| EK-23 | EK-11 | constraint | origin trust |
| EK-23 | EK-20 | contrast | normal vs opaque |
| EK-24 | EK-12 | constraint | same tree |
| EK-24 | EK-25 | contrast | continuation workaround |
| EK-25 | EK-27 | mechanism | session/continuation |
| EK-26 | EK-01 | contrast | imperative vs form |
| EK-26 | EK-02 | contrast | registration vs synthesis |
| EK-27 | EK-13 | contrast | page vs worker |
| EK-27 | EK-25 | mechanism | session state |
| EK-27 | EK-24 | subsystem | routing boundary |
| EK-28 | EK-29 | contrast | adoption vs tests |
| EK-28 | EK-30 | constraint | semantic scope |
| EK-29 | EK-28 | constraint | CI publication |
| EK-29 | EK-30 | contrast | text vs semantics |
| EK-30 | EK-13 | contrast | browser agent path |
| EK-30 | EK-16 | mechanism | protocol vs policy |
| EK-30 | EK-28 | constraint | implementation divergence |

## 2.3 Cluster integrity check

| Cluster | EK members | Connecting edge types | Candidate KO |
|---|---|---|---|
| Registry & identity | EK-01, EK-02, EK-03, EK-04, EK-05 | subsystem, mechanism, causal, constraint | KO-L1-01, KO-L3-02 |
| Execution & cancellation | EK-06, EK-07, EK-08, EK-09 | subsystem, causal, dependency, contrast | KO-L1-02, KO-L2-02, KO-L3-03 |
| Origin & exposure | EK-10, EK-11, EK-12, EK-13, EK-20, EK-23, EK-24 | constraint, contrast, subsystem | KO-L1-03, KO-L2-03, KO-L3-01 |
| Semantic trust | EK-14, EK-15, EK-16, EK-17, EK-18, EK-19, EK-21 | mechanism, constraint, causal | KO-L1-04, KO-L2-05, KO-L3-04, KO-L4-01, KO-L5-01 |
| Future surfaces | EK-25, EK-26, EK-27 | mechanism, contrast | KO-L3-05? (not selected; see candidates) |
| Standardization | EK-28, EK-29, EK-30 | contrast, constraint | KO-L2-04, KO-L3-05 |

No EK is isolated. No KO cluster uses “same subsystem” as its only reason.

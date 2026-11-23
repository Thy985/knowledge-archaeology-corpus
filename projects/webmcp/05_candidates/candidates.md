# 05 · Candidates — open questions, contradictions, cross-project hypotheses

> Candidates are **not** knowledge. They are retained so uncertainty is visible.
> Each has a test that could move it to Fact, Pattern, or rejection.

| ID | Type | Confidence | Status |
|---|---|---|---:|
| C-01 | Contradiction | 0.72 | OPEN |
| C-02 | Bypass | 0.80 | OPEN |
| C-03 | Gap | 0.70 | OPEN |
| C-04 | Contradiction | 0.78 | OPEN |
| C-05 | Design constraint | 0.85 | OPEN |
| C-06 | Gap | 0.76 | OPEN |
| C-07 | Risk | 0.68 | OPEN |
| C-08 | Spec gap | 0.60 | OPEN |
| C-09 | Spec gap | 0.65 | OPEN |
| C-10 | Security hypothesis | 0.58 | OPEN |
| C-11 | Security hypothesis | 0.55 | OPEN |
| C-12 | Design hypothesis | 0.52 | OPEN |
| C-13 | Cross-project hypothesis | 0.45 | OPEN |
| C-14 | Contradiction | 0.70 | OPEN |

---

## C-01 · Built-in agent default exposure contradicts the README

- **claim**: README says the running idea is “top-level document: missing `exposedTo` exposes to built-in agent; iframe: missing `exposedTo` does not expose”; the normative observation algorithm applies no such filter and collects all descendant tools.
- **evidence**: `README.md:520` vs `index.bs:1724–1745`
- **why not promoted**: README explicitly calls it a running idea; the spec’s observation section is non-normative. Neither is normative enough to assert the actual browser behavior.
- **test**: Read Chrome/Edge implementation source or run a top-level + cross-origin iframe trial with a built-in agent; observe whether iframe tools appear. Not possible in this repo.
- **impact**: If the README behavior is implemented, the built-in agent sees more than the in-page API; if not, iframe tools leak to the agent by default.

## C-02 · Observation path bypasses `exposedTo` / `fromOrigins`

- **claim**: The `perform an observation` algorithm loops all `flat descendants` and reads each tool map directly; it does not call `tool is exposed to an origin` and does not filter by `fromOrigins`.
- **evidence**: `index.bs:1724–1745`
- **why not promoted**: Exposure to the browser agent is explicitly implementation-defined; a browser may filter before or after the algorithm.
- **test**: Instrument a browser build or read its WebMCP implementation; check whether `exposedTo` is honored for the native agent.
- **impact**: The three-gate model may protect in-page callers but not the most privileged agent.

## C-03 · No `untrustedContentHint` equivalent for metadata

- **claim**: `untrustedContentHint` marks **output** as untrusted; there is no annotation that marks the tool's **description/schema** as untrusted. The security section says metadata can be poisoned, but the annotation vocabulary does not cover it.
- **evidence**: `index.bs:1263–1355`, `index.bs:1826–1830`, `index.bs:1851–1885`
- **why not promoted**: The spec may consider *all* metadata untrusted by default, making a metadata hint redundant. That is an interpretation, not a stated rule.
- **test**: Search for a `untrustedMetadataHint` in later drafts/issues; ask WG. Not possible now.
- **impact**: An agent that trusts metadata but respects `untrustedContentHint` has a blind spot.

## C-04 · “Schema-conforming arguments” is aspirational

- **claim**: The security questionnaire says cross-origin documents and built-in agents can invoke with “structured, schema-conforming arguments”, but the algorithm only checks that input parses to an object; it does not validate against `inputSchema`.
- **evidence**: `security-privacy-questionnaire.md:49` vs `index.bs:578–585`, `README.md:524`
- **why not promoted**: Native validation is an open issue (#92); the questionnaire may describe the intended contract, not current enforcement.
- **test**: Read WPT `webmcp` tests for schema mismatch behavior; run a browser trial with a bad-typed input.
- **impact**: Any security argument that relies on schema validation is currently unsupported.

## C-05 · `toolactivated` cannot be a pre-execution gate

- **claim**: `toolactivated` fires **after** the execute callback is invoked, so a page cannot use it to approve/deny before side effects. The only pre-execution hook is the callback itself.
- **evidence**: `index.bs:638`
- **why not promoted**: This is a clear design constraint, but whether it is a “bug” depends on intent; the event is observability, not authorization.
- **test**: Check whether any implementation fires the event earlier; inspect commit `f2f2ae2` (“toolactivated fires after execute call”). Already documented.
- **impact**: A page wanting a confirmation gate must implement it inside `execute`, before side effects, not via event.

## C-06 · No per-tool consent or agent identity

- **claim**: The spec defines page-level `tools` policy and origin-level `exposedTo`/`fromOrigins`, but no per-tool user consent, no agent identity, and no “this agent may call this tool” grant. `consequentialHint` is advisory only.
- **evidence**: `index.bs:1647–1655`, `index.bs:2313`, `index.bs:1815`
- **why not promoted**: The spec explicitly leaves user controls to the agent/UA; absence is deliberate.
- **test**: Look at browser permission UI in Chrome 149 trial; check whether a per-tool prompt exists. External.
- **impact**: A compromised/over-eager agent can invoke any exposed tool with the user's ambient session, subject only to its own guardrails.

## C-07 · Description/title length is unbounded

- **claim**: `name` has a 128-char limit, but `description` and `title` have no maximum length. The security section acknowledges input length limits are not fully specified.
- **evidence**: `index.bs:220–240`, `index.bs:2291–2298`
- **why not promoted**: The spec may rely on agent-side truncation/spotlighting; no normative bound.
- **test**: Measure real Chrome/Edge behavior with a 1 MB description; see whether it is accepted and how the agent consumes it.
- **impact**: A large description is a large injection payload; token budget and context poisoning risk.

## C-08 · BFCache pending execution ambiguity

- **claim**: The spec queues the target task and says it runs when the document becomes active again; the questionnaire says pending invocations are abandoned on disconnect but “not yet spec'd”. These are not clearly reconciled.
- **evidence**: `index.bs:1248`, `security-privacy-questionnaire.md:91–100`
- **why not promoted**: The text is explicitly incomplete; behavior may differ by browser.
- **test**: BFCache a page with a pending tool call, restore it, observe whether the callback runs, the caller rejects, or both. External.
- **impact**: A tool call may execute after a long suspension, with stale state; or the caller may hang until abort.

## C-09 · Private browsing boundary unspecified

- **claim**: The spec says user agents are responsible for exposing private-mode activity safely, but gives no normative rule. An agent could join normal and private sessions.
- **evidence**: `index.bs:2261–2268`, `security-privacy-questionnaire.md:78–83`
- **why not promoted**: This is a known browser-policy area, not a WebMCP-specific algorithm.
- **test**: Chrome/Edge private-mode behavior with an agent; check whether tool lists are isolated.
- **impact**: Cross-boundary data joining or retention; privacy leak.

## C-10 · Continuation token is a bearer capability

- **claim**: A continuation token is a v4 UUID, one-shot, origin-bound, same-tab, 30s in Chromium. If leaked to another same-origin document in the same tab, it can be redeemed; cancellation does not revoke an unredeemed token.
- **evidence**: `continuations-explainer.md:40–220`
- **why not promoted**: Explainer only; threat model not formal; no implementation evidence.
- **test**: Prototype the token flow; attempt redemption from a same-origin iframe/sibling; measure whether one-shot is enforced.
- **impact**: A bearer token in page script is accessible to any same-origin script; XSS could redeem it.

## C-11 · Service worker “Lethal Trifecta” routing

- **claim**: The service-worker explainer acknowledges private data + untrusted content + external communication as a lethal trifecta, recommends one origin/session and no web search; routing between tabs and worker is still TBD.
- **evidence**: `docs/service-workers.md:200–274`
- **why not promoted**: Proposal only; no normative routing model.
- **test**: Build a two-tab + one-worker prototype; see which tab/session owns a tool call.
- **impact**: Background tools may broaden authority beyond the active tab and cross session boundaries.

## C-12 · Declarative form schema synthesis may reintroduce injection

- **claim**: The declarative explainer compiles forms into JSON Schema and supports `toolparamdescription`; schema synthesis is TBD. Form labels/descriptions are page-controlled, so the same metadata-injection surface may reappear.
- **evidence**: `declarative-api-explainer.md:1–213`, commit `14ae813`
- **why not promoted**: Not in spec; synthesis algorithm unknown.
- **test**: Read the future declarative spec; test a form with adversarial `toolparamdescription`.
- **impact**: If declarative becomes normative, it may reintroduce every metadata risk in a lower-code form.

## C-13 · Cross-project: authority model comparison

- **hypothesis**: WebMCP’s “positional authority” model (frame tree + origin + policy) is structurally different from Playwright MCP’s “tool namespace exposed by a local MCP server” and browser-use’s “agent owns a browser session and acts via CDP/REPL”. The three models likely produce different failure modes for the same task.
- **evidence**: this run EK-10–13, EK-24; prior corpus `playwright-mcp` (PR #80) and `browser-use` (PR #79)
- **why not promoted**: Only one project is deeply analyzed in this run; prior corpus packages are not re-validated here.
- **test**: Run the same “read a value from a cross-origin iframe and click a button” task in all three; record authority failures and confirmation points.
- **impact**: A benchmark case for “authority surface” as a comparison dimension.

## C-14 · Service worker `self.modelContext` contradicts `[Exposed=Window]`

- **claim**: The service-worker explainer proposes `self.modelContext.registerTool()` in a worker global scope, but the normative spec exposes `ModelContext` only on `Window` (`[Exposed=Window]`). No normative text extends the interface to `ServiceWorkerGlobalScope`.
- **evidence**: `index.bs:739` vs `docs/service-workers.md:80–120`
- **why not promoted**: The explainer is explicit future work; the spec has not added the exposure. This is a design gap, not a runtime contradiction.
- **test**: Search the spec for `ServiceWorkerGlobalScope` / `Exposed=Window,ServiceWorker`; read any future SW integration PR.
- **impact**: A reader could implement the explainer and fail because `self.modelContext` is undefined in a worker. The package initially omitted this contradiction.

---

## Candidate hygiene

- **No candidate is cited as a Fact** in `02` or `03`.
- **C-01/C-02** are the two highest-risk contradictions; both concern the built-in agent path.
- **C-13** is the only cross-project hypothesis; it stays `Hypothesis`, not `Principle`.
- **No candidate was promoted** in this run; all remain for the next refresh or benchmark.

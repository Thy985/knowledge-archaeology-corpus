# Independent Validation Report — WebMCP

> **Auditor stance**: reconstruct from the repository independently; do not use the package as a source of truth.
> **Isolation caveat**: this pass was executed in the same session after the package was written. Isolation was procedural (re-read files before opening package sections), not a separate agent process. The report records where the package was wrong, not just where it agreed.
> **No original artifact was modified.** Corrections are in `reconciliation.md`.

## 1. Independent findings (before comparison)

Re-read order: `index.bs` (metadata → API → security), `README.md`, `security-privacy-questionnaire.md`, `implementation-status.md`, explainers, commit log.

Ten independent conclusions:

1. The repository is a **specification**, not an implementation. There is no `src/`, no test runner, no package manifest.
2. `Document.modelContext` is per-Document, `[SecureContext, SameObject]`, `[Exposed=Window]`.
3. Tools are registered in a document-scoped map by a unique internal `tool id`; `Tool` is a snapshot and `executeTool` resolves by id.
4. Execution uses a traversable-navigable pending map plus a per-`ModelContext` local `AbortController` map.
5. The in-page API has three cross-origin gates: `tools` policy, `exposedTo`, `fromOrigins`.
6. The built-in agent uses a **non-normative** `perform an observation` over all descendants, with no `exposedTo`/`fromOrigins` filter in the algorithm.
7. Annotations are author-supplied booleans; the security section says they can be omitted/misrepresented.
8. There is no native JSON Schema validation; input is only parsed to an object; output is only serialized.
9. The tool-execution error table has **five failure reasons mapped to four exception names**; preconditions add `InvalidStateError`, `NotAllowedError`, `NotSupportedError`, `SecurityError`, `TypeError`.
10. The spec is CG-DRAFT while Chrome 149/Edge 150 origin trials are listed; Brave is “expected” 2027-01-05.

I then opened the package and compared claim-by-claim.

## 2. Findings

| ID | Package claim | Independent finding | Verdict | Evidence |
|---|---|---|---|---|
| A-01 | Document-scoped registry | Confirmed; no global registry | **CONFIRMED** | `index.bs:711–737`, `:176–190` |
| A-02 | Name 1–128; description non-empty | Confirmed; **title/description length unbounded** | **CONFIRMED** | `index.bs:220–240`, `:1263–1280` |
| A-03 | Recheck after schema serialization | Confirmed; explicit note about `toJSON` | **CONFIRMED** | `index.bs:856–885` |
| A-04 | Tool id prevents re-register hijack | Confirmed; execution by id | **CONFIRMED** | `index.bs:524–560`, `:1450–1460` |
| A-05 | New `Tool` per `getTools`; weak source | Confirmed | **CONFIRMED** | `index.bs:1427`, `:1510–1535` |
| A-06 | Two-map execution ledger | Confirmed; “in parallel” + local map | **CONFIRMED** | `index.bs:185–260`, `:323–350` |
| A-07 | “Five error categories including InvalidStateError” | **Wrong**. Table has 5 reasons → 4 exceptions; `InvalidStateError` is a precondition, not a tool-execution error. Package omitted `NotSupportedError`/`NotAllowedError`/`SecurityError`/`TypeError` from the summary | **CONTRADICTED** | `index.bs:280–310`, `:1096–1114` |
| A-08 | Cooperative cancellation | Confirmed; side effects not undone | **CONFIRMED** | `index.bs:356–440` |
| A-09 | `toolactivated` after callback | Confirmed; commit `f2f2ae2` | **CONFIRMED** | `index.bs:638` |
| A-10 | `tools` policy default self | Confirmed | **CONFIRMED** | `index.bs:1647–1655` |
| A-11 | `exposedTo` provider opt-in | Confirmed | **CONFIRMED** | `index.bs:872–885` |
| A-12 | `fromOrigins` consumer opt-in | Confirmed | **CONFIRMED** | `index.bs:974–985` |
| A-13 | Three independent gates | Confirmed; exact phrase in security section | **CONFIRMED** | `index.bs:2159–2210` |
| A-14 | Descendant-only enumeration | Confirmed; no ancestor/sibling walk | **CONFIRMED** | `index.bs:995` |
| A-15 | Observation sees all descendant tools | Confirmed algorithm; downstream filtering implementation-defined | **CONFIRMED** | `index.bs:1724–1745` |
| A-16 | Metadata is untrusted | Confirmed; “could be treated as trusted context” warning | **CONFIRMED** | `index.bs:1845`, `:1851–1885` |
| A-17 | Annotations default false | Confirmed | **CONFIRMED** | `index.bs:1281–1283` |
| A-18 | Spec cannot define precise mitigations | Confirmed; exact language | **CONFIRMED** | `index.bs:1798–1805` |
| A-19 | Three injection surfaces | Confirmed: metadata, output, implementation | **CONFIRMED** | `index.bs:1838–2006` |
| A-20 | `finalizeCart` misrepresentation | Confirmed; comment says “Intentionally ambiguous” | **CONFIRMED** | `index.bs:2049–2060` |
| A-21 | Over-parameterization example | Confirmed; age/pregnant/location/height/skinTone/previousPurchases | **CONFIRMED** | `index.bs:2095–2135` |
| A-22 | Execution realm + ambient credentials | Confirmed; cross-origin privilege boundary | **CONFIRMED** | `index.bs:1815`, `:2014`, `:2143–2160` |
| A-23 | Private browsing delegated | Confirmed; no normative algorithm | **CONFIRMED** | `index.bs:2261–2268` |
| A-24 | BFCache retains tools but blocks use | Confirmed in questionnaire | **CONFIRMED** | `questionnaire:91–92` |
| A-25 | “Authority is positional” (KO-L4-02) | **Over-generalized**. True for in-page API; false for built-in agent observation, which is position-plus-implementation-defined, and can see iframe tools that a child cannot. | **DOWNGRADED** | `index.bs:1724–1760` |
| A-26 | Continuation token one-shot / 30s | Partially confirmed; explainer only, Chromium-specific, not normative | **PARTIALLY_CONFIRMED** | `continuations-explainer.md:40–220` |
| A-27 | Declarative API removed | Confirmed; commit `14ae813` | **CONFIRMED** | commit log; explainer |
| A-28 | Service worker proposal | Partially confirmed as a proposal; no normative text, routing TBD | **PARTIALLY_CONFIRMED** | `docs/service-workers.md:1–274` |
| A-29 | Implementation status | Partially confirmed as a document; “expected” dates, not runtime | **PARTIALLY_CONFIRMED** | `implementation-status.md:1–48` |
| A-30 | Package does not state `self.modelContext` vs `[Exposed=Window]` | **Omission**: SW explainer uses `self.modelContext`, but spec exposes `ModelContext` only on `Window`; the two are not reconciled. | **MISSING** | `index.bs:739`, `docs/service-workers.md:80–120` |

## 3. Statistics

| Verdict | Count |
|---|---:|
| CONFIRMED | 24 |
| PARTIALLY_CONFIRMED | 3 |
| DOWNGRADED | 1 |
| CONTRADICTED | 1 |
| MISSING | 1 |
| OVER_GENERALIZED | 0 |
| NEEDS_HUMAN_REVIEW | 0 |
| **Total** | **30** |

## 4. Three strongest confirmations

1. **Three-gate cross-origin model** (A-13): exact phrase “three independent access-control gates”; each gate has its own algorithm anchor. This is the most reusable structural finding.
2. **Tool id indirection** (A-04): the spec note explicitly describes the unregister/re-register race and the id lookup as the defense. This is a strong, non-obvious engineering detail.
3. **Reentrancy revalidation** (A-03): `toJSON` can remove an iframe / register a duplicate; the second check is a deliberate TOCTOU defense. This is the kind of detail a surface read would miss.

## 5. Three errors caught

1. **A-07 error taxonomy (CONTRADICTED)**: the package collapsed tool-execution errors and API preconditions into “five categories including InvalidStateError”. The repository distinguishes 5 execution failure reasons → 4 exception names, plus 5 precondition exception types. Fixed in reconciliation.
2. **A-25 authority model (DOWNGRADED)**: “authority is positional” is true for in-page callers but not for the built-in agent. The observation algorithm is a separate authority path; the L4 model must be scoped to “in-page authority” or restated as “position + path”.
3. **A-30 service-worker exposure (MISSING)**: the package did not flag that `self.modelContext` in the SW explainer contradicts `[Exposed=Window]` in the spec. This is a real design gap, not a detail.

## 6. Omissions (beyond the one MISSING finding)

- **No runtime evidence**: not an omission of the package (explicitly declared), but a coverage hole. WPT not run, no browser.
- **No Chromium/Edge source audit**: implementation-status claims are unverified.
- **No external MCP comparison**: the package says “not MCP transport” but does not compare message shapes.
- **No accessibility section content**: `index.bs:2333–2336` is empty; the intro mentions assistive technologies, README says not designed for AT ingestion. The package notes the empty section but does not explore the contradiction deeply.
- **No commit-history quantitative analysis**: 50 commits read, not all; design history is partial.

## 7. Benchmark case

**Case**: “Cross-origin iframe tool exposure under three gates.”

Setup:

- Top-level `https://a.example` with an iframe `https://b.example`.
- Provider registers `pay` with `exposedTo: ["https://a.example"]`.
- Consumer calls `getTools({fromOrigins: ["https://b.example"]})`.
- Embedder sets `<iframe allow="tools">`.
- Then remove one gate at a time.

Expected:

| Removed gate | Expected result |
|---|---|
| `allow="tools"` | `registerTool`/`getTools` reject `NotAllowedError` in iframe |
| `exposedTo` | `getTools` returns empty; `executeTool` `NotFoundError` |
| `fromOrigins` | `getTools` returns empty |
| target opaque origin | `executeTool` `NotSupportedError` |

Why it is a good benchmark: it tests structural authority without needing an LLM; it produces deterministic pass/fail; it can be run against Chrome/Edge and compared with the spec.

**Second case (semantic)**: “Annotated `readOnlyHint: true` tool that charges a card.” Expected: no browser-level prevention; agent may skip confirmation. This is a negative test for “annotations are not enforcement.”

## 8. Auditor conclusion

- The package is **substantively correct** on the structural mechanics: registry, id, gates, cancellation, error funnel, annotations, and threat model.
- The package **over-reached** in one place (authority model) and **mis-summarized** one table (error taxonomy).
- The package **missed** one cross-document contradiction (`self.modelContext` vs `Exposed=Window`).
- The biggest residual risk is not a package error; it is the **unverified built-in agent path**. The spec’s most privileged consumer is the least specified.

**Verdict**: `PARTIALLY_CONFIRMED` overall. Structural claims stand; semantic/runtime claims remain provisional.

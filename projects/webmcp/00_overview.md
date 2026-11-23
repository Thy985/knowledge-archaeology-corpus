# 00 · Overview — WebMCP

> Project Archaeology Package · `ARCH-2026-11-24-001` · initial run
> Repository: `webmachinelearning/webmcp` @ `a768d04b4f9ac7f3d4357671db9686675d52c8e5`
> Skill: knowledge-archaeology v3.2

## One-sentence positioning

WebMCP is a **page-declared tool API** that lets a web document register JavaScript functions as structured tools for in-page agents and browser-integrated agents; it standardizes **discovery, exposure, invocation, and cancellation**, but deliberately does **not** standardize intent verification, schema enforcement, authentication, or user consent.

## Why this repository matters

The repository is a W3C Community Group draft, not a product. Its value is not “what code does” but **what a browser vendor group decided to make normative, what it left implementation-defined, and what it moved into explainers**. That makes it a clean case for the question:

> Can an agent-tool protocol standardize the envelope of a tool call without standardizing the trust needed to execute it?

## Core numbers

| Metric | Value |
|---|---:|
| Commit | `a768d04` |
| Commit date | 2026-11-23 |
| Tracked files | 18 |
| Source lines | 3,922 |
| Normative spec | `index.bs`, 2,355 lines |
| Explainer docs | 4 files, 1,347 lines |
| Local tests | 0 |
| External test suite | WPT `webmcp` (not vendored, not run) |
| Engineering Knowledge | 30 (EK-01…EK-30) |
| Knowledge Objects | 19 (5 L1 / 6 L2 / 5 L3 / 2 L4 / 1 L5) |
| Flows | 7 |
| Flow edges | 55 |
| Candidates | 14 |
| Independent findings | 30 |
| Gate | **PARTIAL** |

## Three-layer shape

```
Project Layer
  18 files; spec + explainers + implementation status; no runtime code.
        │
Engineering Knowledge (30 EK)
  registry; execution ledger; error funnel; three-gate exposure;
  annotations; threat model; continuations; declarative/service-worker proposals.
        │
Generalized Knowledge (19 KO)
  L1 5 facts · L2 6 mechanisms · L3 5 patterns · L4 2 models · L5 1 principle
```

## What the archaeology actually found

1. **The tool registry is document-scoped and registration-identified.** `Tool` holds an internal `tool id`; `executeTool` looks up by id, not by name. Re-registering the same name cannot hijack a previously discovered `Tool` (`index.bs:463–540`, `:1420–1470`). This is a deliberate TOCTOU defense.
2. **Execution is mediated by a browser-process-style ledger.** Pending invocations live in a traversable-navigable map mutated in parallel; each `ModelContext` also keeps an event-loop-local `AbortController` map (`index.bs:323–395`). The split is not incidental: it models a browser process that outlives individual renderer event loops.
3. **Cross-origin exposure needs three independent opt-ins.** Permissions Policy `tools`, provider `exposedTo`, consumer `fromOrigins` (`index.bs:2159–2210`). `executeTool` rechecks exposure and returns `NotFoundError` for both “missing” and “not exposed” to avoid disclosure (`index.bs:1180–1192`).
4. **The built-in browser agent takes a different path.** `perform an observation` is non-normative and implementation-defined; its algorithm collects tools from all inclusive descendant navigables with origin, **without** applying `fromOrigins` or `exposedTo` (`index.bs:1724–1760`). The spec explicitly says it does not prescribe the format exposed to the browser agent (`index.bs:1760`).
5. **Annotations are signals, not enforcement.** `readOnlyHint`, `untrustedContentHint`, `consequentialHint`, `debugging` are booleans defaulting false; a malicious site can omit or misrepresent them (`index.bs:1826–1830`). `consequentialHint` may trigger confirmation, but no normative confirmation requirement exists (`index.bs:2313`).
6. **There is no native schema validation.** Schemas are serialized at registration and deep-copied at discovery, but inputs are only parsed to an object and outputs only serialized. The spec tracks native validation as open question #92 (`README.md:524`). “Schema-conforming arguments” in the questionnaire is an aspiration, not an enforced behavior.
7. **The name overstates protocol alignment.** The spec says “Despite the name Web*MCP*, this specification does not prescribe the format in which tools are exposed to the browser agent” (`index.bs:1760`). WebMCP is a page-side registry API; MCP is only a vocabulary/reference.
8. **Adoption is ahead of standardization.** Chrome 149 and Edge 150 origin trials are listed; ChatGPT Desktop is listed as supporting; Brave 1.102.2 is expected 2027-01-05; Firefox and Safari only have standards positions (`implementation-status.md:1–48`). The spec is CG-DRAFT.

## Epistemic declaration

- **Fact**: normative algorithm text, file:line anchors, repository structure, commit history, implementation-status document.
- **Observation**: design intent in README/explainers; the gap between normative text and implementation-defined behavior.
- **Hypothesis**: L4 models and the L5 principle; they are not cross-project validated.
- **Not claimed**: runtime behavior of Chrome/Edge/Brave; WPT pass rate; security effectiveness; exploitability; production readiness.
- **Gate**: PARTIAL because there is no local runtime, no WPT execution, no browser trial, and no external implementation audit. All 30 checked claims trace to repository text, but text is not runtime.

## Package map

- `01_project-layer/project_layer.md` — repository map, modules, lifecycle, governance.
- `02_engineering-knowledge/ek_graph.md` — 30 EK + six edge types + clusters.
- `03_knowledge-layer/knowledge_layer.md` — 19 KO, L1→L5, promotion audit.
- `04_flow-atlas/flow_atlas.md` — 7 flows / 55 edges, including bypass paths.
- `05_candidates/candidates.md` — 12 open questions and cross-project hypotheses.
- `06_validation/evidence_map.md` — 30 evidence checks + validator matrix.
- `06_validation/independent_audit.md` — blind reconstruction and 30 findings.
- `06_validation/reconciliation.md` — audit→package corrections.
- `run_metadata.yaml` — run identity and quality metrics.

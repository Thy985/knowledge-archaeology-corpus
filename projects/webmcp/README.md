# webmcp

`WebMCP` — a W3C Web Machine Learning Community Group draft that lets a web document register JavaScript functions as structured tools for in-page agents and browser-integrated agents.

## Run index

| Run | Mode | Date | Gate | Scope |
|---|---|---|---|---|
| `ARCH-2026-11-24-001` | initial | 2026-11-24 | PARTIAL | `webmachinelearning/webmcp` @ `a768d04`, no tag, CG-DRAFT |

## Why this project matters

- It is the **page-side contract layer** for agent tools, not another browser automation harness: the page declares tools; the browser mediates discovery, exposure, invocation, and cancellation.
- It contrasts cleanly with the existing corpus: `browser-use` (persistent REPL + CDP), `browser-harness` (CDP control plane), `playwright-mcp` (a11y snapshot + ref actions) vs WebMCP (page-declared tools).
- Its central tension is explicit: it standardizes the **envelope** (origins, permissions, schemas, invocation, errors) but refuses to standardize **intent**, **consent**, **schema validation**, or **agent guardrails**.
- It is an implementation-first standardization case: Chrome 149 / Edge 150 origin trials while the spec is CG-DRAFT; Brave is “expected” 2027-01-05; Firefox/Safari have positions only.

## Read order

1. `00_overview.md` — positioning, core numbers, eight findings
2. `01_project-layer/project_layer.md` — repository boundary, modules, lifecycle, governance
3. `02_engineering-knowledge/ek_graph.md` — 30 EK with evidence
4. `03_knowledge-layer/knowledge_layer.md` — 19 KO, L1→L5
5. `04_flow-atlas/flow_atlas.md` — 7 flows, 55 edges, bypass paths
6. `05_candidates/candidates.md` — 14 open questions / contradictions
7. `06_validation/evidence_map.md` — 58 checks, 49 PASS / 9 PARTIAL
8. `06_validation/independent_audit.md` — 30 findings, 24 CONFIRMED / 3 PARTIAL / 1 DOWNGRADED / 1 CONTRADICTED / 1 MISSING
9. `06_validation/reconciliation.md` — audit-to-package corrections

## Boundary note

The repository is **specification + explainer + status**, not runtime. There is no `src/`, no local test suite, and no package manifest. `index.bs` (2,355 lines) is the normative source; `README.md` (548 lines) is the main explainer; `continuations-explainer.md`, `declarative-api-explainer.md`, and `docs/service-workers.md` are future-work proposals. Claims are labelled `[SPEC]`, `[DOC]`, or `[META]`.

## Gate

**PARTIAL**: 49 of 58 checked claims are exact text matches; 9 are explicit partials due to non-normative status, implementation-defined behavior, or a gap. No WPT run, no Chrome/Edge trial, no browser implementation source audit. The package is a complete specification archaeology, not a runtime validation.

## Core finding

WebMCP’s most privileged consumer is its least specified. The in-page API has three cross-origin gates; the built-in browser agent uses a non-normative observation that collects all descendant tools without those filters. That gap — not any individual API detail — is the project’s main knowledge contribution and its main unresolved risk.

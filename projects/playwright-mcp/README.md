# playwright-mcp

`@playwright/mcp` v0.0.86 — Microsoft's official MCP server for browser automation, a11y-snapshot first, with a thin release wrapper around a pinned `playwright-core` implementation.

## Run index

| Run | Mode | Date | Gate | Scope |
|---|---|---|---|---|
| `ARCH-2026-11-23-001` | initial | 2026-11-23 | PARTIAL | `microsoft/playwright-mcp` @ `2337aff`, tag `v0.0.86` |

## Why this project matters

- It is the clearest official example of the **structured accessibility snapshot + element ref** route, contrasting with `browser-use` 1.0's persistent REPL + raw CDP route.
- Its default 25-tool surface includes `browser_run_code_unsafe` (self-described RCE-equivalent), `browser_evaluate` (arbitrary page JS), and `browser_network_request` (headers/bodies).
- It defaults to a **persistent, workspace-derived Chrome profile**, so login-state continuity is the default and isolation is an opt-in.
- Its HTTP transport has **no authentication**, only a Host allowlist for DNS-rebinding protection; the product explicitly says it is not a security boundary.

## Read order

1. `00_overview.md` — positioning and core numbers
2. `01_project-layer/project_layer.md` — repository boundary, modules, lifecycle
3. `02_engineering-knowledge/ek_graph.md` — 30 EK with evidence
4. `03_knowledge-layer/knowledge_layer.md` — L1→L5
5. `04_flow-atlas/flow_atlas.md` — seven flows, 52 edges
6. `05_candidates/candidates.md` — unresolved questions
7. `06_validation/validation_report.md` — evidence map + gate
8. `06_validation/independent_validation_report.md` — blind audit
9. `06_validation/reconciliation.md` — audit-to-package changes

## Boundary note

The target repository owns only the wrapper, types, Docker, README, and packaged-server tests. The MCP server and browser tools live upstream in `microsoft/playwright`, pinned as `playwright-core@1.66.0-alpha-2026-11-18` (`gitHead` `8124fc39233c4c349b9dbebd4abc4103738eae81`). Facts are labelled `[REPO]` vs `[DEP]`.

## Gate

**PARTIAL**: all 34 checked claims trace to file:line or npm metadata, but no `npm ci`, no `npm test`, no browser runtime, and no upstream extension source were available. The Docker "any MCP client" statement is preserved as a contradiction with the Host allowlist.

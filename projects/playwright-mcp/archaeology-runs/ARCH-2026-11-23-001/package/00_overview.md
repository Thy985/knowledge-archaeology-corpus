# 00 · Overview — playwright-mcp (ARCH-2026-11-23-001)

## One-sentence positioning

`@playwright/mcp` v0.0.86 is Microsoft's official **MCP control plane for a real browser**: an LLM client sends structured tool calls, the server captures an accessibility snapshot, the model selects an element by `ref` or selector, and Playwright executes the action. It is deliberately **a11y-first, not vision-first**, but it is not a security boundary.

## Core question

How does a production browser-control server turn an LLM into a deterministic browser operator — and where does its trust boundary actually sit?

The repository answers this in two layers:

- **This repo** (44 files) owns the executable wrapper, public types, Docker image, README, and packaged-server tests.
- **The implementation** lives in `microsoft/playwright` at `packages/playwright-core/src/tools/`, pinned here as `playwright-core@1.66.0-alpha-2026-11-18` (upstream commit `8124fc3`).

A correct archaeology therefore has to cross the repository boundary. This package labels every claim **[REPO]** or **[DEP]**.

## Core numbers

- Target: `microsoft/playwright-mcp`, `main` @ `2337aff0213d28db41d3e3daf48cee19b3d0bd82`, tag `v0.0.86`, commit `2026-11-18T10:49:18-08:00`.
- Package: `@playwright/mcp@0.0.86`; Node ≥ 20; Apache-2.0; 44 tracked files; 6,103 tracked lines; 3.1 MB.
- Tests: 9 cases, 809 lines; all exercise the packaged `cli.js` over stdio.
- Implementation dependency: `playwright` + `playwright-core` pinned to `1.66.0-alpha-2026-11-18`; npm `gitHead` `8124fc39233c4c349b9dbebd4abc4103738eae81`.
- Upstream tool implementation: `mcp/` + `backend/` = 8,868 lines of TypeScript (whole `tools/` tree: 93 files / 16,568 lines).
- Default tool surface: **25 tools**, including `browser_run_code_unsafe`, `browser_evaluate`, and `browser_network_request`.
- Default browser: chromium channel `chrome`; Linux without `DISPLAY` → headless; headless viewport 1280×720.
- Default profile: persistent, `${cache}/ms-playwright-mcp/mcp-${browserToken}-${sha256(clientCwd).slice(0,7)}`.
- HTTP transport: no authentication; Host allowlist only, explicitly DNS-rebinding protection.

## What this project contributes to the knowledge base

1. **A concrete a11y-first action loop**: aria snapshot → `ref`/selector/locator → Playwright action → fresh snapshot. This is the cleanest official counterpoint to browser-use 1.0's REPL + raw CDP route.
2. **A capability-tiered tool registry**: `core*` is always on; `vision`, `pdf`, `devtools` are CLI opt-ins; `storage`, `network`, `config`, `testing` are config-file capabilities but not CLI `--caps` values. Default-open core includes RCE-equivalent code execution.
3. **A state-continuity design**: persistent profile keyed by client workspace, attach modes for CDP / extension / remote, and isolated contexts as an explicit alternative. Login state inheritance is the default, not the exception.
4. **A modal-exclusivity state machine**: an open dialog or file chooser blocks every tab tool except the designated handler. This is a simple, reusable safety pattern for page-level interrupts.
5. **A sharply bounded trust model**: the project itself says it is not a security boundary; `secrets`, origin filters and file guards are convenience defenses. The real boundary is the MCP client/transport, not the tool annotations.

## Knowledge contribution and limits

**Strong evidence (Fact/Observation):**

- repository/implementation split
- exact default tool list
- RCE-equivalent tool and page JS evaluation
- persistent profile derivation
- no HTTP auth; Host allowlist semantics
- modal gating
- aria snapshot/ref pipeline
- test scope (9 packaged-server tests, not upstream tests)

**Pattern-level (L3, reusable but single-project evidence):**

- a11y-first + vision opt-in
- core-always-on + capability opt-in
- state continuity by default, isolation by exception
- modal-state exclusivity gate
- attached-vs-owned browser lifecycle

**Hypothesis-level (L4, not proven law):**

- "Default-open control plane" is a recurring risk shape when code-execution primitives sit in the default tool set of an unauthenticated local/HTTP service. This is supported here, but cross-project validation is pending.

**Principle-level (L5, scoped):**

- In-tool convenience guards must not be used as a security boundary. This is stated by the project and reinforced by `origin='code'` bypass, `--allowed-hosts '*'`, and `--no-sandbox`. Scope: browser-control/agent tooling; cross-project validation pending.

## Epistemic declaration

- **Fact**: directly visible in repository or pinned dependency, with file:line.
- **Observation**: a pattern in code/config, not necessarily intended.
- **Hypothesis**: an inference that needs another project or runtime test.
- **Pattern (L3)**: a reusable implementation template, not a universal rule.
- **Cognitive Model (L4)**: a compact explanatory model, explicitly provisional.
- **Principle (L5)**: a scoped operational rule; never written as a validated universal law.

No test was executed. No browser was launched. No exploit was demonstrated. All quantitative claims come from repository files or `npm view` metadata.

## Package map

- `01_project-layer/` — repository boundary, inventory, modules, lifecycle, config, tests, governance.
- `02_engineering-knowledge/` — 30 EK entries with evidence and edges.
- `03_knowledge-layer/` — L1→L5, with explicit non-promotion notes.
- `04_flow-atlas/` — Control / State / Data / Evidence / Authority / Memory / Policy.
- `05_candidates/` — unresolved claims, cross-project hypotheses, benchmark case.
- `06_validation/` — evidence map, independent audit, reconciliation, gate result.

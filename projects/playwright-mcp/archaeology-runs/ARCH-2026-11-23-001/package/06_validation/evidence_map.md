# 06 · Validation & Evidence Map

## Evidence map

All paths are relative to `repo/` unless marked `[DEP]` (pinned upstream `8124fc3`) or `[NPM]`.

| ID | Claim | Evidence | Type | Verdict |
|---|---|---|---|---|
| E-01 | Repo is thin wrapper | `AGENTS.md:3-20`; `index.js:17`; `cli.js:17-37`; `src/README.md` | `[REPO]` | CONFIRMED |
| E-02 | No build step | `package.json:24` | `[REPO]` | CONFIRMED |
| E-03 | Pinned dependency commit | `package.json:40-43`; `npm view playwright-core@1.66.0-alpha-2026-11-18 gitHead` | `[REPO]` + `[NPM]` | CONFIRMED |
| E-04 | Default 25 tools | `tests/capabilities.spec.ts:19-48` | `[REPO]` test | CONFIRMED |
| E-05 | RCE tool is core/default | `backend/runCode.ts:37-44` `[DEP]`; default test | `[DEP]` + `[REPO]` | CONFIRMED |
| E-06 | RCE description | README:1134-1136 | `[REPO]` docs | CONFIRMED |
| E-07 | Evaluate is core/default | `backend/evaluate.ts:32-64` `[DEP]`; default test | `[DEP]` | CONFIRMED |
| E-08 | Network request is core/default | `backend/network.ts:30-34,140-174` `[DEP]`; default test | `[DEP]` | CONFIRMED |
| E-09 | Default browser/chrome | `mcp/config.ts:226-274` `[DEP]` | `[DEP]` | CONFIRMED |
| E-10 | Linux headless default | `mcp/config.ts:148-149` `[DEP]` | `[DEP]` | CONFIRMED |
| E-11 | Persistent profile derivation | `mcp/browserFactory.ts:193-239` `[DEP]`; README:478-488 | `[DEP]` + docs | CONFIRMED |
| E-12 | Factory precedence | `mcp/browserFactory.ts:66-87` `[DEP]` | `[DEP]` | CONFIRMED |
| E-13 | isolated vs userDataDir | `mcp/config.ts:245-246` `[DEP]` | `[DEP]` | CONFIRMED |
| E-14 | HTTP no auth | `utils/mcp/http.ts:37-161` `[DEP]` | `[DEP]` | CONFIRMED (absence of auth) |
| E-15 | Host allowlist semantics | `utils/mcp/http.ts:57-85` `[DEP]`; README:681-685 | `[DEP]` | CONFIRMED |
| E-16 | Docker "any client" contradiction | README:896-904 vs `http.ts:48-53,63-70` | `[REPO]` vs `[DEP]` | CONTRADICTED |
| E-17 | Config capability not default | `backend/config.ts:20-33` `[DEP]`; `tools.ts:83-94` | `[DEP]` | CONFIRMED |
| E-18 | CLI caps enum limited | `mcp/program.ts:42` `[DEP]` | `[DEP]` | CONFIRMED |
| E-19 | Config/env caps wider | `mcp/config.ts:387,440` `[DEP]`; `config.d.ts` | `[DEP]` | CONFIRMED |
| E-20 | Modal gate | `backend/tool.ts:70-88` `[DEP]`; `tab.ts:224-237` | `[DEP]` | CONFIRMED |
| E-21 | Aria snapshot/ref | `backend/tab.ts:495-548,592-619`; `snapshot.ts:35-55` `[DEP]` | `[DEP]` | CONFIRMED |
| E-22 | Actions add snapshot | `backend/response.ts:161-164` `[DEP]` | `[DEP]` | CONFIRMED |
| E-23 | Crash reset | `backend/context.ts:225-238` `[DEP]` | `[DEP]` | CONFIRMED |
| E-24 | Request cap 200 | `backend/tab.ts:83-84,273-278` `[DEP]` | `[DEP]` | CONFIRMED |
| E-25 | Download auto-save | `backend/tab.ts:239-252` `[DEP]` | `[DEP]` | CONFIRMED |
| E-26 | File guard bypass | `backend/context.ts:582-598` `[DEP]` | `[DEP]` | CONFIRMED |
| E-27 | Redaction text-only | `backend/response.ts:123-129,174-235` `[DEP]` | `[DEP]` | CONFIRMED |
| E-28 | Extension relay | `mcp/cdpRelay.ts:230-280`; `extensionContextFactory.ts:31-51` `[DEP]` | `[DEP]` | CONFIRMED (architecture) |
| E-29 | Extension token not relay-verified | `mcp/cdpRelay.ts:120,158-193,230-245` `[DEP]` | `[DEP]` | PARTIALLY_CONFIRMED |
| E-30 | Idle 1h own headless | `mcp/browserFactory.ts:61-64`; `idleTimer.ts:17` `[DEP]` | `[DEP]` | CONFIRMED |
| E-31 | Watchdog 15s | `mcp/watchdog.ts:21-37` `[DEP]` | `[DEP]` | CONFIRMED |
| E-32 | 9 packaged tests | `tests/*.spec.ts`; `AGENTS.md`; `CONTRIBUTING.md` | `[REPO]` | CONFIRMED |
| E-33 | CI matrix / Trivy | `.github/workflows/ci.yml` | `[REPO]` | CONFIRMED |
| E-34 | Docker non-root/no-sandbox | `Dockerfile:12-103` | `[REPO]` | CONFIRMED |
| E-35 | Test execution absent | no `npm ci`/`npm test` artifacts | run log | MISSING |

## Validator matrix

| Validator | Question | Result | Notes |
|---|---|---|---|
| Truth | Is every claim traceable? | **PASS** | 34/34 checked claims map to file:line or npm metadata; 1 contradiction flagged |
| Coverage | Are core mechanisms covered? | **PARTIAL** | Default control/data/memory/policy covered; extension internals, upstream `tests/mcp`, browser runtime not covered |
| Flow | Are flow edges real? | **PASS (with caveat)** | 52 edges; all but extension token edge have symbol/condition; caveat = no execution |
| Abstraction | Are L3/L4/L5 justified? | **PARTIAL** | L3 templates are reusable; L4 models single-project; L5 scoped; one overgeneralized "no vision" claim corrected |
| Counterexample | Are disconfirming paths preserved? | **PASS** | 8 bypass paths documented; Docker contradiction preserved; `vm` exploitability not claimed |
| Epistemic | Are Fact/Observation/Hypothesis separated? | **PASS** | 8 L1, 6 L2, 6 L3, 2 L4, 1 L5; promotion audit included |

## Gate result

**PARTIAL** — not FAIL, not PASS.

- **Why not PASS**: no `npm ci` / `npm test`; no browser launch; no runtime proof of Docker Host behavior; no upstream extension source; no npm tarball comparison.
- **Why not FAIL**: every substantive fact is traceable; the core default-surface, state, memory and policy claims are directly code-backed; contradictions are explicitly preserved rather than smoothed.
- **What would make it PASS**: run the 9 tests + a minimal HTTP Host experiment + extension token test + npm tarball hash comparison.

## Blind reconstruction summary

Independent auditor reconstructed 30 findings before the package:

- 27 **CONFIRMED**
- 1 **PARTIALLY_CONFIRMED** (extension token: passed via URL; relay does not compare; extension side unverified)
- 1 **CONTRADICTED** (Docker "any MCP client" vs Host allowlist)
- 1 **MISSING** (no runtime test / browser evidence)

Claim-level attacks added 2 **DOWNGRADED** (L4 "default-open control plane"; L5 convenience guard scope), 1 **OVER_GENERALIZED** ("no vision"), and 1 **NEEDS_HUMAN_REVIEW** (`vm` escape exploitability).

Detailed audit: `06_validation/independent_audit.md`.

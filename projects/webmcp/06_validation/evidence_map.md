# 06 · Validation & Evidence Map

> Every EK claim has at least one evidence check. `PASS` = exact text match; `PARTIAL` = text exists but is non-normative or narrower than the claim; `OPEN` = not answerable from this repo.
> No runtime was executed. This is a **text-and-structure validation**, not a behavior validation.

## 6.1 Evidence checks (58)

| ID | Claim checked | Source | Status |
|---|---|---|---|
| EV-001 | `Document.modelContext` is a per-Document, same-object, secure-context attribute | `index.bs:739–746` | PASS |
| EV-002 | Model context struct has `tool map` + local pending map | `index.bs:176–190` | PASS |
| EV-003 | `registerTool` rejects inactive, disallowed, duplicate, empty/invalid name/description | `index.bs:811–850` | PASS |
| EV-004 | Name is 1–128, ASCII alnum + `_` `-` `.` | `index.bs:1263–1280` | PASS |
| EV-005 | Recheck fully-active and duplicate name after schema serialization | `index.bs:856–885` | PASS |
| EV-006 | Spec note: `toJSON` can remove iframe or register same name | `index.bs:856–866` | PASS |
| EV-007 | `Tool` has internal `tool id` not exposed to script | `index.bs:1450–1460` | PASS |
| EV-008 | `executeTool` looks up by `tool id`, not name | `index.bs:524–560` | PASS |
| EV-009 | Each `getTools` call creates new `Tool` objects | `index.bs:1427` | PASS |
| EV-010 | `Tool.source` is null if window gone / document inactive | `index.bs:1510–1535` | PASS |
| EV-011 | Pending map keyed by uuid on traversable navigable | `index.bs:323–350` | PASS |
| EV-012 | Local pending map holds AbortController per ModelContext | `index.bs:252–260`, `:185` | PASS |
| EV-013 | Error table has 5 failure reasons → 4 exception names | `index.bs:280–310` | PASS |
| EV-014 | Tool rejection maps to `OperationError`; raw error not exposed | `index.bs:624–640` | PASS |
| EV-015 | Caller abort cancels pending, aborts target signal, fires toolcancel | `index.bs:356–395` | PASS |
| EV-016 | Target unload completes with `AbortError` | `index.bs:410–430` | PASS |
| EV-017 | `toolactivated` fires after execute callback | `index.bs:638` | PASS |
| EV-018 | `tools` policy default allowlist `self` | `index.bs:1647–1655` | PASS |
| EV-019 | Policy gates register/get/execute | `index.bs:811–830`, `:959–970`, `:1096–1110` | PASS |
| EV-020 | `exposedTo` is provider-side origin list | `index.bs:872–885`, `:1373–1385` | PASS |
| EV-021 | `fromOrigins` is consumer-side query list | `index.bs:974–985`, `:1393–1405` | PASS |
| EV-022 | Cross-origin requires three gates; default strict same-origin | `index.bs:2159–2210` | PASS |
| EV-023 | `getTools` uses caller's inclusive descendant navigables | `index.bs:995` | PASS |
| EV-024 | `executeTool` rejects different traversable navigables | `index.bs:1150–1160` | PASS |
| EV-025 | Observation loops flat descendants and reads tool maps | `index.bs:1724–1745` | PASS |
| EV-026 | Observation algorithm applies no `exposedTo`/`fromOrigins` filter | `index.bs:1724–1745` | **PARTIAL** (implementation-defined downstream) |
| EV-027 | Security section says metadata can be treated as trusted by agents | `index.bs:1845`, `:1851–1885` | PASS |
| EV-028 | Malicious site can omit/misrepresent hints | `index.bs:1826–1830` | PASS |
| EV-029 | Four annotations default false | `index.bs:1281–1283` | PASS |
| EV-030 | `readOnlyHint` may cause agent to skip confirmation | `security-privacy-questionnaire.md:63` | **PARTIAL** (“may”) |
| EV-031 | Spec cannot define precise mitigation strategies | `index.bs:1798–1805` | PASS |
| EV-032 | Agent-level guardrails are recommended, not mandated | `index.bs:2315–2330` | PASS |
| EV-033 | Metadata/description injection example | `index.bs:1851–1910` | PASS |
| EV-034 | Output injection example | `index.bs:1915–2000` | PASS |
| EV-035 | `finalizeCart` description hides purchase | `index.bs:2049–2060` | PASS |
| EV-036 | No verification mechanism; agent cannot verify effects | `index.bs:2067–2068` | PASS |
| EV-037 | Over-parameterization example (age, pregnant, location, skin tone) | `index.bs:2095–2135` | PASS |
| EV-038 | “Personalization-to-fingerprinting pipeline” | `index.bs:2087`, `:2138` | **PARTIAL** (risk analysis, no measured data) |
| EV-039 | Callback runs with user identity/credentials/session | `index.bs:1815`, `:2014` | PASS |
| EV-040 | `executeTool` returns `DOMString`; result serialized | `index.bs:748`, `:1225–1240` | PASS |
| EV-041 | Private browsing delegated to user agents | `index.bs:2261–2268` | PASS |
| EV-042 | BFCache tools remain in memory but unavailable | `security-privacy-questionnaire.md:91–92` | PASS |
| EV-043 | Pending disconnection behavior “not yet spec'd” | `security-privacy-questionnaire.md:100` | **PARTIAL** (explicit gap) |
| EV-044 | Opaque origin execution rejects `NotSupportedError` | `index.bs:1110–1114` | PASS |
| EV-045 | Cross-traversable execution rejects `InvalidStateError` | `index.bs:1150–1160` | PASS |
| EV-046 | Issue #227 tracks cross-top-level support | `index.bs:1155` | PASS |
| EV-047 | Continuation token is one-shot, same-origin, same-tab | `continuations-explainer.md:40–180` | **PARTIAL** (proposal) |
| EV-048 | Chromium continuation timeout is 30s | `continuations-explainer.md:150–200` | **PARTIAL** (proposal, implementation-specific) |
| EV-049 | Declarative API removed from spec 2026-10-08 | commit `14ae813`; `declarative-api-explainer.md:1–10` | PASS |
| EV-050 | Schema synthesis is TBD | `declarative-api-explainer.md:180–213` | PASS |
| EV-051 | Service worker JIT install / `self.modelContext` proposed | `docs/service-workers.md:30–120` | **PARTIAL** (proposal) |
| EV-052 | “Lethal Trifecta”; single-origin session recommendation | `docs/service-workers.md:220–274` | **PARTIAL** (proposal) |
| EV-053 | Chrome 149 / Edge 150 OT; Brave 2027-01-05; ChatGPT Desktop; Meta Ray-Ban | `implementation-status.md:1–48` | **PARTIAL** (status document, not runtime) |
| EV-054 | Spec is CG-DRAFT | `index.bs:3` | PASS |
| EV-055 | CI uses spec-prod and fails on warning | `.github/workflows/auto-publish.yml:1–16` | PASS |
| EV-056 | No local behavioral tests; WPT external | repo file list; `index.bs:17` | PASS |
| EV-057 | Spec does not prescribe format exposed to browser agent | `index.bs:1760` | PASS |
| EV-058 | README rejects direct MCP adoption (origins/permissions/DOM/tab lifecycle) | `README.md:469–484` | PASS |

**Counts**: 58 checks · 49 PASS · 9 PARTIAL · 0 OPEN. The 9 partials are EV-026, EV-030, EV-038, EV-043, EV-047, EV-048, EV-051, EV-052, EV-053 — all due to non-normative status, implementation-defined behavior, or an explicit gap in the repository text.

## 6.2 EK coverage matrix

| EK | Evidence | Truth | Coverage | Flow | Abstraction | Counterexample | Epistemic |
|---|---|---|---|---|---|---|---|
| EK-01 | EV-001, EV-002 | PASS | PASS | F-C/F-S | PASS | PASS | PASS |
| EK-02 | EV-003, EV-004 | PASS | PASS | F-C | PASS | PASS | PASS |
| EK-03 | EV-005, EV-006 | PASS | PASS | F-C | PASS | PASS | PASS |
| EK-04 | EV-007, EV-008 | PASS | PASS | F-C/F-S | PASS | PASS | PASS |
| EK-05 | EV-009, EV-010 | PASS | PASS | F-S | PASS | PASS | PASS |
| EK-06 | EV-011, EV-012 | PASS | PASS | F-C/F-S | PASS | PASS | PASS |
| EK-07 | EV-013, EV-014 | PASS | PASS | F-C | PASS | PASS | PASS |
| EK-08 | EV-015, EV-016 | PASS | PASS | F-S/F-C | PASS | PASS | PASS |
| EK-09 | EV-017 | PASS | PASS | F-E | PASS | PASS | PASS |
| EK-10 | EV-018, EV-019 | PASS | PASS | F-A | PASS | PASS | PASS |
| EK-11 | EV-020, EV-021, EV-022 | PASS | PASS | F-A | PASS | PASS | PASS |
| EK-12 | EV-023, EV-024 | PASS | PASS | F-A | PASS | PASS | PASS |
| EK-13 | EV-025, EV-026 | PASS | PASS | F-A/F-E | PASS | PASS | **PARTIAL** (implementation-defined) |
| EK-14 | EV-027, EV-028 | PASS | PASS | F-P | PASS | PASS | PASS |
| EK-15 | EV-029, EV-030 | PASS | PASS | F-P | PASS | PASS | **PARTIAL** (advisory) |
| EK-16 | EV-031, EV-032 | PASS | PASS | F-P | PASS | PASS | PASS |
| EK-17 | EV-033, EV-034 | PASS | PASS | F-P | PASS | PASS | PASS |
| EK-18 | EV-035, EV-036 | PASS | PASS | F-P | PASS | PASS | PASS |
| EK-19 | EV-037, EV-038 | PASS | PASS | F-P | PASS | PASS | **PARTIAL** (risk not measured) |
| EK-20 | EV-039, EV-040 | PASS | PASS | F-A/F-D | PASS | PASS | PASS |
| EK-21 | EV-041 | PASS | PASS | F-P | PASS | PASS | PASS |
| EK-22 | EV-042, EV-043 | PASS | PASS | F-S | PASS | PASS | **PARTIAL** (gap) |
| EK-23 | EV-044 | PASS | PASS | F-A | PASS | PASS | PASS |
| EK-24 | EV-045, EV-046 | PASS | PASS | F-A | PASS | PASS | PASS |
| EK-25 | EV-047, EV-048 | PASS | **PARTIAL** | F-M | PASS | PASS | **PARTIAL** (proposal) |
| EK-26 | EV-049, EV-050 | PASS | PASS | F-C | PASS | PASS | PASS |
| EK-27 | EV-051, EV-052 | PASS | **PARTIAL** | F-M | PASS | PASS | **PARTIAL** (proposal) |
| EK-28 | EV-053, EV-054 | PASS | **PARTIAL** | F-P | PASS | PASS | **PARTIAL** (status) |
| EK-29 | EV-055, EV-056 | PASS | PASS | — | PASS | PASS | PASS |
| EK-30 | EV-057, EV-058 | PASS | PASS | F-A | PASS | PASS | PASS |

Coverage partials: EK-25/27 only explainer; EK-28 only status doc. No EK is unsupported.

## 6.3 Six-dimension validation

### Truth

- 58 checks against repository text.
- 49 exact PASS; 9 PARTIAL due to non-normative status, implementation-defined behavior, or explicit gap.
- 0 statements contradicted by the repository itself after reconciliation.
- One initial error was caught: “five error categories including InvalidStateError” was imprecise; corrected to “5 execution failure reasons → 4 exceptions, plus precondition exceptions.”
- **Score**: 0.91.

### Coverage

- All 30 EK have evidence.
- All 19 KO have derivation facts and evidence.
- All 7 flows have entry/terminal/edges.
- Gaps: no runtime, no WPT execution, no browser trial, no external implementation source.
- **Score**: 0.78.

### Flow

- 7 flows / 55 primary edges.
- Every edge has a condition and file anchor.
- Bypass paths explicitly listed: observation path, same-origin default, `toolactivated` timing, opaque origin, BFCache, continuation token, service worker.
- Weak edge: F-C-07 (queue target task) is specified but scheduling is implementation-defined; marked PASS with note.
- **Score**: 0.84.

### Abstraction

- L1 5 / L2 6 / L3 5 / L4 2 / L5 1.
- L4/L5 are Hypotheses/Principle, not Facts.
- No L3 pattern is derived from a single EK; minimum cluster size 3.
- L5 has `cross_project: pending` and confidence 0.62.
- **Score**: 0.82.

### Counterexample

- Every KO has at least one counterexample or boundary condition.
- Strongest counterexamples:
  - same-origin child is exposed by default → breaks “always two-sided”.
  - built-in agent observation sees cross-origin iframe tools → breaks “position alone”.
  - a tool ignoring AbortSignal → breaks “cancel undoes effects”.
  - `readOnlyHint` can be false → breaks “annotation enforces safety”.
- **Score**: 0.86.

### Epistemic

- Fact / Observation / Hypothesis / Pattern / Model / Principle separated.
- Normative text vs explainer vs status doc distinguished.
- No runtime claim.
- No cross-project principle.
- **Score**: 0.90.

## 6.4 Gate decision

| Gate | Result | Reason |
|---|---|---|
| Truth | PASS | 49/58 exact; 9 explicit partials; no unresolved contradiction |
| Coverage | **PARTIAL** | No runtime, no WPT, no browser trial |
| Flow | PASS | 7 flows, 55 edges, conditions and bypasses documented |
| Abstraction | PASS | 19 KO, layers justified, L5 scoped |
| Counterexample | PASS | Every KO has a boundary/counterexample |
| Epistemic | PASS | Status separation clean |

**Overall: PARTIAL.** The package is complete as a specification archaeology; it is not a runtime validation. This is the same gate discipline as the 2026-11-23 `playwright-mcp` run, which was PARTIAL for missing `npm ci` / `npm test` / browser start.

## 6.5 What would move the gate to PASS

1. Run WPT `webmcp` tests against a Chrome 149/Edge 150 build.
2. Read the Chromium/Edge WebMCP implementation source and compare with `index.bs`.
3. Run a two-origin iframe trial to test `exposedTo`/`fromOrigins`/policy.
4. Trigger `AbortSignal` during a slow tool and verify `toolcancel` + `AbortError`.
5. Test BFCache pending behavior.
6. Test built-in agent observation with a cross-origin iframe.
7. Verify the `@w3c/webmcp-polyfill` behavior against the spec.

Until then, every “behavior” statement is `Observation` or `Hypothesis`, not `Fact`.

# 04 · Flow Atlas — 7 flows / 55 edges

> Flows are reconstructed from normative algorithm order, then checked against the actual text.
> Every edge has a source anchor. `condition` is the guard that must hold for the edge to fire.
> `bypass` marks an alternate path that skips the ordinary gate.

## Flow index

| Flow | Type | Entry | Terminal | Edges |
|---|---|---|---|---:|
| F-C | Control | `registerTool()` | caller promise settlement | 8 |
| F-S | State | Document creation | document destroyed | 7 |
| F-D | Data | schema object | serialized result | 6 |
| F-E | Evidence | registration/execution | observable event/observation | 6 |
| F-A | Authority | document origin | invoke / reject | 10 |
| F-M | Memory | registration | loss / continuation | 7 |
| F-P | Policy | threat model | mitigation / residual risk | 8 |

---

## F-C · Control flow: registration → discovery → execution → completion

**Entry**: page calls `document.modelContext.registerTool(tool, options)`.
**Terminal**: `executeTool()` promise resolves with a string or rejects with a mapped `DOMException`.

### Nodes

| Node | State | Source |
|---|---|---|
| N-C1 | `registerTool` entered | `index.bs:811` |
| N-C2 | validation passed; schema serialized | `index.bs:839–885` |
| N-C3 | tool in `tool map` | `index.bs:900–950` |
| N-C4 | `getTools` enumerates descendants | `index.bs:959–1000` |
| N-C5 | `Tool` snapshot returned | `index.bs:1000–1090` |
| N-C6 | `executeTool` validates caller/target | `index.bs:1096–1195` |
| N-C7 | pending execution created | `index.bs:1200–1255` |
| N-C8 | target event loop runs `tool execute steps` | `index.bs:500–560` |
| N-C9 | callback invoked; `toolactivated` | `index.bs:570–640` |
| N-C10 | completion removes pending; resolves/rejects | `index.bs:600–640`, `:1225–1245` |

### Edges

| Edge | From → To | Condition | Source |
|---|---|---|---|
| F-C-01 | N-C1 → N-C2 | document fully active; `tools` allowed; unique/valid name; non-empty description | `index.bs:811–850` |
| F-C-02 | N-C2 → N-C3 | schema serialization succeeds; second fully-active/duplicate check passes | `index.bs:856–885` |
| F-C-03 | N-C3 → N-C4 | `getTools` caller fully active; policy allowed | `index.bs:959–970` |
| F-C-04 | N-C4 → N-C5 | tool is in caller's descendant navigable and exposed | `index.bs:995–1040` |
| F-C-05 | N-C5 → N-C6 | caller passes a `Tool` from `getTools`; target not opaque; same traversable | `index.bs:1096–1160` |
| F-C-06 | N-C6 → N-C7 | tool id still exists; target window alive; target document active | `index.bs:1160–1200` |
| F-C-07 | N-C7 → N-C8 | queue target global task; target event loop available | `index.bs:1248` |
| F-C-08 | N-C8 → N-C9 | input JSON parses to object; target realm creates AbortController | `index.bs:570–600` |
| F-C-09 | N-C9 → N-C10 | callback promise settles or throws | `index.bs:600–640` |
| F-C-10 | N-C10 → terminal | output serializable; error mapped | `index.bs:1225–1245` |

> Note: F-C-01…F-C-10 are 10 edge rows, but the flow is counted as 8 **primary** transitions plus 2 terminal/queue edges. The global edge count uses the 10 rows. See “count reconciliation” at the end.

### Bypass / alternate paths

- **B-C-1 (registration abort)**: if `options.signal` is already aborted, reject before registration (`index.bs:867–870`).
- **B-C-2 (schema throws)**: if `toJSON` throws, reject with that error; nothing enters map (`index.bs:856–862`).
- **B-C-3 (target unload after pending)**: completion with `AbortError` skips callback (`index.bs:410–430`).
- **B-C-4 (caller abort)**: cancel pending; target callback may still run but result discarded (`index.bs:1146–1150`).
- **B-C-5 (tool unregistered after discovery)**: `executeTool` returns `NotFoundError`; no callback (`index.bs:1170–1175`).
- **B-C-6 (same name re-registered)**: old `Tool` still points to old id; `NotFoundError` if old id gone (`index.bs:524–560`).

### Invariants

1. A `Tool` cannot be executed outside the traversable navigable of its `getTools` caller (`index.bs:1150`).
2. A target callback is never invoked before the pending entry exists (`index.bs:1200–1250`).
3. A completion removes the pending entry before resolving the caller (`index.bs:1225`).

### Gaps

- No timeout: a callback that never settles leaves the pending entry until abort/unload.
- No native schema validation: `inputObject` can be any object (`index.bs:578–585`).
- BFCache pending behavior is not fully specified (`security-privacy-questionnaire.md:100`).

---

## F-S · State flow: document, tool, execution

### State model

```
Document: CREATED → FULLY_ACTIVE ⇄ INACTIVE/BFCACHE → DESTROYED
Tool:     UNREGISTERED → REGISTERED → DISCOVERED → PENDING → ACTIVATED → COMPLETED
                                                   ↘ CANCELLED
                                                   ↘ REJECTED
```

### Nodes

| Node | State | Source |
|---|---|---|
| N-S1 | Document created, ModelContext associated | `index.bs:711–737` |
| N-S2 | Tool registered | `index.bs:900–950` |
| N-S3 | Tool discovered (`Tool` snapshot) | `index.bs:1000–1090` |
| N-S4 | Pending execution | `index.bs:1200–1255` |
| N-S5 | Activated | `index.bs:638` |
| N-S6 | Completed / rejected / cancelled | `index.bs:600–640` |
| N-S7 | Document inactive / BFCache | `index.bs:1248`, questionnaire 91–92 |
| N-S8 | Document destroyed / navigated | `index.bs:410–440` |

### Edges

| Edge | From → To | Condition | Source |
|---|---|---|---|
| F-S-01 | N-S1 → N-S2 | `registerTool` succeeds | `index.bs:811–950` |
| F-S-02 | N-S2 → N-S3 | `getTools` returns snapshot | `index.bs:959–1090` |
| F-S-03 | N-S3 → N-S4 | `executeTool` passes checks | `index.bs:1096–1255` |
| F-S-04 | N-S4 → N-S5 | target callback invoked | `index.bs:638` |
| F-S-05 | N-S5 → N-S6 | promise settles / throws / abort | `index.bs:600–640` |
| F-S-06 | N-S6 → N-S2 | re-register after unregister | `index.bs:894–899` |
| F-S-07 | N-S7 → N-S2 | BFCache restore; tools available again | questionnaire 91–92 |
| F-S-08 | N-S8 → N-S2 | recreated document registers again | `index.bs:711–737` |

> 8 rows; 7 primary state transitions plus one restore/recreate. Count uses 8.

### Bypass / alternate paths

- **B-S-1**: `AbortSignal` aborts registration → `UNREGISTERED` without a `toolchange`? It fires `toolchange` (`index.bs:452–470`, `:894–899`).
- **B-S-2**: document goes BFCache while pending → queued task waits; not a terminal state (`index.bs:1248`).
- **B-S-3**: `Tool.source` window GC → `executeTool` `InvalidStateError` before pending is created (`index.bs:1150–1157`).
- **B-S-4**: navigation destroys target → pending completes `AbortError` (`index.bs:410–430`).

### Invariants

1. A tool cannot be registered in a non-fully-active document (`index.bs:811–812`).
2. A `Tool` snapshot does not become valid again after its registration id is gone (`index.bs:524–560`).
3. BFCache preserves registration but not availability (`questionnaire:91–92`).

### Gaps

- Pending execution during BFCache is not explicitly transitioned; spec queues the task but questionnaire says “not yet spec'd” (`index.bs:1248`, `questionnaire:100`).
- No explicit `FAILED_PERMANENTLY` state; every failure maps to a DOMException.

---

## F-D · Data flow: schema and payload serialization

### Nodes

| Node | Data | Source |
|---|---|---|
| N-D1 | `inputSchema` / `outputSchema` object | `index.bs:843–850` |
| N-D2 | serialized JSON string stored in tool definition | `index.bs:900–925` |
| N-D3 | string parsed to object for `Tool` | `index.bs:1040–1070` |
| N-D4 | caller `inputObject` | `index.bs:1096` |
| N-D5 | JSON string passed to target | `index.bs:1115–1128` |
| N-D6 | target realm object | `index.bs:570–585` |
| N-D7 | callback result | `index.bs:610–620` |
| N-D8 | serialized string returned to caller | `index.bs:1225–1240` |

### Edges

| Edge | From → To | Condition | Source |
|---|---|---|---|
| F-D-01 | N-D1 → N-D2 | `toJSON`/serialization succeeds | `index.bs:843–862` |
| F-D-02 | N-D2 → N-D3 | `getTools` creates `Tool` | `index.bs:1040–1070` |
| F-D-03 | N-D4 → N-D5 | `executeTool` called with object | `index.bs:1115–1128` |
| F-D-04 | N-D5 → N-D6 | target realm parses JSON; result is object | `index.bs:578–585` |
| F-D-05 | N-D7 → N-D8 | `JSON.stringify` succeeds | `index.bs:610–625` |
| F-D-06 | N-D7 → error | result is `BigInt`, circular, or otherwise unserializable | `index.bs:620` |

### Bypass / alternate paths

- **B-D-1**: `inputObject` omitted → treated as `undefined` → `null`? The callback receives `undefined`/null; source says if omitted, the tool decides (`index.bs:790`).
- **B-D-2**: output is `undefined` → serialization fails? `JSON.stringify(undefined)` returns `undefined`, not a string → `OperationError` (`index.bs:620`).
- **B-D-3**: `inputSchema` empty string → `Tool.inputSchema` null? The spec says empty string maps to null in `Tool` (`index.bs:234–238`).
- **B-D-4**: schema is a getter with side effects → only `toJSON` path is rechecked; WebIDL conversion may run earlier (`index.bs:843–856`).

### Invariants

1. Caller never receives a live target-realm object; only a string (`index.bs:748`, `:1225`).
2. Target never receives the caller's object identity; it parses a JSON copy (`index.bs:578–585`).
3. No schema semantic validation occurs on either side (`README.md:524`).

### Gaps

- “Schema-conforming arguments” in the questionnaire is not enforced by the algorithm (`security-privacy-questionnaire.md:49` vs `index.bs:578`).
- No native output validation; `outputSchema` is informational (`index.bs:1318–1325`).

---

## F-E · Evidence flow: events and observation

### Nodes

| Node | Evidence | Source |
|---|---|---|
| N-E1 | `toolchange` event | `index.bs:452–470` |
| N-E2 | `Tool` snapshot fields | `index.bs:1420–1470` |
| N-E3 | `toolactivated` event | `index.bs:638` |
| N-E4 | `toolcancel` event | `index.bs:393` |
| N-E5 | console warning on tool rejection | `index.bs:624` |
| N-E6 | browser agent `observation` | `index.bs:1682–1760` |

### Edges

| Edge | From → To | Condition | Source |
|---|---|---|---|
| F-E-01 | registration → N-E1 | tool added/removed; notify descendant documents | `index.bs:452–470` |
| F-E-02 | N-E1 → N-E2 | consumer calls `getTools` | `index.bs:959–1090` |
| F-E-03 | callback invoked → N-E3 | execute callback started | `index.bs:638` |
| F-E-04 | cancellation → N-E4 | caller abort or caller document destroyed | `index.bs:393` |
| F-E-05 | tool rejection → N-E5 | promise rejects with an error | `index.bs:624` |
| F-E-06 | observation → N-E6 | browser agent calls `perform an observation` | `index.bs:1724–1760` |

### Bypass / alternate paths

- **B-E-1**: `toolactivated` is **after** callback invocation, so it cannot be used as a pre-execution gate (`index.bs:638`).
- **B-E-2**: target unload does not fire `toolcancel` at target; only caller-side cancel does (`index.bs:410–440`, `:393`).
- **B-E-3**: no user-visible confirmation event in the spec; `consequentialHint` is not an event (`index.bs:2313`).
- **B-E-4**: built-in agent observation is not an event; it is a pull snapshot (`index.bs:1682–1760`).

### Invariants

1. Events carry only `toolName`; no input/result/origin (`index.bs:1604–1647`).
2. `toolchange` notification order is parent-before-child within one registration, but timers can interleave (`index.bs:470–490`).
3. `toolactivated` is evidence of invocation, not authorization.

### Gaps

- No audit log, no invocation ID, no origin in events.
- No normative user-facing record of which agent invoked which tool.

---

## F-A · Authority flow: who can see/invoke

### Nodes

| Node | Authority check | Source |
|---|---|---|
| N-A1 | Document origin | `index.bs:2143–2159` |
| N-A2 | `tools` Permissions Policy | `index.bs:1647–1655` |
| N-A3 | provider `exposedTo` | `index.bs:872–885` |
| N-A4 | consumer `fromOrigins` | `index.bs:974–985` |
| N-A5 | `tool is exposed to an origin` | `index.bs:498–510` |
| N-A6 | descendant navigable enumeration | `index.bs:995` |
| N-A7 | same traversable navigable | `index.bs:1150–1160` |
| N-A8 | opaque origin rejection | `index.bs:1110–1114` |
| N-A9 | built-in agent observation | `index.bs:1724–1760` |
| N-A10 | ambient credentials | `index.bs:1815`, `:2014` |

### Edges

| Edge | From → To | Condition | Source |
|---|---|---|---|
| F-A-01 | N-A1 → N-A2 | same-origin default; policy self | `index.bs:2159–2165` |
| F-A-02 | N-A2 → N-A3 | `allow="tools"` for cross-origin frame | `index.bs:2168–2177` |
| F-A-03 | N-A3 → N-A4 | provider lists consumer origin | `index.bs:2178–2188` |
| F-A-04 | N-A4 → N-A5 | consumer lists provider origin | `index.bs:2189–2198` |
| F-A-05 | N-A5 → N-A6 | same-origin OR exposed origin | `index.bs:498–510` |
| F-A-06 | N-A6 → N-A7 | target in caller's inclusive descendants | `index.bs:995` |
| F-A-07 | N-A7 → N-A8 | target origin is not opaque | `index.bs:1110–1114` |
| F-A-08 | N-A8 → invoke | all checks pass; tool id exists | `index.bs:1160–1200` |
| F-A-09 | N-A9 → observation | browser agent path; no `fromOrigins`/`exposedTo` filter | `index.bs:1724–1745` |
| F-A-10 | N-A10 → invoke | page session credentials available | `index.bs:1815`, `:2014` |

### Bypass / alternate paths

- **B-A-1 (built-in agent bypass)**: observation algorithm collects all descendant tools with origin and does not apply `exposedTo`/`fromOrigins` (`index.bs:1724–1745`). This is the biggest authority bypass in the text.
- **B-A-2 (same-origin default)**: same-origin descendants need no `exposedTo`/`fromOrigins`; only policy (`index.bs:2159–2165`).
- **B-A-3 (parent→child only)**: child cannot see parent, even same-origin (`index.bs:995`).
- **B-A-4 (opaque origin)**: sandboxed/data URL target rejected (`index.bs:1110–1114`).
- **B-A-5 (no consent)**: `consequentialHint` does not create a mandatory user gate (`index.bs:2313`).
- **B-A-6 (extension agents)**: not specified; no normative path.

### Invariants

1. Cross-origin requires all three gates for the in-page path (`index.bs:2159–2210`).
2. A child cannot widen authority upward (`index.bs:995`).
3. Ambient credentials are available to the callback; the protocol adds no separate auth (`index.bs:1815`).

### Gaps

- Built-in agent filtering is implementation-defined; the spec does not guarantee `exposedTo` applies (`index.bs:1692`).
- No per-tool user consent, no agent identity, no rate limit.
- README's “top-level default exposed, iframe default not exposed” is a **running idea**, not normative (`README.md:520`).

---

## F-M · Memory flow: what persists, what is lost

### Nodes

| Node | Memory | Source |
|---|---|---|
| N-M1 | tool registration in `tool map` | `index.bs:176–190` |
| N-M2 | `Tool` snapshot in caller | `index.bs:1420–1470` |
| N-M3 | pending execution in traversable map | `index.bs:323–350` |
| N-M4 | BFCache-retained registrations | questionnaire 91–92 |
| N-M5 | continuation token state | `continuations-explainer.md:1–312` |
| N-M6 | service-worker tool registry | `docs/service-workers.md:1–274` |
| N-M7 | agent context / cross-site personalization | `index.bs:2073–2143` |

### Edges

| Edge | From → To | Condition | Source |
|---|---|---|---|
| F-M-01 | N-M1 → N-M2 | `getTools` snapshot | `index.bs:1000–1090` |
| F-M-02 | N-M1 → N-M3 | `executeTool` creates pending | `index.bs:1200–1255` |
| F-M-03 | N-M1 → N-M4 | document enters BFCache | questionnaire 91–92 |
| F-M-04 | N-M3 → lost | completion/cancel removes entry | `index.bs:356–395`, `:1225` |
| F-M-05 | N-M3 → N-M5 | continuation token across navigation | `continuations-explainer.md:40–120` |
| F-M-06 | N-M1 → N-M6 | service worker registers by origin/scope | `docs/service-workers.md:30–120` |
| F-M-07 | N-M7 → N-M1 | agent fills over-parameterized inputs | `index.bs:2073–2143` |

### Bypass / alternate paths

- **B-M-1**: `Tool` snapshot does not survive unregister; executing it returns `NotFoundError` (`index.bs:524–560`).
- **B-M-2**: navigation destroys `Tool.source`; snapshot remains but unusable (`index.bs:1510–1535`).
- **B-M-3**: continuation token is one-shot; second redemption fails (`continuations-explainer.md:150–190`).
- **B-M-4**: service worker tools are not page tools; routing by tab/session is TBD (`docs/service-workers.md:120–200`).

### Invariants

1. Registration lifetime is document lifetime; no persistent storage (`index.bs:711–737`).
2. Pending execution is removed on completion/cancel (`index.bs:356–395`).
3. `Tool` identity is registration-specific; it does not survive re-registration (`index.bs:524–560`).

### Gaps

- BFCache pending behavior not fully spec'd (`questionnaire:100`).
- No cross-session persistence; each page load re-registers.
- Continuation and service-worker proposals are not normative.

---

## F-P · Policy flow: threat model → mitigation → residual risk

### Nodes

| Node | Policy element | Source |
|---|---|---|
| N-P1 | threat model: injection, misrepresentation, privacy | `index.bs:1836–2143` |
| N-P2 | annotations as signals | `index.bs:1263–1355` |
| N-P3 | Permissions Policy disable | `index.bs:2268–2283` |
| N-P4 | input name limit / no description limit | `index.bs:2291–2298` |
| N-P5 | shared attack eval dataset (issue #106) | `index.bs:2291–2298` |
| N-P6 | agent-level guardrails | `index.bs:2315–2330` |
| N-P7 | implementation status / origin trials | `implementation-status.md` |
| N-P8 | open issues | `README.md:512–535` |

### Edges

| Edge | From → To | Condition | Source |
|---|---|---|---|
| F-P-01 | N-P1 → N-P2 | tool metadata exists | `index.bs:1826–1830` |
| F-P-02 | N-P1 → N-P3 | site wants to disable | `index.bs:2268–2283` |
| F-P-03 | N-P1 → N-P4 | name length exists; description length does not | `index.bs:2291–2298` |
| F-P-04 | N-P1 → N-P5 | shared eval proposal | `index.bs:2291–2298` |
| F-P-05 | N-P1 → N-P6 | agent chooses guardrail | `index.bs:2315–2330` |
| F-P-06 | N-P6 → residual risk | guardrails probabilistic | `index.bs:2315–2330` |
| F-P-07 | N-P7 → N-P8 | trials reveal gaps | `implementation-status.md`, `README.md:512–535` |
| F-P-08 | N-P8 → N-P1 | open issues feed threat model | `README.md:512–535` |

### Bypass / alternate paths

- **B-P-1**: a site can omit `Permissions-Policy` and rely on default `self`; same-origin tools still available (`index.bs:1647–1655`).
- **B-P-2**: a site can set all annotations false; no enforcement (`index.bs:1826–1830`).
- **B-P-3**: a site can declare `readOnlyHint: true` and perform a write; agent may skip confirmation (`questionnaire:63`).
- **B-P-4**: description length is unbounded; injection payload can be large (`index.bs:2291–2298`).
- **B-P-5**: `consequentialHint` can be omitted or lied about (`index.bs:2313`).

### Invariants

1. Structural isolation is browser-enforced; semantic safety is not (`index.bs:1798–1830`).
2. Every mitigation is either structural (policy/origin) or advisory (annotation/guardrail).
3. Residual risk is acknowledged, not eliminated.

### Gaps

- No normative requirement for confirmation, authentication, or rate limits.
- No standard for agent guardrails.
- No shared attack eval yet (issue #106 open).

---

## 4.1 Edge count reconciliation

| Flow | Rows | Primary transitions | Notes |
|---|---:|---:|---|
| F-C | 10 | 8 | 2 terminal/queue rows |
| F-S | 8 | 7 | 1 restore row |
| F-D | 6 | 6 | — |
| F-E | 6 | 6 | — |
| F-A | 10 | 10 | — |
| F-M | 7 | 7 | — |
| F-P | 8 | 8 | — |
| **Total** | **55** | **55** | All edge rows counted |

## 4.2 Flow-level contradictions / gaps

| ID | Type | Finding | Evidence |
|---|---|---|---|
| FL-G1 | Contradiction | README says built-in agent default exposure is “top-level yes, iframe no”; observation algorithm applies no such filter | `README.md:520` vs `index.bs:1724–1745` |
| FL-G2 | Gap | Pending execution during BFCache not fully specified | `questionnaire:100` |
| FL-G3 | Gap | No native schema validation; “schema-conforming” in questionnaire is not enforced | `index.bs:578–585` vs `questionnaire:49` |
| FL-G4 | Bypass | `toolactivated` fires after callback, so it cannot gate execution | `index.bs:638` |
| FL-G5 | Bypass | Built-in agent observation bypasses `exposedTo`/`fromOrigins` | `index.bs:1724–1745` |
| FL-G6 | Gap | No per-tool consent or agent identity | `index.bs:2313` |
| FL-G7 | Contradiction | `security-privacy-questionnaire` says “does not expose new information to origins”, but over-parameterization sends user data to origins | `questionnaire:5` vs `index.bs:2073–2143` |
| FL-G8 | Gap | `outputSchema` is informational only | `index.bs:1318–1325` |

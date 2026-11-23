# 03 · Knowledge Layer — L1→L5

> 19 Knowledge Objects: 5 L1 · 6 L2 · 5 L3 · 2 L4 · 1 L5.
> Promotion rule: a KO only rises when the evidence supports the higher abstraction, not because the sentence sounds better.
> `[SPEC]` = normative; `[DOC]` = design intent; `[META]` = repository/status.

## 3.1 Layer table

| ID | Level | Type | Epistemic | Confidence | One-line claim |
|---|---|---|---|---|---|
| KO-L1-01 | L1 | Fact | Fact | 0.95 | Tools are per-document registrations keyed by an internal id |
| KO-L1-02 | L1 | Fact | Fact | 0.93 | Execution is a browser-process ledger with cooperative cancellation |
| KO-L1-03 | L1 | Fact | Fact | 0.94 | Cross-origin access needs policy + provider + consumer gates |
| KO-L1-04 | L1 | Fact | Fact | 0.90 | Annotations are author-supplied hints, default false |
| KO-L1-05 | L1 | Fact | Fact | 0.92 | Cross-origin errors are normalized to five DOMException types |
| KO-L2-01 | L2 | Mechanism | Fact | 0.90 | Registration revalidates after schema serialization to close reentrancy |
| KO-L2-02 | L2 | Mechanism | Fact | 0.88 | Two maps split browser-process authority from event-loop-local cancellation |
| KO-L2-03 | L2 | Mechanism | Fact | 0.88 | Discovery is descendant-only, creating a parent→child authority asymmetry |
| KO-L2-04 | L2 | Mechanism | Fact | 0.86 | Built-in agent observation is a separate implementation-defined path |
| KO-L2-05 | L2 | Mechanism | Fact | 0.90 | The spec intentionally delegates semantic safety to agent/UA policy |
| KO-L2-06 | L2 | Mechanism | Fact | 0.88 | Metadata and schemas are untrusted, unvalidated inputs |
| KO-L3-01 | L3 | Pattern | Hypothesis | 0.80 | Two-sided opt-in is a reusable cross-origin tool exposure pattern |
| KO-L3-02 | L3 | Pattern | Hypothesis | 0.82 | Snapshot + id + revalidation is a reusable registry TOCTOU pattern |
| KO-L3-03 | L3 | Pattern | Hypothesis | 0.80 | Platform error funneling is a reusable cross-trust-boundary pattern |
| KO-L3-04 | L3 | Pattern | Hypothesis | 0.78 | Capability metadata is advisory, not enforceable |
| KO-L3-05 | L3 | Pattern | Hypothesis | 0.74 | Implementation-first standardization leaves normative semantic gaps |
| KO-L4-01 | L4 | Model | Hypothesis | 0.70 | The protocol defines the envelope, not the intent |
| KO-L4-02 | L4 | Model | Hypothesis | 0.62 | In-page authority is positional; built-in agent is a separate path |
| KO-L5-01 | L5 | Principle | Principle | 0.62 | Treat page-declared tool metadata as untrusted; enforce consequences outside the protocol |

---

## KO-L1-01 · Per-document registration identity

- **level**: L1
- **type**: Fact
- **epistemic_status**: Fact
- **confidence**: 0.95
- **claim**: WebMCP tools are per-`Document` registrations identified by an internal `tool id`; name is a display/discovery key, not the execution identity.
- **abstraction_rationale**: Directly stated by the `ModelContext` and `Tool` definitions; no inference beyond the text.
- **derivation**: facts EK-01, EK-02, EK-03, EK-04
- **aggregation_rule**: R4 (thematic: registry identity)
- **scope**: Imperative WebMCP tools in a document. Does not cover declarative forms or service-worker tools.
- **counterexample**: A page can register the same name in two different documents; they are distinct tools. A page cannot register two tools with the same name in one document.
- **evidence**: EV-001, EV-003, EV-005, EV-007
- **source**: `[SPEC] index.bs:176–250`, `:711–950`, `:1420–1470`

## KO-L1-02 · Execution ledger and cooperative cancellation

- **level**: L1
- **type**: Fact
- **epistemic_status**: Fact
- **confidence**: 0.93
- **claim**: A tool invocation becomes a pending execution in a traversable-navigable map; cancellation removes it and aborts the target callback's signal, but cannot undo side effects already performed.
- **abstraction_rationale**: Direct algorithm description; cancellation semantics explicitly “cooperative”.
- **derivation**: facts EK-06, EK-07, EK-08, EK-09
- **aggregation_rule**: R2 (causal chain: invoke → pending → cancel/completion)
- **scope**: Same traversable navigable. Not continuation tokens.
- **counterexample**: A tool that ignores its `AbortSignal` and performs a fetch/charge will still have performed it after the caller sees `AbortError`.
- **evidence**: EV-011, EV-013, EV-015, EV-017
- **source**: `[SPEC] index.bs:323–440`, `:600–640`, `:1140–1255`

## KO-L1-03 · Three-gate cross-origin exposure

- **level**: L1
- **type**: Fact
- **epistemic_status**: Fact
- **confidence**: 0.94
- **claim**: Cross-origin tool access requires (1) `tools` Permissions Policy delegation, (2) provider `exposedTo`, and (3) consumer `fromOrigins`; same-origin is the default.
- **abstraction_rationale**: The security section calls these “three independent access-control gates”; direct quote-level support.
- **derivation**: facts EK-10, EK-11, EK-12
- **aggregation_rule**: R3 (structural invariant: all three required)
- **scope**: In-page agents in the same traversable navigable. The built-in agent observation path is not covered by this KO.
- **counterexample**: A cross-origin iframe with `allow="tools"` and matching `fromOrigins` still sees nothing if the provider did not list its origin in `exposedTo`.
- **evidence**: EV-018, EV-020, EV-021, EV-023
- **source**: `[SPEC] index.bs:1647–1655`, `:872–885`, `:974–985`, `:2159–2210`

## KO-L1-04 · Advisory annotations

- **level**: L1
- **type**: Fact
- **epistemic_status**: Fact
- **confidence**: 0.90
- **claim**: `readOnlyHint`, `untrustedContentHint`, `consequentialHint`, and `debugging` are booleans defaulting to false; they are author-supplied signals, not browser-enforced policy.
- **abstraction_rationale**: Direct dictionary definition and security-section statement.
- **derivation**: facts EK-14, EK-15
- **aggregation_rule**: R4 (thematic: semantic hints)
- **scope**: Imperative tools. Declarative annotations are not specified.
- **counterexample**: A malicious page can set all hints false while performing a purchase; the browser does not verify them.
- **evidence**: EV-027, EV-029
- **source**: `[SPEC] index.bs:1263–1355`, `:1826–1830`, `:2313`

## KO-L1-05 · Error normalization across trust boundaries

- **level**: L1
- **type**: Fact
- **epistemic_status**: Fact
- **confidence**: 0.92
- **claim**: The tool-execution error table exposes five platform failure reasons mapped to four exception names (`NotFoundError`, `DataError`, `OperationError`, `AbortError`); API preconditions add `InvalidStateError`, `NotAllowedError`, `NotSupportedError`, `SecurityError`, and `TypeError`. Raw target exceptions and stacks never cross the boundary.
- **abstraction_rationale**: Direct error table and completion mapping; the precondition exceptions are stated at each API entry.
- **derivation**: facts EK-07, EK-08, EK-20
- **aggregation_rule**: R3 (invariant: no raw exception crosses)
- **scope**: Imperative execution. Same-origin callers also receive mapped errors.
- **counterexample**: A tool can still return an error message **inside** a successful JSON result; the protocol does not prevent semantic error leakage.
- **evidence**: EV-013, EV-014, EV-039
- **source**: `[SPEC] index.bs:280–320`, `:600–640`, `:1096–1114`, `:1225–1245`

## KO-L2-01 · Reentrancy-safe registration

- **level**: L2
- **type**: Mechanism
- **epistemic_status**: Fact
- **confidence**: 0.90
- **claim**: `registerTool()` checks fully-active and duplicate-name before and after schema serialization because `toJSON()` can run script that mutates the document or registry.
- **abstraction_rationale**: The mechanism is explicit in the spec note; it explains **why** the second check exists.
- **derivation**: facts EK-02, EK-03, EK-04, EK-05
- **aggregation_rule**: R2 (causal: script during conversion → revalidate)
- **scope**: Any registration whose schema can execute script. Does not cover property getters outside WebIDL conversion.
- **counterexample**: A schema whose `toJSON` only returns a plain object does not trigger a meaningful second check; the check is still executed but is harmless.
- **evidence**: EV-003, EV-005, EV-006
- **source**: `[SPEC] index.bs:843–885`

## KO-L2-02 · Browser-process ledger split

- **level**: L2
- **type**: Mechanism
- **epistemic_status**: Fact
- **confidence**: 0.88
- **claim**: The traversable-navigable pending map models browser-process authority; the per-`ModelContext` local `AbortController` map models event-loop-local cancellation. The split prevents a renderer-local controller from claiming browser-process scope.
- **abstraction_rationale**: The spec explicitly says the traversable map is mutated “in parallel” and the local map is “local”; the design rationale is inferable from the two mutations.
- **derivation**: facts EK-06, EK-07, EK-08, EK-22
- **aggregation_rule**: R3 (structural invariant: authority vs cancellation are separate)
- **scope**: Implementation model in the spec. Does not assert the actual browser process architecture.
- **counterexample**: A browser could implement both maps in one process and still satisfy the observable algorithms; the split is a specification device, not a physical requirement.
- **evidence**: EV-011, EV-012, EV-015
- **source**: `[SPEC] index.bs:185–260`, `:323–395`

## KO-L2-03 · Descendant-only authority asymmetry

- **level**: L2
- **type**: Mechanism
- **epistemic_status**: Fact
- **confidence**: 0.88
- **claim**: `getTools()` walks only the caller's inclusive descendants, so authority flows downward in the frame tree; a child cannot discover a parent's tools, and siblings cannot see each other.
- **abstraction_rationale**: Direct consequence of the traversal start point; explains the security shape.
- **derivation**: facts EK-12, EK-13, EK-24
- **aggregation_rule**: R3 (structural invariant: tree direction)
- **scope**: In-page API. Built-in agent observation starts at top-level and sees all descendants, so it is an exception.
- **counterexample**: A top-level page can call a same-origin child's tool; the child cannot call the top-level page's tool. This is intentional, not a bug.
- **evidence**: EV-023, EV-024, EV-025
- **source**: `[SPEC] index.bs:995`, `:1150–1160`

## KO-L2-04 · Two agent paths

- **level**: L2
- **type**: Mechanism
- **epistemic_status**: Fact
- **confidence**: 0.86
- **claim**: The in-page API (`getTools`/`executeTool`) and the built-in browser agent (`perform an observation`) are separate authority paths; the first is fully specified and filterable, the second is non-normative and implementation-defined.
- **abstraction_rationale**: The spec places `perform an observation` in a non-normative section and explicitly says exposure to the browser agent is implementation-defined.
- **derivation**: facts EK-13, EK-30, EK-28
- **aggregation_rule**: R3 (structural invariant: two paths)
- **scope**: Browser-integrated agents. Author-provided in-page agents use the first path.
- **counterexample**: An extension agent may use neither path; the spec does not define extension agents.
- **evidence**: EV-025, EV-026, EV-057
- **source**: `[SPEC] index.bs:1653–1770`, `:1760`

## KO-L2-05 · Protocol-not-policy

- **level**: L2
- **type**: Mechanism
- **epistemic_status**: Fact
- **confidence**: 0.90
- **claim**: The spec deliberately defines structural boundaries (origins, policy, exposure, error mapping) and refuses to mandate semantic safeguards (intent verification, confirmation, authentication, rate limits).
- **abstraction_rationale**: The security section states this limitation explicitly; multiple design choices are consistent with it.
- **derivation**: facts EK-16, EK-17, EK-18, EK-19, EK-21
- **aggregation_rule**: R4 (thematic: division of responsibility)
- **scope**: Whole spec. Not a claim that agents are unsafe; a claim about where enforcement lives.
- **counterexample**: A browser could add its own mandatory confirmation UI; the spec would not forbid it. The absence is a floor, not a ceiling.
- **evidence**: EV-031, EV-032, EV-035, EV-037
- **source**: `[SPEC] index.bs:1798–1830`, `:2268–2330`

## KO-L2-06 · Untrusted, unvalidated metadata

- **level**: L2
- **type**: Mechanism
- **epistemic_status**: Fact
- **confidence**: 0.88
- **claim**: Tool names, descriptions, schemas, and outputs are page-controlled strings/objects; the browser serializes and copies them but does not validate schema semantics or sanitize natural language.
- **abstraction_rationale**: The registration algorithm only checks serializability; the security section treats metadata as untrusted; open question #92 tracks native validation.
- **derivation**: facts EK-14, EK-15, EK-20
- **aggregation_rule**: R3 (invariant: no semantic validation)
- **scope**: Imperative tools. Output schema is optional and not enforced.
- **counterexample**: A schema can declare `type: "string"` while the callback accepts a number; the browser will not reject the call. The tool must validate.
- **evidence**: EV-027, EV-029, EV-030, EV-050
- **source**: `[SPEC] index.bs:1313–1325`, `:1777–1840`; `[DOC] README.md:524`

## KO-L3-01 · Two-sided opt-in for cross-origin tool exposure

- **level**: L3
- **type**: Pattern
- **epistemic_status**: Hypothesis
- **confidence**: 0.80
- **claim**: When a capability provider and a capability consumer live in different origins, require independent opt-in on both sides plus embedder delegation; this prevents either side from unilaterally widening authority.
- **abstraction_rationale**: The pattern is visible in the three gates and is stated as “two-sided opt-in” in the spec; it generalizes beyond WebMCP, but only one project supports it.
- **derivation**: facts EK-10, EK-11, EK-12, EK-20, EK-23
- **aggregation_rule**: R3 (structural invariant: no unilateral widening)
- **scope**: Cross-origin capability exposure in a frame tree. Does not automatically apply to server-to-server MCP or signed manifests.
- **counterexample**: A same-origin child is exposed by default, so the pattern is not “always two-sided”; it is specifically for cross-origin.
- **evidence**: EV-018, EV-020, EV-021, EV-022
- **source**: `[SPEC] index.bs:2159–2210`

## KO-L3-02 · Snapshot + id + revalidation

- **level**: L3
- **type**: Pattern
- **epistemic_status**: Hypothesis
- **confidence**: 0.82
- **claim**: A registry that hands out executable handles should combine a registration snapshot, an internal identity, and post-conversion revalidation to prevent time-of-check/time-of-use substitution.
- **abstraction_rationale**: Three independent mechanisms in the same registry point to one reusable design pattern; only WebMCP evidence is available, so Hypothesis.
- **derivation**: facts EK-02, EK-03, EK-04, EK-05
- **aggregation_rule**: R1 (mechanism: same causal chain)
- **scope**: Tool/callback registries with mutable registration and script-executing serialization. Not all registries need all three parts.
- **counterexample**: A registry with immutable registrations and no script execution during conversion does not need the post-conversion recheck; the pattern is conditional.
- **evidence**: EV-003, EV-005, EV-007, EV-009
- **source**: `[SPEC] index.bs:843–885`, `:1420–1470`

## KO-L3-03 · Platform error funnel across trust boundaries

- **level**: L3
- **type**: Pattern
- **epistemic_status**: Hypothesis
- **confidence**: 0.80
- **claim**: When invoking code across an origin boundary, map all provider failures onto a small platform-defined error taxonomy and keep raw exceptions, stacks, and provider-defined codes inside the provider realm.
- **abstraction_rationale**: The error table and completion mapping are consistent across multiple paths; generalizable to other cross-origin RPC designs.
- **derivation**: facts EK-07, EK-08, EK-20
- **aggregation_rule**: R1 (mechanism: completion → error mapping)
- **scope**: Cross-origin invocation. Does not prevent semantic error text inside a success payload.
- **counterexample**: gRPC-style status details intentionally carry provider codes; this pattern trades diagnosability for isolation.
- **evidence**: EV-013, EV-014, EV-039
- **source**: `[SPEC] index.bs:280–320`, `:600–640`, `:1225–1245`

## KO-L3-04 · Advisory capability metadata

- **level**: L3
- **type**: Pattern
- **epistemic_status**: Hypothesis
- **confidence**: 0.78
- **claim**: Capability metadata (read-only, untrusted-content, consequential) should be treated as advisory input to a policy engine, never as an enforcement mechanism, because the provider controls the metadata.
- **abstraction_rationale**: The spec says a malicious site can omit/misrepresent hints; the same shape appears in MCP annotations and browser permission prompts.
- **derivation**: facts EK-14, EK-15, EK-16, EK-17, EK-18, EK-19
- **aggregation_rule**: R4 (thematic: metadata vs enforcement)
- **scope**: Agent tool protocols where the tool provider is not fully trusted. Does not apply to signed, verified manifests with trusted publishers.
- **counterexample**: A first-party tool from the same origin as the agent may be trusted enough that its annotations are effectively policy; the pattern is trust-dependent.
- **evidence**: EV-029, EV-031, EV-035
- **source**: `[SPEC] index.bs:1263–1355`, `:1826–1830`, `:2313`

## KO-L3-05 · Implementation-first standardization with normative gaps

- **level**: L3
- **type**: Pattern
- **epistemic_status**: Hypothesis
- **confidence**: 0.74
- **claim**: When browser implementations ship origin trials before a CG draft reaches consensus, the spec tends to normatively encode structural mechanics while leaving semantic policy, validation, and cross-implementation consistency to future work.
- **abstraction_rationale**: The status table, CG-DRAFT status, open issues, and “not yet spec'd” notes point to one pattern; only one project supports it.
- **derivation**: facts EK-28, EK-29, EK-30
- **aggregation_rule**: R2 (causal: adoption pressure → normative scope)
- **scope**: Browser/agent standards with origin trials. Not a universal law of standardization.
- **counterexample**: A spec can be implementation-first and still close semantic gaps before trial; WebMCP has not. The pattern describes a tendency, not a necessity.
- **evidence**: EV-053, EV-055, EV-057
- **source**: `[META] implementation-status.md:1–48`, `[SPEC] index.bs:3`, `README.md:512–535`

## KO-L4-01 · The protocol defines the envelope, not the intent

- **level**: L4
- **type**: Model
- **epistemic_status**: Hypothesis
- **confidence**: 0.70
- **claim**: A page-declared tool protocol can standardize discovery, schema shape, exposure, invocation, and error transport; it cannot standardize whether the declared intent matches the effect, whether the user consented, or whether the output is trustworthy.
- **abstraction_rationale**: This is a compression of the security sections, not a restatement: it explains why so many “gaps” are structural rather than missing features.
- **derivation**: facts EK-14, EK-15, EK-16, EK-17, EK-18, EK-19, EK-20
- **aggregation_rule**: R4 (thematic: envelope vs semantics)
- **scope**: Page-declared tools with an untrusted provider. It may not hold for a closed ecosystem of signed first-party tools.
- **counterexample**: A protocol with mandatory code signing and a verified publisher could move some intent verification into the envelope; WebMCP does not.
- **evidence**: EV-027, EV-031, EV-035, EV-037
- **source**: `[SPEC] index.bs:1777–2333`

## KO-L4-02 · Authority is positional

- **level**: L4
- **type**: Model
- **epistemic_status**: Hypothesis
- **confidence**: 0.68
- **claim**: For **in-page callers**, who can see or invoke a tool is determined by frame-tree position, origin, and policy — not by tool name, global registration, or agent identity. The built-in browser agent is a separate, implementation-defined authority path, not an instance of this positional model.
- **abstraction_rationale**: Combines descendant-only enumeration, three gates, same-traversable restriction, and explicitly excludes the observation path.
- **derivation**: facts EK-10, EK-11, EK-12, EK-13, EK-24
- **aggregation_rule**: R3 (structural invariant: position → authority for in-page path)
- **scope**: In-page agents within a top-level traversable. Built-in browser agents and extension agents are outside the scope of this model.
- **counterexample**: A built-in agent can observe a cross-origin iframe tool that an in-page same-tree caller cannot, so the model does not apply to that path.
- **evidence**: EV-018, EV-020, EV-023, EV-025
- **source**: `[SPEC] index.bs:995`, `:1150–1160`, `:1724–1760`, `:2159–2210`

## KO-L5-01 · Treat page-declared tool metadata as untrusted; enforce consequences outside the protocol

- **level**: L5
- **type**: Principle
- **epistemic_status**: Principle
- **confidence**: 0.62
- **claim**: When an agent consumes tools declared by a web page, it should treat names, descriptions, schemas, annotations, and outputs as untrusted input, and enforce consequential-action policy (confirmation, authorization, rate limits) in the agent or user-agent layer, not rely on the page's declarations.
- **abstraction_rationale**: This is the actionable form of L4-01 plus the threat model; it is scoped to page-declared tools, not all tools.
- **derivation**: facts EK-14, EK-15, EK-16, EK-17, EK-18, EK-19
- **aggregation_rule**: R4 (thematic: untrusted provider → external enforcement)
- **scope**: Browser agents and in-page agents consuming untrusted page tools. Does not apply to a trusted local tool registry or a signed enterprise manifest.
- **counterexample**: A browser that sandboxes every tool in a separate, user-approved capability domain may safely rely on stronger metadata; the principle is a default, not an absolute.
- **evidence**: EV-027, EV-029, EV-031, EV-035, EV-037
- **source**: `[SPEC] index.bs:1777–2333`

---

## 3.2 Promotion audit

| KO | From | To | Rule | Why it holds | What would break it |
|---|---|---|---|---|---|
| KO-L1-01 | EK facts | L1 | direct | Exact definitions | A second registry or global namespace appears |
| KO-L1-02 | EK facts | L1 | direct | Exact algorithm | Cancellation becomes preemptive/transactional |
| KO-L1-03 | EK facts | L1 | direct | Three named gates | A gate is removed or bypassed normatively |
| KO-L1-04 | EK facts | L1 | direct | Dictionary defaults | Annotations become enforced |
| KO-L1-05 | EK facts | L1 | direct | Error table | Raw exceptions become observable |
| KO-L2-01 | EK-02/03/04/05 | L2 | R2 causal | Script can re-enter; recheck exists | Schema serialization becomes side-effect-free |
| KO-L2-02 | EK-06/07/08/22 | L2 | R3 invariant | Two maps with distinct scope | One map suffices observably |
| KO-L2-03 | EK-12/13/24 | L2 | R3 invariant | Traversal starts at caller | Traversal becomes bidirectional |
| KO-L2-04 | EK-13/30/28 | L2 | R3 invariant | Normative vs non-normative split | Observation becomes normative |
| KO-L2-05 | EK-16/17/18/19/21 | L2 | R4 theme | Explicit delegation | Spec adopts mandatory semantic safeguards |
| KO-L2-06 | EK-14/15/20 | L2 | R3 invariant | No schema validation | Native validation lands (issue #92) |
| KO-L3-01 | 5 EK | L3 | R3 invariant | Two-sided opt-in stated | Cross-origin default becomes allow |
| KO-L3-02 | 4 EK | L3 | R1 mechanism | Snapshot+id+recheck | Registrations become immutable |
| KO-L3-03 | 3 EK | L3 | R1 mechanism | Error funnel repeated | Provider errors intentionally exposed |
| KO-L3-04 | 6 EK | L3 | R4 theme | Hints explicitly advisory | Annotations become verified |
| KO-L3-05 | 3 EK | L3 | R2 causal | Adoption vs CG draft | Spec reaches REC before trials |
| KO-L4-01 | 7 EK | L4 | R4 theme | Envelope/semantics split | Intent becomes formally verifiable |
| KO-L4-02 | 5 EK | L4 | R3 invariant | Position determines scope | Agent identity becomes primary |
| KO-L5-01 | 6 EK | L5 | R4 theme | Actionable default | Cross-project evidence contradicts |

**No L4→L5 promotion without a second project.** KO-L5-01 is marked `Principle` but its `cross_project` field is `pending`; it must not be cited as a validated universal law.

## 3.3 What did *not* become knowledge

| Candidate insight | Why not promoted |
|---|---|
| “WebMCP is an MCP transport” | False; spec explicitly does not prescribe browser-agent format (`index.bs:1760`) |
| “Built-in agents see only top-level tools” | README says this is a running idea, not normative; observation algorithm sees all descendants (`README.md:520`, `index.bs:1724`) |
| “Schema-conforming arguments are guaranteed” | No native validation; questionnaire aspiration (`README.md:524`) |
| “`consequentialHint` prevents purchases” | It is a hint; no normative confirmation (`index.bs:2313`) |
| “Continuation tokens are part of WebMCP” | Explainer only; not in `index.bs` |
| “Service worker tools are supported” | Explainer only; not in `index.bs` |
| “Brave ships on 2027-01-05” | Status document says “expected”; not verified |

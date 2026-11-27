# 03 Knowledge Layer — browser-harness 0.1.15 (10 KO)

> Generalized layer: 9 L3 Patterns, 1 L4 Cognitive Model, 0 L5. Each KO is a cluster on the EK graph, not a curated list.
> `derivation.facts` is exactly the cluster EK set. All claims are scoped; cross-project validation is pending unless stated.
> No KO is promoted to L5 because this run has one project plus one downstream comparison, not a cross-domain evidence base.

## Aggregation matrix

| KO | Rule | Cluster EK | Edge types | Abstraction |
|---|---|---|---|---|
| KO-01 | R1 mechanism | 12,14,15,16,17,18,28,29 | mechanism/subsystem | L3 |
| KO-02 | R2 causal chain | 09,10,11,14,18 | causal/constraint | L3 |
| KO-03 | R2 causal chain | 13,20,24,25,26 | causal | L3 |
| KO-04 | R3 invariant | 21,26,49 | constraint | L3 |
| KO-05 | R4 thematic | 11,28,29,44,45 | subsystem/contrast | L3 |
| KO-06 | R1 mechanism | 31,32,33,34,35,36,38 | mechanism/dependency | L3 |
| KO-07 | R3 invariant | 36,37,49,50 | constraint | L3 |
| KO-08 | R1 mechanism | 53,54,55 | mechanism/contrast | L3 |
| KO-09 | R3 invariant | 14,16,19,56 | constraint/contrast | L3 |
| KO-10 | R4 thematic | 07,09,19,58 | subsystem/contrast | L3 |

---

## KO-01 · Tab ownership is a routing contract, not a security boundary (L3 Pattern)

- claim: In a multi-agent browser broker, per-tab ownership can prevent cooperating agents from driving each other's tabs, but it only protects the surfaces it routes; it does not authenticate the owner or cover browser-level capabilities.
- epistemic_status: Validated Pattern (within this repository)
- confidence: high
- derivation.facts: EK-12, EK-14, EK-15, EK-16, EK-17, EK-18, EK-28, EK-29
- aggregation_rule: R1
  - rule: R1 mechanism cluster
  - cluster_eks: [EK-12,EK-14,EK-15,EK-16,EK-17,EK-18,EK-28,EK-29]
  - edges: `mechanism: EK-12↔EK-14↔EK-17`; `subsystem: EK-15/EK-16/EK-18/EK-28/EK-29`
  - naming: every member implements one of the gates that answers "may this caller act on this target/session?"
  - scope_expansion: the members individually describe a guard, a proof, a filter, or a retry; together they describe a broker-level authority surface with a measurable perimeter.
- scope:
  - applies_when: one daemon multiplexes several cooperating callers over one browser; callers can supply a stable owner name.
  - does_not_apply_when: callers are mutually untrusted same-OS-user processes, or the capability is browser-domain rather than tab-targeted.
- evidence: EV-12, EV-14, EV-15, EV-16, EV-17, EV-18, EV-28, EV-29
- counterexample: EK-19 (browser-level session escapes the map); EK-11 (sub-agent can drive parent tab via `cdp()`).
- validation: truth PASS; coverage PASS; abstraction OK; counterexample found (bounded); blind reconstruction performed.
- links: contrast→KO-02, KO-09

## KO-02 · Owner identity is a cooperative label, not an authenticated principal (L3 Pattern)

- claim: When owner identity is taken from environment variables and sent as a request field, the resulting boundary separates well-behaved collaborators but cannot separate an adversary who can set the same variables or speak the same IPC protocol.
- epistemic_status: Validated Pattern (code fact)
- confidence: high
- derivation.facts: EK-09, EK-10, EK-11, EK-14, EK-18
- aggregation_rule: R2
  - rule: R2 causal chain cluster
  - cluster_eks: [EK-09,EK-10,EK-11,EK-14,EK-18]
  - edges: `causal: EK-09→EK-10→EK-11→EK-14→EK-18`
  - naming: the chain is "identity source → inheritance → partial refusal → guarded routing → current-tab pinning."
  - scope_expansion: each EK says one step; together they show the boundary is a chain of voluntary checks with two explicit holes.
- scope:
  - applies_when: same-user agents, host-provided session ids, no adversarial IPC client.
  - does_not_apply_when: a process can write `BH_OWNER`, connect to the 0600 socket, or call `cdp()` directly.
- evidence: EV-09, EV-10, EV-11, EV-14, EV-18
- counterexample: `_refuse_for_shared_owner` wraps only two helpers (EK-11); request owner is unvalidated (`Daemon.handle`).
- validation: truth PASS; abstraction OK; counterexample found; blind reconstruction performed.
- links: dependency→KO-01; contrast→KO-09

## KO-03 · Reclamation is a staged return, not a destructive cleanup (L3 Pattern)

- claim: A long-lived shared browser needs a staged lifecycle: keep the current tab while active, return non-current tabs on idleness, return the last tab after a long forget window, and close only explicitly-created unmarked tabs on an explicit release.
- epistemic_status: Validated Pattern
- confidence: high
- derivation.facts: EK-13, EK-20, EK-24, EK-25, EK-26
- aggregation_rule: R2
  - rule: R2 causal chain
  - cluster_eks: [EK-13,EK-20,EK-24,EK-25,EK-26]
  - edges: `causal: EK-13→EK-20→EK-24→EK-25→EK-26`
  - naming: "create → active → idle → forget → explicit done" is a complete lifecycle.
  - scope_expansion: each EK is one phase; the cluster explains why a single "cleanup" operation would either leak tabs or destroy the person's page.
- scope:
  - applies_when: a broker owns tabs on behalf of ephemeral agents and a human shares the browser.
  - does_not_apply_when: an agent owns a dedicated browser with no human tabs (destructive cleanup is then acceptable).
- evidence: EV-13, EV-20, EV-24, EV-25, EV-26
- counterexample: marked tabs can stay forever (by design); default idle never closes, so storage still accumulates detached tabs.
- validation: truth PASS; abstraction OK; blind reconstruction performed.
- links: dependency→KO-04

## KO-04 · Destructive cleanup must preserve the last shared real page (L3 Pattern)

- claim: Any cleanup that can close tabs must treat the browser's last real page as a conserved resource, independent of which agent created it.
- epistemic_status: Validated Pattern
- confidence: high
- derivation.facts: EK-21, EK-26, EK-49
- aggregation_rule: R3
  - rule: R3 invariant cluster
  - cluster_eks: [EK-21,EK-26,EK-49]
  - edges: `constraint: EK-21→EK-26←EK-49`
  - naming: all three constrain what may be destroyed or abandoned when ownership ends.
  - scope_expansion: from "do not close a marked tab" to "do not leave the browser page-less" is a broader invariant about shared resources.
- scope:
  - applies_when: a human may still be using the browser; agents' tabs share the window.
  - does_not_apply_when: headless/dedicated browser with a known lifecycle owner.
- evidence: EV-21, EV-26, EV-49
- counterexample: `last` counts only `type=="page" && is_real_page` across `self.info`; a stale `info` map could mis-count. Not tested against a real browser in this run.
- validation: truth PASS; abstraction OK; counterexample open (info staleness).
- links: contrast→KO-03

## KO-05 · Typed failures are part of the ownership protocol (L3 Pattern)

- claim: A broker that refuses an action must return a stable, typed error that says what was not sent and what the caller can do next; silent fallback to another tab is a protocol violation.
- epistemic_status: Validated Pattern
- confidence: high
- derivation.facts: EK-11, EK-28, EK-29, EK-44, EK-45
- aggregation_rule: R4
  - rule: R4 thematic cluster
  - cluster_eks: [EK-11,EK-28,EK-29,EK-44,EK-45]
  - edges: `subsystem: EK-28/EK-29/EK-44/EK-45`; `contrast: EK-11`
  - naming: each member defines a terminal state and its caller-visible explanation.
  - scope_expansion: individual errors become a protocol: `tab-taken`, `tab-closed`, `tab-released`, `tab-crashed`, `owner-shared` all preserve "nothing was sent" semantics.
- scope:
  - applies_when: remote/broker execution where the caller cannot inspect the browser directly.
  - does_not_apply_when: local in-process library where exceptions can carry a live object.
- evidence: EV-11, EV-28, EV-29, EV-44, EV-45
- counterexample: `tab-released` is returned only once for `gone`; a later same command may create a new tab and look successful.
- validation: truth PASS; abstraction OK; blind reconstruction performed.
- links: contrast→KO-01

## KO-06 · Verify process identity end-to-end before signaling or reusing it (L3 Pattern)

- claim: A PID file, socket file, or open port is not proof of the process that owns it; before signaling or reusing a daemon, require a live protocol answer plus a self-reported identity plus a start-time/fingerprint match.
- epistemic_status: Validated Pattern
- confidence: high
- derivation.facts: EK-31, EK-32, EK-33, EK-34, EK-35, EK-36, EK-38
- aggregation_rule: R1
  - rule: R1 mechanism cluster
  - cluster_eks: [EK-31,EK-32,EK-33,EK-34,EK-35,EK-36,EK-38]
  - edges: `mechanism: EK-31↔EK-32↔EK-38`; `dependency: EK-33→EK-31`; `constraint: EK-34/EK-35/EK-36`
  - naming: all members answer "is this the daemon I think it is, and may I touch it?"
  - scope_expansion: from a single `identify()` call to a lifecycle policy covering ready, pending, active, remote, and mismatched endpoints.
- scope:
  - applies_when: a supervisor may SIGTERM/reuse a long-lived process by name.
  - does_not_apply_when: PID namespaces or kernel-level process handles are the authority.
- evidence: EV-31, EV-32, EV-33, EV-34, EV-35, EV-36, EV-38
- counterexample: pending daemon with no fingerprint refuses to act (fail-closed); active owners block restart; both are intentional limits.
- validation: truth PASS; abstraction OK; blind reconstruction performed.
- links: contrast→KO-07

## KO-07 · Fail closed when a name is bound to a remote or different browser (L3 Pattern)

- claim: A reusable daemon name must never silently fall back to a different browser after a remote browser ends or an endpoint changes; the correct behavior is an explicit error that preserves the binding until an explicit reset.
- epistemic_status: Validated Pattern
- confidence: high
- derivation.facts: EK-36, EK-37, EK-49, EK-50
- aggregation_rule: R3
  - rule: R3 invariant cluster
  - cluster_eks: [EK-36,EK-37,EK-49,EK-50]
  - edges: `constraint: EK-36→EK-37`; `constraint: EK-49/EK-50→EK-37`
  - naming: all four prevent a silent substitution of the browser behind a name.
  - scope_expansion: endpoint mismatch, remote marker, idle-exit policy, and CDP death are four independent ways a name could drift; together they form one fail-closed invariant.
- scope:
  - applies_when: named daemons are reused across CLI invocations and may outlive their browser.
  - does_not_apply_when: the caller always uses a fresh unique name and no remote browser.
- evidence: EV-36, EV-37, EV-49, EV-50
- counterexample: `restart_daemon` is an explicit reset and can clear the marker; that is intentional, not a bypass.
- validation: truth PASS; abstraction OK.
- links: contrast→KO-06

## KO-08 · Redact at the egress boundary, with per-surface rules (L3 Pattern)

- claim: Output surfaces that leave the agent's machine need their own redaction at the point of egress; a single generic key filter is not equivalent to text scrubbing, and each surface's data shape dictates the rule.
- epistemic_status: Validated Pattern
- confidence: high
- derivation.facts: EK-53, EK-54, EK-55
- aggregation_rule: R1
  - rule: R1 mechanism cluster
  - cluster_eks: [EK-53,EK-54,EK-55]
  - edges: `mechanism: EK-53↔EK-54`; `contrast: EK-55`
  - naming: three egress paths (report text, telemetry properties, HTTP proxy) each redact or avoid egress by a different mechanism.
  - scope_expansion: from "scrub one string" to "classify egress surfaces and apply the right rule at each."
- scope:
  - applies_when: telemetry/feedback/HTTP fetch leave the process.
  - does_not_apply_when: purely local logs with no external sink.
- evidence: EV-53, EV-54, EV-55
- counterexample: `telemetry._safe_properties` only redacts strings containing `://`; other secret-shaped values in a non-URL string are truncated, not scrubbed. Report's scrub is stronger.
- validation: truth PASS; abstraction OK; counterexample found.
- links: contrast→KO-09

## KO-09 · Authority checks must cover the whole capability surface, not the visible one (L3 Pattern)

- claim: A system can enforce ownership on its most visible operations while leaving a higher-authority path unguarded; the correct unit of analysis is the capability surface (every way to reach the resource), not the set of methods that look tab-scoped.
- epistemic_status: Validated Pattern (within this repository); cross-project generalization is Hypothesis C-06, not established
- confidence: medium
- derivation.facts: EK-14, EK-16, EK-19, EK-56
- aggregation_rule: R3
  - rule: R3 invariant cluster
  - cluster_eks: [EK-14,EK-16,EK-19,EK-56]
  - edges: `constraint: EK-14/EK-16→EK-19`; `contrast: EK-56`
  - naming: three tab-scoped guards plus one browser-scoped counterexample define the perimeter gap.
  - scope_expansion: the claim stays inside this broker: authorization must be reasoned over the full reachability graph because the visible tab surface is not the whole capability surface. It is not yet a cross-domain model.
- scope:
  - applies_when: broker/proxy designs where a lower-level API can bypass higher-level guards; browser CDP, container runtimes, database pools.
  - does_not_apply_when: the lower-level API is inaccessible to callers (separate process/namespace with no capability).
- evidence: EV-14, EV-16, EV-19, EV-56
- counterexample: no exploit was executed; the code path is direct and the session id is unguessable, so the practical risk depends on sid exposure. This weakens severity, not the structural claim.
- validation: truth PASS; abstraction OK (L4 argued); counterexample open (exploitability); blind reconstruction performed.
- links: contrast→KO-01, KO-02

## KO-10 · A capability shipped upstream is not a capability in force downstream (L3 Pattern)

- claim: An upstream release can add an ownership model while its main downstream stays pinned to an older version and uses a different concurrency strategy; release notes then describe a capability that is not active in the product users run.
- epistemic_status: Validated Pattern (one upstream/downstream pair)
- confidence: high for the pin and API absence; medium for user impact
- derivation.facts: EK-07, EK-09, EK-19, EK-58
- aggregation_rule: R4
  - rule: R4 thematic cluster
  - cluster_eks: [EK-07,EK-09,EK-19,EK-58]
  - edges: `subsystem: EK-07/EK-09/EK-19`; `contrast: EK-58`
  - naming: the theme is "version seam between a harness capability and its consumer."
  - scope_expansion: from a single pin to a general adoption question: which layer actually executes the ownership check in the shipped product?
- scope:
  - applies_when: a library advertises a new safety/concurrency feature and a downstream pins an older version.
  - does_not_apply_when: downstream vendors the new version or the feature is not safety-relevant.
- evidence: EV-07, EV-09, EV-19, EV-58
- counterexample: browser-use gives each non-shared `Browser` a unique daemon and browser, so it may not need the owner API for those modes. The shared `Browser('chrome')` path is the unresolved case. This bounds the claim: it is an adoption/coverage gap, not "browser-use is unsafe."
- validation: truth PASS; abstraction OK; counterexample found; blind reconstruction performed.
- links: contrast→KO-02

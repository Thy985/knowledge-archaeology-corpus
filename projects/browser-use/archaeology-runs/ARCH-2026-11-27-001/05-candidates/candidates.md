# 05 Candidates — browser-use 1.0.8

Candidates are not conclusions. Each entry states what is known, what is inferred, and what would falsify it. Cross-project candidates stay hypotheses until a second project provides a contrasting case.

## A. Within-project uncertain conclusions

### C-01 — Does the 1.0 rewrite improve or reduce controllability?
**Known**: the action schema is gone; the model writes Python; the control surfaces are budgets, delivery checks, domain rules, and process boundaries (KO-01, KO-03, KO-04, KO-11).
**Hypothesis**: for an aligned model on a trusted task, code-as-action increases task capability and reduces schema-maintenance cost; for an untrusted task, it shifts risk to the execution plane, where this project does not sandbox by default.
**Would confirm**: a benchmark comparing task success and policy violations between 0.13.10 and 1.0.8 on the same tasks, with and without container isolation.
**Would falsify**: code-as-action produces more policy violations even inside a container, or typed-action agents match its success rate on the same benchmark.
**Status**: open; no benchmark evidence in this repository.

### C-02 — Is the judge useful despite not being independent?
**Known**: judge uses the same model, often the same prefix, and sees the agent’s trace; judge failure does not change the run (KO-05).
**Hypothesis**: a same-model judge catches a meaningful fraction of inconsistencies (claimed success vs visible failure) because it reads the trace in a fresh turn, even though it cannot independently re-execute.
**Would confirm**: a labelled set of failed runs where the judge’s `success=False` rate is materially above the agent’s self-report.
**Would falsify**: judge agreement with the agent is near-perfect on a set where an external verifier finds failures.
**Status**: plausible, untested here.

### C-03 — Does the local control-plane exposure matter in practice?
**Known**: loopback DevTools has no authentication; any local account can attach; profile copies carry cookies; the docs recommend container/VM/cloud on shared machines (KO-08).
**Hypothesis**: the exposure is acceptable for single-user developer machines but becomes a real vulnerability on shared CI runners, multi-tenant desktops, or remote dev boxes.
**Would confirm**: a shared-host test where a second UID connects to the debug port and reads cookies.
**Would falsify**: platform-level controls (e.g., per-user network namespaces or Chrome’s own approval) block cross-UID access in the tested environments.
**Status**: security-relevant candidate; no live multi-user test performed.

### C-04 — Is `allowed_domains` safe enough for prompt-injection containment?
**Known**: it blocks main-frame documents and sweeps tabs, but not subresources/iframes/Python HTTP (KO-07).
**Hypothesis**: it is a navigation guard, not a data-exfiltration guard; an injected page can exfiltrate via `fetch()` or an image URL to an attacker host inside an allowed frame.
**Would confirm**: a test page that `fetch()`es an attacker host while the main frame stays allowed, with the request observed.
**Would falsify**: the Guard also blocks subresource requests, or the browser’s network stack is otherwise constrained.
**Status**: high-priority candidate for a security-focused follow-up.

### C-05 — Does the compatibility facade create a false sense of behavioral continuity?
**Known**: constructors/names are preserved; control flow is not; removed options raise (EK-01, EK-52).
**Hypothesis**: users upgrading from 0.x may keep passing options that now warn/ignore (`max_actions_per_step`, `flash_mode`, etc.) and assume the old loop semantics still apply.
**Would confirm**: upgrade tests with real 0.x configurations that show silent behavior changes beyond the documented `REMOVED` list.
**Would falsify**: every ignored option has a visible warning and a migration test.
**Status**: plausible documentation/UX risk.

## B. Cross-project hypotheses (not principles)

### H-01 — Code-as-action shifts governance to the execution plane
**Statement**: as agents move from typed actions to code execution, the effective policy surface moves from action schemas to process/user/filesystem/browser boundaries.
**Support**: browser-use 1.0.8 (KO-01, KO-11); compare a typed-action agent such as a DOM-action framework with a real sandbox.
**Falsifier**: a code-executing agent that keeps a strong action-level policy layer (for example, an instrumented interpreter that mediates every browser call) and achieves the same capability.
**Status**: hypothesis; one project only.

### H-02 — “Redaction” is routinely mistaken for “sandbox”
**Statement**: agent products that redact secrets in outputs often describe themselves as safe, while the model process can still read the secrets; the gap is a recurring design bug.
**Support**: browser-use’s explicit “not a sandbox” language (KO-06, KO-12); compare another agent runtime that claims secret safety.
**Falsifier**: a runtime where redaction and process isolation are the same control, or where users consistently understand the distinction.
**Status**: hypothesis; needs a second codebase and a user-study or docs audit.

### H-03 — Same-model self-judging is a weak verifier
**Statement**: when a judge shares the actor’s model and context prefix, it is a second opinion, not independent verification; its value depends on trace quality and prompt separation.
**Support**: browser-use judge design (KO-05); compare frameworks with external verifiers or execution-based checks.
**Falsifier**: a same-model judge with a separately constructed evidence set outperforms an external verifier on a labelled benchmark.
**Status**: hypothesis; benchmark needed.

### H-04 — Local debug ports are ambient authority in agent runtimes
**Statement**: browser agents that attach to a user’s local Chrome inherit an unauthenticated loopback control plane; the convenience is an authority grant, not a sandbox.
**Support**: browser-use local modes (KO-08); compare other local browser-control tools.
**Falsifier**: a local browser agent that authenticates its DevTools proxy or runs in a per-agent network namespace by default.
**Status**: hypothesis; likely testable across several tools.

### H-05 — Managed cloud default is an OSS distribution strategy
**Statement**: open-source agent frameworks increasingly default to a managed cloud runtime (browser + model + key), with local execution as an explicit alternative; the default shapes both billing and the threat model.
**Support**: browser-use defaults (KO-10); compare a framework that defaults to local and self-hosted models.
**Falsifier**: a project with the same cloud-first defaults but no commercial runtime, or a local-first project with equal adoption.
**Status**: hypothesis; business/architecture, not a safety conclusion.

### H-06 — Delivery ladders are a generic budget pattern
**Statement**: long-running agents benefit from a staged degradation ladder (cell deadline → delivery deadline → delivery-only turn → partial checkpoint) rather than a single timeout.
**Support**: browser-use implementation and tests (KO-03); compare other long-horizon agent runtimes.
**Falsifier**: a runtime with a single hard timeout that matches delivery completeness on long tasks.
**Status**: hypothesis; benchmark needed.

### H-07 — Context compaction has an evidence-retention trade-off
**Statement**: server-side compaction is opaque but cheap; client-side summary checkpoints are inspectable but lossy; the choice affects whether a later delivery can cite earlier evidence.
**Support**: browser-use dual path (KO-09); compare frameworks with retrieval-based memory.
**Falsifier**: an opaque compaction method that preserves all task-relevant evidence, or a checkpoint method that loses no evidence.
**Status**: hypothesis; needs task-level evaluation.

### H-08 — Domain allowlists are navigation policy, not exfiltration policy
**Statement**: allowlisting main-frame hosts does not prevent data exfiltration through subresources or the agent’s own network calls; teams that treat it as a security boundary will be surprised.
**Support**: browser-use Guard scope (KO-07); compare any browser agent with a domain allowlist.
**Falsifier**: a browser agent that mediates all browser and Python network egress through one policy engine.
**Status**: hypothesis; security-relevant, testable.

## Candidate discipline

- None of H-01…H-08 is a validated principle.
- C-01…C-05 are open questions about this project; they are not Knowledge Objects.
- A candidate may be promoted only after a second project provides independent evidence and a counterexample search; until then it stays in this file.

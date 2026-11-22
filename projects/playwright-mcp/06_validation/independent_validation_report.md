# 06 · Independent Audit Report

## Independence statement

A separate read-only pass reconstructed findings from the repository and pinned upstream source **before** the formal package was finalized. It did not use the package as a source of truth. No code was executed, no browser launched, no test run.

Full independent notes: `../../../audit/independent_findings.md` (job-local).

## Verdict statistics — 30 findings

| Verdict | Count | Findings |
|---|---:|---|
| CONFIRMED | 27 | A-01…A-13, A-15…A-19, A-21…A-29 |
| PARTIALLY_CONFIRMED | 1 | A-20 extension token: passed via URL, relay does not compare; extension-side validation unverified |
| CONTRADICTED | 1 | A-14 Docker "any MCP client" vs Host allowlist |
| MISSING | 1 | A-30 no runtime test / browser evidence |
| **Total** | **30** | |

## Claim-level attacks

| Claim | Verdict | Reconciliation |
|---|---|---|
| "a11y snapshot is the architecture" | PARTIALLY_CONFIRMED | Scope to "a11y-first default with vision opt-in"; screenshot is default, coordinate tools are `--caps=vision` |
| "default-open control plane" → L5 | DOWNGRADED | Kept as L4 hypothesis, not principle |
| "convenience guard is not security" → L5 | DOWNGRADED with scope | Kept L5 but scoped to browser/agent tooling; cross-project pending |
| "no vision" | OVER_GENERALIZED | Corrected to "vision not required by default"; screenshot + opt-in coordinate tools exist |
| "extension token authenticates relay" | DOWNGRADED | EK-30 medium; candidate C-04 |
| "`vm` is escapable" | NEEDS_HUMAN_REVIEW | Not tested; treat as RCE-equivalent, do not claim exploit |
| "Docker can be reached by any client" | CONTRADICTED | Gate PARTIAL; candidate C-03 |

## Three strongest successes

1. **Boundary correction**: thin wrapper vs pinned upstream implementation. Prevented false claims from a 44-file repo.
2. **Default-surface reconstruction**: exact 25 tools + RCE/evaluate/network primitives. Highest safety value.
3. **State/default reconstruction**: persistent profile keyed by cwd, factory precedence, isolated/userDataDir exclusion, idle timer.

## Three independent errors / corrections

1. **Docker reachability**: README prose overstates "any MCP client"; Host allowlist rejects external Host by default. **CONTRADICTED.**
2. **Extension token**: initially treated as relay authentication; no server-side comparison in relay. **DOWNGRADED.**
3. **"No vision"**: screenshot is default and coordinate tools are opt-in. **OVER_GENERALIZED → scope tightened.**

## Omissions

- No upstream `tests/mcp` execution; only 9 packaged-server tests.
- No browser/Chrome runtime validation.
- No `npm pack` comparison of `@playwright/mcp@0.0.86` to this tree.
- No proof of `vm` escape.
- No benchmark of token/latency vs browser-use.

## Benchmark case B-01

**Default-open browser control plane.** Minimum evidence checklist:

1. transport authentication/authorization (not just Host allowlist)
2. default tool list; code execution / page JS / credential-reading primitives
3. default profile persistence and workspace derivation
4. attach modes (CDP/extension/remote) and trust implications
5. file/network guard semantics and bypass flags
6. redaction as convenience vs boundary
7. runtime/test evidence for the packaged artifact

This is a candidate benchmark, not a validated universal rubric.

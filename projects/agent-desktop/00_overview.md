# agent-desktop — Project Archaeology Package

- **Project**: agent-desktop (`vercel-labs/agent-desktop`)
- **Repository**: https://github.com/vercel-labs/agent-desktop.git
- **Commit**: `ce2ddf07226346818de2a1c6bae9869cfad142df` (`main`, 2026-11-23 12:34:52 -0600)
- **Tag distance**: `v1.3.1-10-gce2ddf0`; workspace version `1.3.1`
- **Run**: `ARCH-2026-11-25-001` / job `KA-2026-11-25-001`
- **Mode**: `initial`
- **Skill**: knowledge-archaeology v3.2
- **Date**: 2026-11-25 (Asia/Shanghai)
- **Delivery**: local Package + `knowledge-archaeology-corpus` PR. No Feishu write (no explicit authorization this run).

## One-paragraph verdict

`agent-desktop` is not a demo wrapper around an accessibility API. It is a **cross-platform desktop capability runtime** whose central design problem is: *how does an agent claim that a GUI action happened, and what may it safely do when it cannot prove it?* The repository answers with a layered evidence architecture: a platform-neutral `PlatformAdapter` contract, a single compiled command surface projected into CLI, batch, and MCP; an actionability preflight; a five-state delivery lattice; a second-observation postcondition model; and a ref store that distinguishes *observed*, *unique-at-capture*, *stale*, and *unproven*. Its strongest engineering knowledge is not "it supports macOS/Windows/Linux" but **how it refuses to convert unverifiable action outcomes into retryable success**.

## Headline findings

1. **The command surface is one authority, projected three ways.** CLI, batch JSON, and MCP tools all resolve to the same `Commands` enum / clap tree; MCP is generated from clap leaves and re-enters the same binary as a child process. This eliminates registry drift, but concentrates every new command into a known multi-file change set enforced by drift tests.
2. **Delivery is a constrained lattice, not a boolean.** `not_delivered` is the only safe-retry state. Pre-dispatch errors are forced to it; unknown mutation outcomes stay unknown; read-only commands may settle unknown to `not_delivered` because they sent nothing. Retry is derived from delivery, and inconsistent serialized pairs are rejected at deserialization.
3. **The ref system proves identity rather than trusting coordinates.** A ref is `@<snapshot_id>:eN`; bare `@eN` is invalid. Tokenless refs are published only when a whole-window observation proves the full predicate unique; otherwise `identity_unproven` is surfaced and those refs are refused. Predicate fields exclude mutable value and bounds as comparison fields and treat unreadable fields as `Unknown`, not match; label-like roles may still promote a value into the stable-name field.
4. **Snapshot "completeness" is a three-valued operational fact.** A depth-limited snapshot can be `complete: true` while the logical tree is cut at the requested depth; an over-budget read is `complete:false` + `truncated:true` + `nodes_observed`; a failed whole-window proof adds `identity_unproven: true`. These are different failure/coverage states, not synonyms for "timeout".
5. **Platform divergence is named, not hidden.** Observation and semantic refs work on X11 and Wayland; pointer gestures, wheel scrolling, window control, `focus-window`, and headed shell input require X11 on Linux. macOS uses AX, Windows uses UIA, Linux uses AT-SPI. The adapter trait defaults every operation to `NotSupported`, so a platform must opt in explicitly.
6. **Privacy is channel-scoped, and one channel is deliberately unredacted.** Secure fields withhold value and related text; JSONL trace keys are recursively redacted; replay screenshots/refmap artifacts are explicitly unredacted operator-owned artifacts. The boundary is documented rather than accidental.
7. **The system tolerates exactly one kind of unknown in the actionability battery.** `receives_events` with no occlusion evidence is non-blocking; other `Unknown` checks block. This is an explicit exception, not a general "unknown means proceed" policy.

## Package map

| Path | Content |
|---|---|
| `00_overview.md` | This file: verdict, headline findings, quality metrics, limits |
| `01_project-layer/project_layer.md` | Identity, architecture, entrypoints, lifecycle, state, contracts, permissions, tests, extension surface |
| `02_engineering-knowledge/ek_graph.md` | 39 EK nodes with typed links and evidence anchors |
| `03_knowledge-layer/knowledge_layer.md` | 13 KO across L1-L5, each with aggregation rule and EK traceability |
| `04_flow-atlas/flow_atlas.md` | Seven flows (Control/State/Data/Evidence/Authority/Memory/Policy) with anchored edges |
| `05_candidates/candidates.md` | 12 unresolved candidates, including doc-vs-code and authority-edge questions |
| `06_validation/evidence_map.md` | EV-001..EV-046 source ledger |
| `06_validation/independent_audit.md` | Blind reconstruction and adversarial audit (stage 5) |
| `06_validation/reconciliation.md` | Accepted/downgraded/corrected findings and corpus write decision (stage 6) |
| `run_metadata.yaml` | Machine-readable run provenance and metrics |

## Quality metrics (package-level, post-audit)

| Metric | Value | Basis |
|---|---:|---|
| EK nodes | 39 | `02_engineering-knowledge/ek_graph.md` |
| Typed EK links | 118 | count of typed `links` entries in EK Graph |
| EK with ≥1 link | 39/39 (100%) | every EK section declares links |
| Free-floating EK | 0/39 (0%) | every EK is in ≥1 KO cluster |
| KO nodes | 13 | L1=4, L2=4, L3=3, L4=1, L5=1 |
| KO with aggregation rule | 13/13 (100%) | each KO declares R1-R4 |
| Average KO cluster size | 4.0 | 52 EK memberships / 13 KO (all 39 EK covered) |
| Flow edges | 44 | seven flows, each edge anchored to symbol/file |
| Evidence anchors | 46 | `06_validation/evidence_map.md` |
| Candidates | 12 | `05_candidates/candidates.md` |
| Runtime execution | none | no GUI session in this environment |

## What this package does not claim

- It does **not** claim any runtime behavior was executed. No `cargo test`, E2E, click, snapshot, or MCP server was run in this environment. Runtime claims are marked Observation/Hypothesis or are read from code paths and tests.
- It does **not** claim npm download volume. The `~50k weekly downloads` figure from the radar card was not re-verified; `npm view` only confirmed version `1.3.1` and `time.modified 2026-11-13`.
- It does **not** claim the radar card's `1.2.0 / 60 commands / 56 operational` numbers. Repository README and Cargo say `1.3.1`, `61 command names`, `57 operational`; those win.
- It does **not** treat `CONCEPTS.md` prose as implementation fact. Where prose and code diverge, the code is Fact and the prose is a Candidate (CAND-03).
- It does **not** promote a single-project pattern to a validated cross-project Principle. L5 is explicitly a candidate.

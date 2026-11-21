# 05 · Candidates（未确认、跨项目、待验证）— browser-use 1.0.8rc1

> Candidates are not facts and not principles. Each has a status, a reason it stays open, and a test that would promote, downgrade, or kill it. None of these may be cited as validated without the stated confirmation.

## C-01 `1.0.8rc1` 与稳定版的关系未确认

- **Type**: release/version question.
- **Status**: Open — needs external release metadata.
- **Observation**: HEAD is tagged `1.0.8rc1`; `pyproject.toml` version is `1.0.8rc1`; commit subject says “1.0.8rc1”. There is no GA tag at HEAD.
- **Why it matters**: the package must not describe this as a stable 1.0.8. A refresh package can state “1.0.8rc1” but cannot claim GA behavior or a final release date.
- **What would confirm**: GitHub releases/tags showing whether `1.0.8` final exists, and the delta from `1.0.8rc1` to final.
- **What would refute**: no final release, or a final release with a different commit.
- **Evidence**: EV-01, EV-02.

## C-02 `browser-harness 0.1.13` 的 helper 行为是外部权威

- **Type**: dependency boundary.
- **Status**: Open — dependency internals not audited.
- **Observation**: `browser_exec` cells get helpers and `cdp()` from `browser-harness==0.1.13`. The repo pins the version, but the helper implementation, daemon discovery, auth storage, and helper-level navigation checks are outside this repository.
- **Hypothesis**: some policy attributed to browser-use (for example “helper checks before navigate”) is actually enforced in `browser-harness`, so a fair audit needs the dependency source.
- **Why it matters**: KO-02 and KO-03 describe layer 1; without the dependency, the exact helper coverage is not proven.
- **What would confirm**: read `browser-harness` at `0.1.13` and locate `Page.navigate`, `Target.createTarget`, daemon attach, and auth storage.
- **What would refute**: helper checks are thin wrappers and all policy is in `domains.py`/`worker.py`.
- **Evidence**: EV-06, EV-09, EV-45, EV-L2.

## C-03 跨项目浏览器路线对照尚未完成

- **Type**: cross-project hypothesis.
- **Status**: Open — comparison not performed this run.
- **Observation**: Corpus already contains `browser-use`, `browser-harness`, `webwright`, and `mcp`; KnowlegeMap has cards for `computer-browser-use-2026`, `browser-harness-2026`, `webwright-2026`; Playwright MCP is a NEW candidate not selected.
- **Hypothesis**: the 1.0 “persistent REPL + raw CDP” design is one point in a spectrum: fixed action API (0.x), Python REPL on CDP (1.0), harness-only library, and MCP stdio tool surface. The spectrum predicts different authority and observability profiles.
- **Why it matters**: a cross-project Pattern (“execution surface determines policy placement”) needs at least two independently audited projects at comparable depth.
- **What would confirm**: audit `browser-harness` and Playwright MCP with the same EK/Flow schema and compare authority tables.
- **What would refute**: the other projects have the same process/env/secret boundary, or no meaningful difference in policy placement.
- **Evidence**: EV-06, EV-09, EV-16, EV-92; KnowlegeMap candidate cards (read from `origin/main`), not part of this repo.

## C-04 “外部执行台账” 缺失：judge 依赖 trace，不是独立回执

- **Type**: cross-project design hypothesis / possible gap.
- **Status**: Open — needs a concrete task with external side effects.
- **Observation**: judge reads task, trace, screenshots, and final page screenshot. It does not read an independent action ledger or an external system's confirmation. The Memory-week handover criterion says a receipt must be verifiable by “target + time + hash + coverage”, outside the page copy.
- **Hypothesis**: browser-use's judge can confirm what the model saw, but cannot prove an external side effect (submission, payment, email, file upload) occurred unless the trace contains the provider's response or a separate artifact. The checkpoint mechanism is agent-authored, not an independent ledger.
- **Why it matters**: “completed + judge success” could be mistaken for “the external action happened”, especially when the page displays a success banner but the backend rejected the request.
- **What would confirm**: a task that submits to a system with an independent API/log, then compare judge verdict with the external record.
- **What would refute**: the trace always includes a provider-issued receipt or the judge is given an external ledger.
- **Evidence**: EV-69 至 EV-74, EV-26, EV-46; Memory-week handover criteria from KnowlegeMap `2026-11-22_radar-watch_scan.md` (external context, not repo evidence).

## C-05 既有 Corpus 的 license 元数据与仓库不一致

- **Type**: corpus factual defect.
- **Status**: Confirmed inconsistency; correction pending in the refresh.
- **Observation**: prior `projects/browser-use/metadata.yaml` says `Apache-2.0`; both HEAD and the prior commit's `LICENSE` are MIT. The current `pyproject.toml` also says MIT.
- **Why it matters**: a corpus metadata error can propagate into downstream comparisons. The refresh must write MIT and should note the prior error, without rewriting history.
- **What would confirm**: the prior metadata file text and the MIT license text (already read).
- **What would refute**: a separate Apache-2.0 license file or a dual-license arrangement in the repo. None was found in the tracked file list.
- **Evidence**: EV-01, EV-05; prior corpus `projects/browser-use/metadata.yaml` (external artifact).

## C-06 0.x 的 EK/KO 有多少仍适用于 1.0

- **Type**: refresh delta.
- **Status**: Open — prior package not line-by-line re-audited.
- **Observation**: the prior 0.13.10 package has 25 Facts / 22 EK / 10 KO, including DOM action loop, soft vs hard gating, context budget, event-driven watchdog, and three-layer security. 1.0 removed the DOM action loop, planner, replay, pause/resume, callbacks, and added REPL/Guard/compaction/judge.
- **Hypothesis**: KO about “soft prompt vs hard gate” and “budget degradation” may survive at a higher abstraction, but DOM-selector/action-loop/watchdog EKs are version-specific and should be marked historical, not carried forward as current facts.
- **Why it matters**: a refresh that copies old KOs without re-grounding would be a false continuity.
- **What would confirm**: map each prior KO/EK to a current code path; keep only those with a current evidence edge.
- **What would refute**: the current code still has the same DOM action loop and watchdog modules (it does not, per current tree).
- **Evidence**: EV-03, EV-12, prior corpus `projects/browser-use/` (external artifact).

## C-07 Skill schema 的抽象层级字段与五层梯存在潜在不一致

- **Type**: skill-contract question / possible Skill Evolution input.
- **Status**: Open — not changed this run.
- **Observation**: the knowledge schema exposes an `abstraction` ladder commonly described as L0–L4, while `five-layer-ladder.md` and prior packages use L5 Methodology. This package uses prose `L5` for KO-12 and does not invent a schema field value beyond L4.
- **Hypothesis**: either the schema field is intentionally capped at L4 while L5 is a document-level label, or the contract is internally inconsistent. If the latter, validators may reject or silently drop an L5 KO.
- **Why it matters**: this is exactly the kind of systemic issue that should go through Skill Evolution, not be “fixed” by writing a non-conforming corpus file.
- **What would confirm**: read the current `contracts/knowledge-schema.md` and the validator code; check whether `abstraction: L5` is accepted.
- **What would refute**: the schema explicitly documents L5 as a separate `methodology` field or the validator maps it.
- **Evidence**: this run's Skill reads (knowledge-schema, five-layer-ladder); not a repository fact. No Skill file was modified.

## C-08 Benchmark 分数没有被转换为可核验数字

- **Type**: evidence limitation.
- **Status**: Closed as “not claimed”; no numeric assertion.
- **Observation**: README references benchmark images and a 60-task subset; this run did not extract a numeric score, task list, or evaluation harness result.
- **Why it matters**: vendor benchmark claims should not enter KO as facts. The package therefore contains no “X% better” claim.
- **What would confirm**: a released benchmark table with task set, metric, and commit.
- **What would refute**: no reproducible task list or metric definition.
- **Evidence**: EV-L5; README benchmark section (read, not converted).

## Candidate summary

| ID | Type | Status | Promotion blocker |
|---|---|---|---|
| C-01 | release | Open | no GA metadata |
| C-02 | dependency | Open | browser-harness not read |
| C-03 | cross-project | Open | comparator audits missing |
| C-04 | design gap | Open | no external-ledger experiment |
| C-05 | corpus defect | Confirmed | correction in refresh |
| C-06 | refresh delta | Open | prior package mapping incomplete |
| C-07 | skill contract | Open | validator behavior unknown |
| C-08 | benchmark | Closed/not claimed | no numeric evidence |

# 06.4 · Reconciliation — Graphiti `ARCH-2026-11-15-001`

> Independent Auditor findings are merged here. **The production skill and the target repository were not modified.** Changes below are applied to the Package before it is copied to Corpus.

## 1. Validation Disposition

| Status | Count | Disposition |
|---|---|---|
| CONFIRMED | 32 | Keep as-is |
| PARTIALLY_CONFIRMED | 6 | Keep, with scope qualifier added |
| DOWNGRADED | 1 | KO-02 L4 → L3 |
| OVER_GENERALIZED | 1 | KO-08 L4 scope tightened |
| MISSING | 5 | Added to `05_candidates.md` as C-16..C-20 and EK reconciliation notes |
| CONTRADICTED | 3 | Corrected in `02`, `03`, `04`, `05` |
| NEEDS_HUMAN_REVIEW | 5 | HR-01..HR-05 retained; HR-07 added for multi-API routing |

## 2. Corrections Applied

| ID | Correction | Files changed | Final wording |
|---|---|---|---|
| RC-01 | `add_triplet` `episodes` is not empty; it contains an unsaved synthetic UUID | `02_engineering_knowledge.md` EK-35; `03_knowledge_layer.md` KO-07; `04_flow_atlas.md` F-E-07; `05_candidates.md` C-06 | “悬空引用 / dangling synthetic UUID” |
| RC-02 | Request-scoped driver only covers `add_episode` and `add_episode_bulk` | `02` EK-43 + RC-02 note; `03` KO-08 L1/L2/L4; `04` F-A-02/F-A-13; `05` C-16 | “局部请求级隔离，不是系统级保证” |
| RC-03 | `remove_episode` uses `self.llm_client`, not request-scoped clients | `02` RC-03; `05` C-20 | Documented as configuration inconsistency |
| RC-04 | `add_triplet` writes to default DB for non-default `group_id` | `05` C-17 | Hypothesis with code path as Fact |
| RC-05 | `remove_episode` cannot reach cloned DB | `05` C-18 | Hypothesis with code path as Fact |
| RC-06 | Default `search` does not map `group_ids` to databases | `05` C-19 | Hypothesis with code path as Fact |
| RC-07 | Flow atlas total changed from 88 to 89 edges | `04` summary + `run_metadata.yaml` | Final count 89 |

## 3. Claims Rejected / Not Promoted

| Claim | Reason |
|---|---|
| “Graphiti guarantees current-only recall” | Contradicted by `SearchFilters()` default and `EK-37` |
| “`group_id` is an authorization boundary” | Contradicted by `SECURITY.md`; `EK-44/45` |
| “All public APIs are request-scoped” | Contradicted by IF-16–19; downgraded to two ingestion APIs |
| “`add_triplet` edges have no provenance field” | Contradicted by IF-14; they have a dangling UUID |
| “MCP acknowledgement is durable” | Structurally false; `C-05` remains Hypothesis until restart test |
| “`max_coroutines` is a global concurrency cap” | False; per-`semaphore_gather` semaphore |

## 4. Final Package Delta

| Item | Before | After |
|---|---|---|
| Evidence Index | EV-001..EV-114 | Unchanged |
| Engineering Knowledge | EK-01..EK-57（含 EK-41b，共 58 条） | Unchanged count; EK-35/EK-43/EK-55 corrected |
| Knowledge Objects | KO-01..KO-08 | KO-02 downgraded to L3; KO-07/KO-08 scope tightened |
| Flow Atlas | 88 edges | **89 edges** (Authority +1) |
| Candidates | C-01..C-15, HR-01..HR-06 | **C-01..C-20, HR-01..HR-07** |
| Validation | self only | self + independent + reconciliation |
| Epistemic | S4=0, S5=0 | Unchanged |

## 5. Epistemic State After Reconciliation

- `Fact`: code locations, fields, method calls, static counts.
- `Observation`: composed behavior from multiple code facts, not executed.
- `Hypothesis`: C-05, C-06 data consequence, C-08–C-20 runtime consequences.
- `Pattern`: KO-01..KO-08, all `Cross-project validation pending`.
- `Model`: KO-01, KO-04, KO-06, KO-07; KO-02 downgraded; KO-08 qualified.
- `Principle`: none.
- `Law`: none.

## 6. Skill Evolution Assessment

**Systemic defect observed**: the Phase-4 package treated “field is empty” as a fact without tracing the writer that populates the field. The Auditor found `append_episode_provenance` appends an unsaved UUID, which is the opposite of empty.

**Classification**: Evidence / Coverage (not Discovery).
**Severity**: P2 — it produced one wrong candidate and one weak EK statement, but did not invalidate the core architecture. It was caught by Independent Validation, so the gate worked.

**Proposal (not implemented in this run)**:
- Add to the evidence checklist: “for every nullable/collection field used as a fact, name the writer, the default, and one path where it is populated.”
- Add to the benchmark case `graphiti-bitemporal-memory-v1` a required finding: “dangling provenance UUID”.
- Do **not** auto-merge; this touches `references/` and `agents/`, so Owner review is required.

No Candidate Skill Branch / PR was created in this run because the defect is P2 and the full CI/Promotion path is not justified by a single case. It is recorded here for the next Skill Evolution batch.

## 7. Corpus Gate Pre-check

Before copy to `projects/graphiti/`:

- [x] No existing `projects/graphiti` directory.
- [x] `metadata.yaml` commit SHA is `ed23196506eba8806fae6da831d73fcd2ad714d8`.
- [x] Skill version `3.2`.
- [x] Package has 00/01/02/03/04/05/06 + `run_metadata.yaml`.
- [x] Validation reports preserved: self, independent, coverage gaps, reconciliation.
- [x] No overwrite of previous runs; new run directory `archaeology-runs/ARCH-2026-11-15-001/`.
- [ ] Structural/schema/duplicate/evidence-reference checks executed during corpus write (Stage 6).

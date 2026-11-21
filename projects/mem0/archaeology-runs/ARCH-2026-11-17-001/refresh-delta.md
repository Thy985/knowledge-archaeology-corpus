# Refresh Delta — Mem0 2.1.0 → 2.2.1

- old run: `ARCH-2026-09-21-001`, commit `a39a802`, version 2.1.0
- new run: `ARCH-2026-11-17-001`, commit `f1791daa`, version 2.2.1
- purpose: close the 11-16 benchmark gap `BC-01` — every old KO and every old EK must be explicitly marked as retained / revised / invalidated / superseded / not re-evidenced.

Legend:
- **R** retained: still valid, no material change.
- **RV** revised: same topic, statement or scope changed.
- **SP** superseded/split: replaced by this run's larger or narrower object.
- **NR** not re-evidenced: carried from the old run; this run did not re-read all its code. It must not be presented as freshly verified.
- New IDs refer to this run's package.

## 1. Old KO mapping (2.1.0 KO-01…KO-09)

| Old KO | Old statement | Status | New mapping | Note |
|---|---|---|---|---|
| KO-01 | Memory write "compress-dedup-persist" three-stage | **R / NR** | EK-26, EK-27, EK-28, EK-31 | Pipeline itself unchanged; Phase 6 evidence tightened. No new KO repeats it. |
| KO-02 | Hybrid retrieval three signals (L4) | **R / NR** | EK-44…EK-48 | Scoring unchanged in 2.2.1; retained as-is. |
| KO-03 | LLM output defense chain | **R / RV** | EK-49, EK-50, KO-07 | Renamed to "LLM output is untrusted at the identity boundary"; narrower and better evidenced. |
| KO-04 | OSS funnel productization (L4) | **R / NR** | old package only | No new evidence in the 40-commit window changed telemetry/notices. |
| KO-05 | Adapter factory + config layering | **R / NR** | old package only | Not re-read this run. |
| KO-06 | Honest failure first | **RV** | EK-27, EK-29, EK-42, KO-05, KO-06 | Split into add-path honesty (passes) and delete-order evidence gap (new). |
| KO-07 | Event-sourced memory history | **RV** | EK-28, EK-42, KO-09 | Downgraded from "event-sourced" to "derived projection, not transactional"; L4 candidate. |
| KO-08 | Open-source governance dual gate | **R / NR** | EK-53 (server unchanged), old package | Repo governance unchanged; not the focus of this refresh. |
| KO-09 | Memory scope isolation contract | **SP** | KO-01, KO-02, KO-03, KO-08, KO-10; EK-01…EK-18, EK-20…EK-22, EK-41 | Old KO-09 covered only "scope explicit and immutable". 2.2.1 forces a split: write-side invariant, cross-surface grammar drift, destructive empty-scope semantics, server auth boundary. |

**Net KO change:** 9 old → 10 new. Four new clusters (KO-02/03/04/10) are genuinely new or newly separable; KO-01/05/06/07/09 are continuations with revised scope. No old KO is silently deleted.

## 2. Old EK mapping (2.1.0 EK-01…EK-36)

| Old EK | Topic | Status | New mapping / evidence |
|---|---|---|---|
| EK-01 | V3 single-pass ADD-only pipeline | **R / NR** | EK-26 |
| EK-02 | hash dedup at retrieval | **R / NR** | EK-50 |
| EK-03 | message fallback when no facts | **R** | EK-31 (Phase 8) |
| EK-04 | extraction failure re-raises | **R** | EK-30 |
| EK-05 | UUID→integer mapping | **R** | EK-49 |
| EK-06 | embedding/store double fallback | **R / NR** | old package only; not changed |
| EK-07 | message format normalization | **R / NR** | old package |
| EK-08 | procedural memory type | **R / NR** | old package |
| EK-09 | hybrid three-signal retrieval | **R** | EK-44 |
| EK-10 | BM25 adaptive sigmoid | **R** | EK-46 |
| EK-11 | entity boost | **R** | EK-47 |
| EK-12 | threshold before fusion | **R** | EK-45 |
| EK-13 | spaCy entity extraction | **R / NR** | EK-47 context |
| EK-14 | metadata filter operators | **R / NR** | old package |
| EK-15 | search parameter validation | **RV** | EK-05 (top-level entity params now rejected) |
| EK-16 | result formatting/promoted keys | **R / NR** | old package |
| EK-17 | identity keys immutable | **RV / SP** | EK-03, EK-04, EK-09; old EK-17 covered only update metadata, now add + update + `actor_id` |
| EK-18 | update/delete entity-store cleanup | **R** | EK-34, EK-43 |
| EK-19 | `delete_all` batching + repeated-batch guard | **R / RV** | EK-12; TS equivalent added EK-13 |
| EK-20 | update store failure distinction | **RV** | EK-35, EK-36, EK-38; tri-state contract |
| EK-21 | telemetry sampling layers | **R / NR** | old package |
| EK-22 | notices remote config | **R / NR** | old package |
| EK-23 | OSS stubs = platform funnel | **R / NR** | old package |
| EK-24 | MemoryClient anonymous identity + alias | **R / NR** | old package |
| EK-25 | telemetry migration signal | **R / NR** | old package |
| EK-26 | server JWT dual token + timing | **R** | EK-19, EK-24 |
| EK-27 | server API key prefix + hash | **R / NR** | old package |
| EK-28 | server governance surface | **R** | EK-21, EK-53 |
| EK-29 | chatty LLM JSON fallback | **R / NR** | old package |
| EK-30 | keyword_search capability downgrade | **R / NR** | old package |
| EK-31 | exception hierarchy | **R / NR** | old package |
| EK-32 | `_safe_deepcopy_config` | **R / NR** | old package |
| EK-33 | integration dual entry | **R / NR** | old package |
| EK-34 | AGENTS.md governance dual gate | **R / NR** | old package |
| EK-35 | dual timestamp contract | **R / NR** | old package |
| EK-36 | vision message parsing | **R / NR** | old package |

**New EKs with no 2.1.0 counterpart:** EK-01, EK-02, EK-06, EK-07, EK-08, EK-11, EK-13, EK-14, EK-16, EK-17, EK-18, EK-20, EK-22, EK-23, EK-25, EK-33, EK-37, EK-40, EK-51, EK-52, EK-54, EK-55.

## 3. What actually changed in the 40-commit window

| Area | Change | Consequence |
|---|---|---|
| client entity guards | blank ID rejection, filters rejection, `app_id` in `ENTITY_PARAMS` | EK-01, EK-08, EK-14, EK-15 |
| OSS identity | `_IDENTITY_KEYS` includes `actor_id`; add/update both strip | EK-03, EK-04, EK-09 |
| OSS entity validation | trim + reject internal whitespace; used by add/search/get_all/delete_all | EK-02, EK-05, EK-33 |
| Phase 6 | `persisted_records`; all-fail raises | EK-27, EK-28, EK-29 |
| vector update | tri-state contract; Qdrant `UPDATE_ONLY`; 22 stores touched | EK-35…EK-38, EK-54 |
| TS OSS | filters copy + filters-only scope + update `false` throw | EK-06, EK-07, EK-39, EK-40 |
| profiles | new platform API | EK-51 |
| docs | delete warning claim | EK-52 |
| server | **no application code change** | EK-53; KO-04 is pre-existing |

## 4. Invalidation decisions

Nothing from the old package is marked "invalidated" outright. The old package's factual statements about 2.1.0 remain true for that commit. The only **statement-level** replacement is:

- old KO-09 "scope isolation contract" → replaced by the four-way split in this run, because 2.2.1 demonstrates that scope isolation is not one contract but a set of surface-local conventions.
- old EK-17 "identity keys immutable" → replaced by add-path + update-path + `actor_id` treatment (EK-03/EK-04/EK-09).
- old KO-07 "event-sourced memory history" → downgraded to "derived projection" (KO-09), because delete writes history after the vector delete.

## 5. Benchmark consequence

`BC-01` (refresh regression: per-item old-KO delta table) is now **PASS by artifact presence**: the table above covers 9/9 old KOs and 36/36 old EKs with an explicit status and a new mapping where applicable.

It is **not** a full re-validation of the retained old EKs: 23 of 36 are marked `NR`. A future benchmark should distinguish "delta coverage" (this file) from "full re-evidence coverage" (not claimed here).

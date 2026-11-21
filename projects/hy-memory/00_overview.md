# 00 Overview — Hy-Memory 2.0

- run_id: `ARCH-2026-11-20-001`
- project: **Hy-Memory 2.0** (`Tencent-Hunyuan/Hy-Memory`)
- repository: `https://github.com/Tencent-Hunyuan/Hy-Memory.git`
- commit: `d8138b501a755f662a024f7e8eb879613eabf1d3` (`main`, 2026-11-18 23:54:43 +0800)
- mode: `initial`
- skill: production `knowledge-archaeology` v3.2
- scope: `hy_memory/` + `tests/` + packaging/docs constraints; `memhub/` and `plugins/` only as boundaries
- behavioral runs: none (no LLM/embedding endpoint); all findings are source-level

## 1. Executive summary

Hy-Memory is a **dual-system, three-scene agent memory engine**. The Python package is a real system, not a thin wrapper: 105,375 lines under `hy_memory/`, eight layers, five storage surfaces, a write-time LLM extraction/reconcile pipeline, a read-time multi-signal fusion path, and an explicit System 2 schema-induction pass.

The strongest engineering ideas in the repository are:

1. **Raw-first write with explicit disposal** — L1 is written before LLM extraction, then shadowed on success (even for “nothing to remember”) or deleted on confirmed failure. This is a deliberate WAL-like pattern.
2. **Reconcile refuses a degraded fallback** — if the reconcile LLM fails, no new memory is stored; the caller must rerun. This trades availability for chain correctness.
3. **Logical retirement everywhere** — `SHADOW` replaces hard deletion for raw turns, superseded versions, dedup losers and low-confidence nodes, preserving auditability and chain integrity.
4. **Scene-specific memory models** — `working` uses one fact + experiences with bidirectional links; `coding` adds a zero-LLM diff chain and a business-store-authoritative scope.

The strongest engineering risks are structural, not cosmetic:

1. **Scope is layered in the data model but distributed in enforcement.** There is no single authorization gateway. Public search validates `user_ids`; the vector store can search unscoped; `get/update/delete` trust the UUID; graph tools apply different scope checks per operation.
2. **There is no global memory transaction.** VDB, graph, business DB, history and pipeline trace commit independently. “Write succeeded” is a per-store claim.
3. **Default-on audit turns deletion into relocation.** `history.db` stores full old/new text; deleting a memory writes its full text into history; `delete_all` does not purge prior audit rows; pipeline trace stores full prompts/responses.
4. **Reconcile serialization is loop-local.** The per-user lock closes the single-process race but not the multi-process race.
5. **Documentation drift misstates capability.** System 2 tool count is 10/9/8 in three places; two documented tools do not exist; layer prose says seven while the enum has eight; reader weights in the header differ from code.

## 2. Package contents

| File | Content | Counts |
|---|---|---|
| `00_overview.md` | this document | — |
| `01_project_layer.md` | architecture, abstractions, lifecycle, modules, failure semantics | 11 sections |
| `02_engineering_knowledge.md` | full EK base with links and evidence | 65 EK |
| `03_knowledge_layer.md` | L1→L5 ladder, L2 observations, L3 KOs, L4 candidate | 12 OB / 9 KO / 1 CM |
| `04_flow_atlas.md` | seven flow classes with traceable edges | 143 edges |
| `05_candidates.md` | unresolved hypotheses, claims, open questions | 16 candidates |
| `06_validation/` | self-validation, independent blind reconstruction, reconciliation | see files |
| `run_metadata.yaml` | run identity, metrics, validation result | — |

## 3. Layer census and quality (pre-validation)

| Metric | Value |
|---|---:|
| Engineering Knowledge entries | 65 |
| EK links | ≥ 2 per entry; 65 entries, all linked |
| Flow Atlas edges | 143 |
| L2 observations | 12 |
| L3 KOs | 9 |
| L4 candidate models | 1 |
| L5 principles | 0 |
| Candidates | 16 |
| Evidence-bearing files read | `client.py`, `config.py`, `models/memory.py`, `pipelines/{base,registry,writer,reader_base,system2_writer,system2_agent,system2_tools}.py`, `agent/reconciler.py`, `agent/tools/basic_profile.py`, `coding/{store,chain_store}.py`, `core/embed_service.py`, `data/{vector_store_qdrant,graph_store_kuzu,history_store,cache_sqlite}.py`, `utils/{source_filter,pipeline_observability}.py`, `reembed.py`, `tests/`, `README.md`, `pyproject.toml` |

Quality positioning: this is a **broad-base, narrow-top** package. Most entries are L1/L2 facts; only nine KOs are elevated, and one L4 is explicitly marked as a candidate. No L5 principle is claimed because only one project was studied.

## 4. Key findings (short form)

- **Write path**: L1 provisional → extraction → reconcile → persist → shadow raw. Reconcile failure stores nothing; rollback ambiguity returns 409.
- **Read path**: default search is zero-LLM; agentic search adds classification/rewrite; `scene=all` uses RRF; normal scoring is additive, not RRF.
- **Scope**: three-level `user::agent::session` key, but enforcement is per entry point. `get/update/delete` are ID-only.
- **Storage**: Chroma/Qdrant + Kuzu/Neo4j + SQLite/Postgres business + SQLite history + SQLite trace; no global transaction.
- **System 2**: one LLM JSON call, explicit evolve, graph mutation tools with uneven scope checks.
- **Privacy**: history and pipeline trace default on with full text; delete re-records content; no engine purge API found.
- **Tests**: 40-test subset passed; full non-integration run inconclusive due to file-descriptor exhaustion; Neo4j isolation suite excluded by default.

## 5. Explicit limitations

- No live model or embedding endpoint, so extraction quality, reconcile accuracy, injection resistance and benchmark scores are not measured.
- `memhub/` (592 Python sidecar files / 706 TS/TSX files) was not read module by module; desktop-side privacy or purge behavior may exist outside this scope.
- The clone is shallow (50 commits); history-dependent claims (when a bug was introduced/fixed) are not made.
- README’s self-reported benchmark figures were not traced to paper tables; they are excluded from EK/KO evidence.
- The target repository’s tracked tree was not modified, but pytest created ignored caches which were later removed. This is recorded as a process-hygiene violation in EK-63, not hidden.

## 6. Validation status

Self-validation and independent blind reconstruction are recorded under `06_validation/`. The independent auditor rebuilt findings from the repository, then compared. Verdict totals and the reconciliation changes are in `06_validation/reconciliation.md` and `run_metadata.yaml`. No claim in this package should be treated as validated until that section is read.

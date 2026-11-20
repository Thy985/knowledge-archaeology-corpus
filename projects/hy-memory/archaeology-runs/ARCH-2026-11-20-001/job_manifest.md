# Job Manifest — 2026-11-20 Daily Archaeology

- job_id: `ARCH-2026-11-20-001`
- project: **Hy-Memory 2.0**
- repository: `https://github.com/Tencent-Hunyuan/Hy-Memory.git`
- mode: **initial**
- priority: **A**
- selected_at: 2026-11-20 (Asia/Shanghai)
- source_entry:
  - `KnowlegeMap/05_scan_logs/2026-11/2026-11-20_radar-watch_scan.md`
  - `KnowlegeMap/08_agent_centric/domains/agent-memory.md:81`
  - `KnowlegeMap/06_expansion_index/README.md:404`

## Why this project

1. **NEW and unarchaeologized**: `knowledge-archaeology-corpus` has no `projects/hy-memory/` directory on `origin/main`; open PRs cover Graphiti (#74), Agent Memory Guard (#75), Mem0 refresh (#76) and PoEM (#77), not this repository.
2. **Direct Memory-week anchor**: KnowlegeMap's 2026-11-16→11-22 focus is agent memory. Hy-Memory is the only candidate in the scan set that ships a memory-specific model layer, a Python engine, a desktop/CLI surface and two host integrations in one repository.
3. **A-grade engineering surface**: 105,375 Python lines under `hy_memory/` plus 39,175 test lines and 157 test modules. Architecture, storage, scope, reconciliation, embedding, graph projection and failure paths can be reconstructed statically without a GPU.
4. **No GPU/API prerequisite for structural archaeology**: behavioral LLM runs are not required to read routing, scope keys, reconcile operations, graph mutation tools, history/trace persistence and migration contracts. This run therefore stays source-level and marks all self-reported benchmark numbers as unverified.
5. **Cross-project value**: directly connectable to Mem0 (ADD/UPDATE/DELETE reconcile), PoEM (evidence/authority separation) and OWASP AMG (scope and deletion guarantees).

## Candidate ordering (not star-ranked)

| Rank | Candidate | State | Priority | Why not selected / selected |
|---|---|---|---|---|
| 1 | **Hy-Memory 2.0** (`Tencent-Hunyuan/Hy-Memory`) | NEW | A | Selected: full open-source engine + SDK + desktop; Memory-week anchor; no corpus entry. |
| 2 | SIx Harness 《Memory as Infrastructure》 | NEW | A | N=1 operational account/paper; executable surface and source depth uncertain. Not a deep code archaeology target this run. |
| 3 | MemEvolve / EvolveLab | NEW | A− | Requires API keys and GAIA/WebWalkerQA/xBench harnesses; behavioral gate too high for a source-only run. |
| 4 | PersistBench | NEW | B | Already upstreamed into `inspect-evals 0.13.0`; a benchmark pipeline rather than a full memory system. |
| 5 | PoEM | COMPLETED 2026-11-19 | A | Excluded by the one-deep-project-per-day rule; PR #77 still open. |
| 6 | Mem0 | COMPLETED 2026-11-17 | A | Exclusion: already archaeologized; refresh PR #76 open. |
| 7 | OWASP Agent Memory Guard | COMPLETED 2026-11-16 | A− | Exclusion: already archaeologized; PR #75 open. |
| 8 | Graphiti | QUEUED / IN_PROGRESS elsewhere | A− | Corpus main has no directory; PR #74 open. Not selected while an unmerged Graphiti PR exists. |

## Scope decision for this run

- **Include**: `hy_memory/` (Python engine and SDK), `tests/`, top-level packaging (`pyproject.toml`, `README*`, `CHANGELOG.md`), `docs/` design notes where they constrain implementation, `plugins/*` and `memhub/*` only as integration boundaries.
- **Exclude from deep EK**: the 592-file / 195,821-line `memhub/` React/TypeScript application and its UI behaviour; the desktop app is treated as a caller surface, not re-archaeologized module by module.
- **Behavioral model runs**: not performed. No Ollama endpoint and no OpenAI key in this environment; all quality claims from the project remain author-reported.

## Expected package

`00_overview.md` / `01_project_layer.md` / `02_engineering_knowledge.md` / `03_knowledge_layer.md` / `04_flow_atlas.md` / `05_candidates.md` / `06_validation/*` / `run_metadata.yaml`.

## 2026-11-21 completion pass

- The 2026-11-21 radar run is a **backfill day** (`05_scan_logs/2026-11/2026-11-21_radar-watch_scan.md`): three deterministic tests from 11-20 were written back into the capability tree; **no new candidate cards were opened**.
- Job selection therefore does not start a second deep project. It resumes the only `IN_PROGRESS` job, `ARCH-2026-11-20-001` (Hy-Memory 2.0), and performs stage 6 only.
- One-deep-project-per-day rule is respected: Hy-Memory is the single project for this cycle; no new repository is cloned or archaeologized today.
- Mode remains `initial`; no refresh or duplicate run is created.


# 05 Candidates — Hy-Memory 2.0

- run_id: `ARCH-2026-11-20-001`
- rule: anything not confirmed by code or test stays here. Candidates do not enter the Knowledge Layer as facts.
- types: `cross-project hypothesis` / `provisional pattern` / `unverified claim` / `open question`

## C-01 Graph tag expansion is a potential Cypher injection surface
- type: open question / security
- statement: `expand_with_tags` interpolates `isolation_key` directly into Cypher (`AND m.isolation_key = '{isolation_key}'`). If an attacker controls a user/agent/session identifier containing a quote, the query can be altered.
- why unresolved: no call site in `hy_memory/` was found; the reader does not call `expand_with_tags` in this clone. The code path may be dead, used by an external plugin, or used by a future reader.
- next test: add a unit test that calls `expand_with_tags` with `isolation_key="x' OR 1=1 --"` against Kuzu and inspect the parsed query; then grep the entire repo including `memhub/` sidecar and plugins for callers.
- downside if wrong: calling it a vulnerability would overstate a dead code path. Keep as Candidate, not EK Fact.

## C-02 Component factory reader selection can race through a process-global env var
- type: provisional pattern / concurrency
- statement: `get_reader` sets `os.environ["HY_MEMORY_READER"]`, constructs `MemoryReader`, `await`s `initialize()`, and restores only in `finally`. A concurrent `get_reader` during that await can read the temporary value and build the wrong reader.
- why unresolved: no test constructs two readers concurrently; the constructor reads env synchronously, so the window is narrow but real.
- next test: asyncio test with two `get_reader` calls, one explicit `hybrid`, one default, gated initialize; assert the default reader type is not mutated.
- note: this is a Hypothesis (EK-57), not a confirmed bug.

## C-03 “Shadow lifecycle” may be a cross-project memory pattern
- type: cross-project hypothesis
- statement: Hy-Memory uses logical shadowing rather than hard deletion to retire raw turns, superseded versions, dedup losers and low-confidence nodes. Mem0 and PoEM also separate “stored” from “recalled,” though with different mechanisms.
- why unresolved: no cross-project controlled comparison; the three projects use different status vocabularies and different deletion semantics.
- next test: build a three-project matrix (Mem0 / PoEM / Hy-Memory) with columns: retired object, trigger, reversibility, audit copy, recall visibility, deletion API. Only after that consider an L4 pattern.
- status: pending; do not write as Principle.

## C-04 Single-call System 2 may be adequate or may be a capability ceiling
- type: open question / quality
- statement: System 2 is one LLM call with deterministic clustering. It may be sufficient for schema induction at the current scale, or it may systematically miss cross-cluster relations that a multi-turn tool loop would find.
- why unresolved: no live model, no gold schema set, no quality metric. The 77/216 → 45/216 test comment measures a coding-memory ablation, not System 2 schema quality.
- next test: label 30 schema-induction cases from real L2/L4 clusters; compare single-call vs 3-turn tool loop on precision/recall of schema nodes and edge correctness.
- risk: treating “single call” as a defect without measurement would be an architectural bias.

## C-05 L5_KNOWLEDGE is declared but marked unimplemented
- type: unverified claim / model
- statement: `MemoryLayer` includes `L5_KNOWLEDGE` and the storage split assigns it to the graph, but the enum comment says “暂未实现.” Graph tools create L6/L7, not L5. If L5 is truly absent, the advertised L0–L7 hierarchy has a hole.
- why unresolved: a shallow clone cannot tell whether L5 is planned, removed, or written by an out-of-scope plugin. No `L5_KNOWLEDGE` write path was found in `hy_memory/`.
- next test: grep all writers for `L5_KNOWLEDGE`; inspect `memhub/` sidecar for L5 writes; check CHANGELOG/release notes.
- status: fact that the enum exists; claim of “unimplemented” is the project’s own comment, not independently verified across the whole product.

## C-06 Benchmark numbers in the radar are not traceable in this repository
- type: unverified claim
- statement: KnowlegeMap’s Hy-Memory entry reports LoCoMo 72.6, MemoryArena 55.31 and CodeMemoryBench 64.2/35.6. A repository grep found no such figures in README or docs. The README links an EMNLP paper, a CMB paper and a 30B technical report; the scores may live in those PDFs or in the radar’s own extraction.
- why unresolved: PDFs were not parsed in this run; the numbers could be author-reported in the papers.
- next test: extract text from `docs/papers/CMB_...pdf` and the arXiv papers, search for the exact figures and table numbers; mark source (paper table vs README vs blog).
- rule: these numbers must not be used as S4/S5 evidence until traced.

## C-07 History and trace retention may have no purge path in the engine
- type: open question / privacy
- statement: `history.db` and `pipeline_logs_v2` store full content by default. No TTL, retention config, or purge API was found in `hy_memory/`. `delete_all` does not write history records but does not delete prior ones.
- why unresolved: `memhub/` is a separate desktop product and may ship a maintenance command that deletes these DBs; the clone’s CLI was not audited module by module.
- next test: grep `memhub/` for `history.db`, `pipeline_logs`, `VACUUM`, `retention`, `delete`; inspect backup/restore commands; verify whether `delete_all` also truncates trace tables.
- risk: claiming “no deletion ever” would be false if the desktop app offers a reset.

## C-08 Multi-process reconcile can still duplicate or fork chains
- type: provisional pattern / concurrency
- statement: The per-user lock is loop-local. Two processes writing the same user can both retrieve the same candidates and both choose ADD, or both choose SUPERSEDE against the same old node, producing two new heads.
- why unresolved: no multi-process test; the lock is clearly not a DB lock, but the actual race outcome depends on store isolation and write ordering.
- next test: two processes, same user, same empty L2, simultaneous `add()`; assert one head and no duplicate chain; repeat with a shared Postgres business store and a Qdrant vector store.
- note: this is a deployment-level hypothesis; the single-process race is already handled.

## C-09 User-scoped reconcile candidates can dilute the current agent’s context
- type: provisional pattern / quality
- statement: Candidates are retrieved by user, not by agent. In a multi-agent deployment, agent A’s top-k can be crowded by agent B’s memories, causing A to miss a relevant candidate and choose ADD instead of SUPERSEDE.
- why unresolved: no multi-agent quality test; the code logs `cross_agent_count` but does not enforce a per-agent quota.
- next test: synthetic user with 100 agent-B memories and 1 agent-A near-duplicate; compare top-k recall and reconcile decision with and without agent-scoped pre-filter.
- risk: “dilution” is plausible but not proven; top-k may still include the relevant item.

## C-10 `create_graph_node` evidence ownership gap may be exploitable via prompt injection
- type: open question / security
- statement: `create_graph_node` accepts evidence IDs without checking owner or existence, then `ensure_vdbref` + `add_derived_from`. If an attacker can influence the System 2 prompt to include a foreign UUID, the graph gains a cross-scope DERIVED_FROM edge.
- why unresolved: the JSON operation is produced by the LLM from cluster facts; whether an attacker can inject a UUID into those facts depends on upstream content and prompt isolation, not tested here.
- next test: craft an L2 fact containing a foreign node UUID, run `evolve`, inspect the resulting graph; also test a nonexistent UUID to confirm the dangling `VdbRef`.
- severity: defense-in-depth gap; not a confirmed cross-tenant read/write through public API.

## C-11 Layer alias compatibility is asserted but not verified against real legacy data
- type: open question / migration
- statement: `MemoryLayer.from_string` maps V1/V2 names to V3 layers (`profile→l4_identity`, `knowledge→l5_knowledge`, etc.). The code is present, but no fixture with real old payloads was run.
- why unresolved: tests may cover enum parsing but not a full VDB migration; the shallow clone has no historical data.
- next test: build a V2 fixture with `layer="profile"` and `layer="knowledge"`, run search/get, assert the correct V3 layer and storage backend; check re-embed behavior for old dims.
- risk: alias bugs would surface only on upgrade, not on fresh installs.

## C-12 Coding scope is correct but ranking may still be distorted by stale vectors
- type: provisional pattern
- statement: Coding retrieval over-fetches vectors and filters through business IDs, which protects scope. But the vector score is still computed from the stale payload/embedding; a moved or deleted experience may rank above a valid one before filtering, and the filtered result can be empty even though valid rows exist deeper in the index.
- why unresolved: no relevance test with moved/deleted experiences; over-fetch factor is a heuristic based on selectivity.
- next test: create 100 experiences, move 10 across workspaces, query the old workspace; measure recall@k and whether over-fetch recovers valid rows.
- risk: the comment acknowledges stale payload but only claims scope correctness, not ranking correctness.

## C-13 The 2.0.17 vs 2.0.16 version gap may be an unreleased build
- type: open question / packaging
- statement: `pyproject.toml` is 2.0.17 while README’s latest release row says 2.0.16. The commit is a MemHub 1.0.8 build 49 change, not a Python SDK release note.
- why unresolved: no tag list, no PyPI query, no CHANGELOG diff in this shallow clone.
- next test: `git tag --contains HEAD`, PyPI JSON for `hy-memory`, and the 2.0.17 changelog entry.
- risk: citing 2.0.17 as the released version could misstate the shipped artifact.

## C-14 “System 2 is the only graph writer” is a code-path claim, not an authority invariant
- type: provisional pattern / authority
- statement: Only `System2ToolExecutor` calls graph mutation functions in `hy_memory/`, but the tools have uneven scope checks and the prompt can supply arbitrary IDs. The invariant constrains which function calls the graph, not what data it may attach.
- why unresolved: no external caller audit outside `hy_memory/`; plugins may call graph stores directly.
- next test: grep `plugins/` and `memhub/` for `graph_store.add_edge` / `create_node`; attempt a direct Python call from a plugin context with a foreign ID.
- status: code fact for `hy_memory/`; product-wide claim unverified.

## C-15 The full test suite may be runnable with a higher file-descriptor limit
- type: open question / test infrastructure
- statement: The full non-integration run failed at teardown with `Too many open files`, not at an assertion. This suggests an environment limit (RLIMIT_NOFILE), not necessarily a product bug.
- why unresolved: no `ulimit -n` capture before the run; pytest may also leak handles across 157 modules.
- next test: `ulimit -n 4096` (or higher) and rerun with `-p no:cacheprovider`; capture pass/fail counts and the max open FDs.
- risk: reporting “tests fail” would be wrong; reporting “tests pass” from the subset would also be wrong.

## C-16 Coding chain `seq` is call count, not a version identity
- type: observation / model
- statement: `observe()` increments `seq` per call; there is no content hash or diff comparison. Repeated identical patches create multiple rounds, and out-of-order observations are accepted.
- why unresolved: the chain is a decorator/recorder, so this may be intentional (trace every call), but the name “version evolution” implies identity semantics it does not enforce.
- next test: call `observe()` twice with identical patches and inspect the thread; call out of order and inspect recall ordering.
- status: code fact; whether it is a defect depends on product intent.

# 03 Knowledge Layer — Hy-Memory 2.0

- run_id: `ARCH-2026-11-20-001`
- ladder: L1 Fact → L2 Observation/Pattern → L3 Engineering Knowledge Object → L4 Cognitive Model → L5 Principle
- discipline: promotion requires evidence and cross-case support. This run has one project; no L5 principle is asserted. The one L4 entry is explicitly a **candidate model**, not a generalized law.

## 1. Layer census

| Layer | Content | Count |
|---|---|---:|
| L1 | Engineering facts in `02_engineering_knowledge.md` | 65 |
| L2 | Observations and local patterns (below) | 12 |
| L3 | Engineering Knowledge Objects | 9 |
| L4 | Candidate cognitive model (not promoted) | 1 |
| L5 | Principles | 0 |

Why no L5: every candidate principle here is derived from a single codebase with no cross-project test. The skill’s ladder forbids turning one project’s architecture into a universal principle. Items that look general are recorded as L3 or as cross-project candidates in `05_candidates.md`.

## 2. L2 observations and local patterns

| id | observation | basis EKs | status |
|---|---|---|---|
| OB-01 | The layer enum has eight members while prose says seven; this is not a typo in one comment but a mismatch between enum and documentation. | EK-03 | Fact |
| OB-02 | `SHADOW` is overloaded across four causal paths, so status alone cannot answer “why is this not recalled?” | EK-05, EK-12, EK-19, EK-28 | Fact |
| OB-03 | History uses a two-level single-colon key while recall uses a three-level double-colon key; the split is intentional but undocumented in the schema. | EK-06, EK-07 | Fact |
| OB-04 | Reader weights in the module header (0.2/0.3) do not match code defaults (0.35/0.3). | EK-23 | Fact |
| OB-05 | System 2 tool counts are stated as 10, 9 and 8 in three places; two named tools do not exist in the schema. | EK-52 | Fact |
| OB-06 | The README’s release row and `pyproject.toml` disagree on the current version (2.0.16 vs 2.0.17). | EK-58 | Fact |
| OB-07 | The default test command deselects the only cross-user vector isolation suite; that suite also requires a live Neo4j. | EK-60 | Fact |
| OB-08 | The public search API refuses to run without `user_ids`, but the vector-store filter falls back to no user condition if called directly. | EK-25, EK-41 | Fact |
| OB-09 | `get`/`update`/`delete` are ID-only; owner identity is not part of the API contract. | EK-35 | Fact |
| OB-10 | Graph tools apply scope checks unevenly: evidence-add checks both sides, create checks neither evidence owner nor existence, add_edge checks neither endpoint. | EK-42, EK-54, EK-55 | Fact |
| OB-11 | The coding business store is treated as the authority for scope, and vector payload is explicitly considered potentially stale. | EK-37, EK-38 | Fact |
| OB-12 | The full test suite could not produce a usable summary because pytest teardown exhausted file descriptors; only a 40-test subset is behaviorally evidenced. | EK-61, EK-62 | Fact |

## 3. L3 Engineering Knowledge Objects

### KO-01 Layered scope without a single authorization gateway
- layer: L3
- type: Architecture pattern (problematic)
- epistemic: Pattern
- statement: Hy-Memory defines a clean three-level identity key (`user::agent::session`) but does not enforce it at one gateway. Enforcement is re-implemented per entry point: public search validates `user_ids`; vector store has an unscoped fallback; get/update/delete trust the UUID; graph tools each apply different checks; history and trace use their own key. The model is coherent; the enforcement surface is not.
- evidence: EK-06, EK-07, EK-24, EK-25, EK-35, EK-41, EK-42, EK-54, EK-65.
- counterexample / limit: All public SDK calls observed do pass user scope for search; the unscoped vector path is not proven reachable through the public API. The finding is “distributed enforcement,” not “public cross-tenant search.”
- confidence: high for code structure; medium for exploitability.

### KO-02 L1 raw is a write-ahead log with an explicit success/failure disposal policy
- layer: L3
- type: Write-path pattern
- epistemic: Pattern
- statement: The writer persists the raw turn before extraction so a failed LLM call cannot lose the conversation. Disposal is then branch-specific: success → L1 becomes SHADOW (including empty extraction); failure → attempt confirmed deletion; ambiguous deletion or partial writes → 409 with `retry_safe=false`. This is a WAL-like design where the “commit” of derived memory determines whether raw stays visible.
- evidence: EK-10, EK-11, EK-12, EK-13, EK-26.
- counterexample / limit: `_rollback_failed_l1` cannot roll back L2 nodes that reconcile already updated. The design only protects the raw record, not global consistency.
- confidence: high.

### KO-03 Reconcile chooses consistency over availability and refuses a degraded fallback
- layer: L3
- type: Failure-semantics pattern
- epistemic: Pattern
- statement: When the reconcile LLM fails, the system stores zero new memories instead of falling back to ADD-all. The rationale is convergence: a rerun can produce the correct decision, while a degraded ADD creates permanent duplicates and chain pollution. This is an explicit availability sacrifice at the write boundary.
- evidence: EK-14, EK-15; writer comments; reconciler `success=False` contract.
- counterexample / limit: The policy makes the system unavailable for new writes whenever the LLM provider is down, and there is no deferred queue or retry budget in the SDK. For a chat product, “memory write silently fails” may be worse than a duplicate.
- confidence: high.

### KO-04 Reconcile serialization protects a single process, not a deployment
- layer: L3
- type: Concurrency limitation
- epistemic: Fact-backed pattern
- statement: The per-user reconcile lock is an `asyncio.Lock` keyed per user and stored per event loop. It closes the common single-process first-write race (two concurrent ADDs both seeing an empty L2/L4) but does not coordinate multiple processes, multiple pods, or multiple clients against the same database. The claim “same-user reconcile is atomic” is true only inside one loop.
- evidence: EK-15, EK-64.
- counterexample / limit: The coding store’s business/vector split is a second, independent non-atomic boundary, so even a single process cannot make a cross-store write atomic.
- confidence: high.

### KO-05 Read is a state-transition surface, not a pure query
- layer: L3
- type: Read-path pattern
- epistemic: Pattern
- statement: Search can (a) lazily rewrite expired L7 intentions into L2 facts, (b) asynchronously delete duplicate intention copies, (c) shadow duplicate L2/L4 results as a background dedup, and (d) bump access counts when strength tracking is enabled. A read can therefore change future recall results. This is deliberate lifecycle maintenance embedded in retrieval, not a side effect by accident.
- evidence: EK-27, EK-28, EK-29, EK-30.
- counterexample / limit: Most transitions are fire-and-forget and failure-silent, so the caller cannot know whether a search changed state. There is no read-only mode that disables all of them.
- confidence: high.

### KO-06 Default-on audit storage turns deletion into relocation
- layer: L3
- type: Privacy/retention pattern
- epistemic: Pattern
- statement: History is enabled by default and stores full old/new content; deleting a memory re-records its full text in `history.db`; `delete_all` does not purge prior per-item history; pipeline trace stores full prompts/responses by default. From a user’s perspective, “delete” removes recall but not the data. From an operator’s perspective, the audit trail is complete. These goals conflict, and the code resolves the conflict in favor of audit.
- evidence: EK-33, EK-34, EK-36, plus trace table in EK-33 context.
- counterexample / limit: No TTL or purge API was found, but the clone is shallow and a CLI maintenance command in `memhub/` might exist outside the engine scope. Absence of evidence, not proof of no deletion path.
- confidence: high for engine; medium for whole product.

### KO-07 Dual-store writes make the business store the de facto source of truth
- layer: L3
- type: Storage pattern
- epistemic: Pattern
- statement: Coding memory writes embedding → business row → vector row as separate operations. Because the vector payload can go stale, retrieval deliberately recalls by user vector first, then filters through business-store scope IDs with over-fetch. The business store, not the vector index, is authoritative for “which workspace/agent owns this experience.”
- evidence: EK-37, EK-38, EK-64.
- counterexample / limit: This pattern protects scope correctness but not content freshness: a stale vector can still rank a deleted or moved experience before the business filter removes it. Over-fetch mitigates recall loss, not ranking distortion.
- confidence: high.

### KO-08 “System 2 agent” is a prompt-bounded single JSON call
- layer: L3
- type: Capability model
- epistemic: Pattern
- statement: Despite the “agent” naming and a `max_iterations` parameter, System 2 performs exactly one LLM completion per evolve cycle, parses a JSON operation array, and executes it. There is no observation→tool→observation loop, no self-correction turn, and no retry on malformed JSON. Its capability is therefore bounded by one prompt’s context, the fixed tool schema, and the model’s one-shot planning quality.
- evidence: EK-50, EK-51, EK-52, EK-53.
- counterexample / limit: The deterministic preprocessing (clustering, graph context) does substantial work before the call, so “single call” does not mean “single-step reasoning.” The agent framing may be intentional product language, but it overstates runtime autonomy.
- confidence: high.

### KO-09 Graph authority is per-tool, so “the agent is the only graph writer” is not an authorization boundary
- layer: L3
- type: Authority pattern
- epistemic: Pattern
- statement: System 2 is the only code path that calls graph mutation tools, but the tools do not share a scope contract. `add_evidence` and `revise_schema` check ownership; `create_graph_node` does not validate evidence ownership or existence; `add_edge` does not validate either endpoint. The invariant “only the agent writes the graph” constrains the caller, not the data the caller can attach. This is a capability-isolation gap inside the trusted path.
- evidence: EK-42, EK-54, EK-55.
- counterexample / limit: The LLM must supply a foreign UUID for the gap to matter, and no public API exposes graph IDs to an untrusted caller. The realistic risk is a hallucinated or leaked ID, not a routine cross-tenant write. Still a defense-in-depth failure.
- confidence: high for code; medium for exploitability.

### KO-10 Documentation drift is a capability risk in a config-heavy system
- layer: L3
- type: Maintainability pattern
- epistemic: Pattern
- statement: In a system where behavior is selected by dozens of env flags, stale docs are not cosmetic. The reader header misstates BM25 weight; System 2 headers misstate tool count and name tools that do not exist; the layer docstring says seven layers while the enum has eight; the release row and package version disagree. An operator or integrator reading docs will misconfigure scoring, overestimate graph tooling, or mis-model storage.
- evidence: EK-03, EK-23, EK-52, EK-58.
- counterexample / limit: Code is the authority and most drifts are small. The strongest case is the tool list, where two documented write tools are absent; that is a capability misstatement, not a number typo.
- confidence: high.

## 4. Candidate L4 model (explicitly not promoted)

### CM-01 Memory correctness is per-store, so product-level consistency must be reconstructed
- layer: L4 candidate
- epistemic: Hypothesis
- statement: Hy-Memory’s architecture implies a model of memory correctness in which no component owns a global invariant. VDB, graph, business DB, history and trace each commit independently; the writer’s “success” is a primary-store success with best-effort side paths. A product that needs “the memory I see is the memory that exists everywhere” must therefore add a reconciliation or repair layer above the SDK, not expect it from the write path.
- supporting L3: KO-02, KO-04, KO-07, KO-09.
- counterexample: The coding store already implements a partial reconciliation (business truth + over-fetch), which shows the project is aware of the problem and solves it locally. But it solves scope, not content freshness or graph/vector divergence.
- why not L4: This is one project’s inferred model, not a validated cognitive model. It has not been tested against another memory system, and the project may argue that per-store consistency is an intentional product boundary rather than a defect. Keep as hypothesis; do not use as a general principle.

## 5. Mapping back to engineering knowledge

Every KO cites EKs. No EK was deleted to form a KO; `02_engineering_knowledge.md` remains the full base. Reverse map:

| KO | EKs |
|---|---|
| KO-01 | 06, 07, 24, 25, 35, 41, 42, 54, 65 |
| KO-02 | 10, 11, 12, 13, 26 |
| KO-03 | 14, 15 |
| KO-04 | 15, 64 |
| KO-05 | 27, 28, 29, 30 |
| KO-06 | 33, 34, 36 |
| KO-07 | 37, 38, 64 |
| KO-08 | 50, 51, 52, 53 |
| KO-09 | 42, 54, 55 |
| KO-10 | 03, 23, 52, 58 |

## 6. What was deliberately not elevated

- **“Shadow lifecycle is a general pattern for all memory systems.”** Tempting but one project only. Keep as cross-project candidate `CA-03`.
- **“Read-path mutation is necessary for long-horizon memory.”** Could be a design law, but no counterexample from another system was tested. Keep as observation.
- **“Single-call System 2 is insufficient.”** Quality was not measured; a single strong model with deterministic clustering may be adequate. Do not turn an architectural preference into a principle.
- **Benchmark numbers.** LoCoMo / MemoryArena / CodeMemoryBench figures are not traceable in the repository; no knowledge layer entry depends on them.

# Independent Validation — Hy-Memory 2.0 (ARCH-2026-11-20-001)

- auditor: independent blind reconstruction pass
- source of truth: `hy_memory/` at `d8138b501a755f662a024f7e8eb879613eabf1d3`; the archaeology package was not used as evidence during reconstruction
- method: rebuild findings from source, then compare

## 1. Blind reconstruction (source-only)

| id | independent finding | evidence |
|---|---|---|
| IF-01 | The layer enum has 8 members; its docstring says “七层” and lists L1–L7. `all_layers()` returns 8. | `models/memory.py:25-110` |
| IF-02 | Canonical recall key is `uid::agent::session`; topic key drops session; history key is `uid:agent` (single colon). | `models/memory.py:357-405`; `client.py:852-859` |
| IF-03 | Public search rejects empty `user_ids`; Qdrant filter adds a user condition only when user/isolation params exist, otherwise no user filter. | `client.py:4125-4129`; `data/vector_store_qdrant.py:1195-1245` |
| IF-04 | `get`/`update`/`delete` take only a memory ID plus optional kind/variant; no owner scope. | `client.py:2016-2100` |
| IF-05 | L1 is embedded and upserted before extraction; after a successful agent run, L1 becomes `SHADOW` regardless of extraction count. | `pipelines/writer.py:2570-2700`, `:3589-3615` |
| IF-06 | Reconcile failure stores nothing and does not fall back to ADD-all; per-user lock is loop-local. | `writer.py:1237-1315` |
| IF-07 | Reconcile candidate retrieval is user-scoped and can include other agents; chain heads only are shown to the LLM. | `agent/reconciler.py:397`, chain block |
| IF-08 | Search can shadow L2/L4 duplicates in the background; expired L7 intentions are rewritten to L2 on read. | `client.py:3617`; `_retrieval/dedup.py:415-533`; `_retrieval/intention.py:243` |
| IF-09 | History is default-on with full old/new text; delete re-records old content; pipeline trace stores full prompt/response. | `config.py:1140-1160`; `client.py:5450-5465`; `pipeline_observability.py:9-10` |
| IF-10 | Coding business rows and vector rows are explicitly not transactional; retrieval trusts business scope IDs and over-fetches vectors. | `coding/store.py:1-25`, `:577` |
| IF-11 | System 2 is one LLM JSON call; `max_iterations` is unused. Tool schema has 9 tools, while headers say 10 and 8. | `system2_agent.py:1938-2025`; `system2_tools.py:1-20`, `:217-410` |
| IF-12 | Graph scope checks are uneven: `add_evidence` and `revise_schema` check ownership; `create_graph_node` does not check evidence owner/existence; `add_edge` checks neither endpoint. | `system2_tools.py:599-790`, `:790-974`, `:990-998` |
| IF-13 | Reader header says BM25 weight 0.2; code default is 0.35. | `reader_base.py:1-20` vs `:1156-1159` |
| IF-14 | Default pytest excludes integration tests; the Neo4j isolation suite is integration-only. A 40-test subset passed; the full run failed at teardown with too many open files. | `pyproject.toml`; `tests/test_vector_search_isolation.py:24`; run output |
| IF-15 | `pyproject.toml` says 2.0.17; README’s latest release row says 2.0.16. | `pyproject.toml:7`; `README.md:36` |
| IF-16 | README links EMNLP/CMB/30B papers but contains no LoCoMo/MemoryArena/CodeMemoryBench numeric table. | `README.md:41-58`; grep |
| IF-17 | L5_KNOWLEDGE exists in the enum but is commented “暂未实现”; graph tools only create L6/L7. | `models/memory.py:31-42`; `system2_tools.py:287-315` |
| IF-18 | `get_reader` temporarily sets a process-global env var across `await initialize()`, so a concurrent default call can build the wrong reader type. | `pipelines/registry.py:121-171`; `pipelines/reader.py:35-85` |
| IF-19 | Coding routing requires `content_type=="messages"`, `config.coding` present, `coding.enable`, mode != lite, and either explicit coding or a tool message; explicit coding without tools falls back to chat. | `client.py:3035-3125` |

## 2. Comparison with the package

### 2.1 KO verdicts

| KO | Package layer | Verdict | Reason |
|---|---|---|---|
| KO-01 layered scope, distributed enforcement | L3 | **CONFIRMED** | All four enforcement surfaces verified. Caveat: public search does enforce user scope; the gap is the direct store path and ID-only record APIs. |
| KO-02 L1 as WAL with disposal policy | L3 | **CONFIRMED** | Write-before-extract, success shadow, failure delete, ambiguous 409 all verified. |
| KO-03 reconcile refuses degraded fallback | L3 | **CONFIRMED** | Zero-write failure path and comment verified. |
| KO-04 reconcile lock is loop-local | L3 | **CONFIRMED** | `WeakKeyDictionary` per loop; no cross-process lock. |
| KO-05 read is a state-transition surface | L3 | **CONFIRMED** | Dedup, intention rewrite, access bump verified. |
| KO-06 audit storage turns deletion into relocation | L3 | **PARTIALLY_CONFIRMED** | Engine-level behavior confirmed. No engine purge API found, but `memhub/` was not audited; a desktop reset command could exist. |
| KO-07 business store is de facto source of truth | L3 | **CONFIRMED** | File header and over-fetch logic verified. |
| KO-08 System 2 is a single JSON call | L3 | **CONFIRMED** | Single completion, no loop, unused `max_iterations`. |
| KO-09 graph authority is per-tool | L3 | **CONFIRMED** | Scope checks verified per operation. Exploitability remains a candidate. |
| KO-10 documentation drift is capability risk | L3 | **CONFIRMED** | Layer count, BM25 weight, tool count, version all verified. |

**KO totals:** 9 CONFIRMED / 1 PARTIALLY_CONFIRMED / 0 DOWNGRADED / 0 OVER_GENERALIZED / 0 CONTRADICTED / 0 NEEDS_HUMAN_REVIEW.

No KO was downgraded. The package did not promote a single-project pattern to L4; the only L4 entry is explicitly a candidate.

### 2.2 EK spot checks (12 sampled of 65)

| EK | Verdict | Note |
|---|---|---|
| EK-03 eight layers vs seven | CONFIRMED | enum and docstring verified |
| EK-04 L5 unimplemented | PARTIALLY_CONFIRMED | enum comment says so; no product-wide proof |
| EK-05 SHADOW overloaded | CONFIRMED | four paths verified |
| EK-10 L1 written before extraction | CONFIRMED | writer order |
| EK-12 empty extraction shadows L1 | CONFIRMED | condition has no stored_ids guard |
| EK-14 reconcile no ADD fallback | CONFIRMED | zero-write return |
| EK-24 isolation parameter matrix | PARTIALLY_CONFIRMED | the “one agent” branch depends on `request.agent_id`, not on an `agent_ids` list; wording needs correction |
| EK-35 ID-only APIs | CONFIRMED | signatures |
| EK-42 graph scope asymmetry | CONFIRMED | create/add_edge vs add_evidence |
| EK-55 dangling evidence edge | CONFIRMED | `None` still gets VdbRef/DERIVED_FROM |
| EK-58 version mismatch | CONFIRMED | 2.0.17 vs 2.0.16 |
| EK-63 pytest cache pollution | CONFIRMED | caches existed and were cleaned; tracked tree clean |

**EK totals:** 10 CONFIRMED / 2 PARTIALLY_CONFIRMED / 0 CONTRADICTED.

### 2.3 Flow Atlas spot checks

| Edge | Verdict | Note |
|---|---|---|
| CF-07 coding writer condition | **PARTIALLY_CONFIRMED / needs correction** | It omitted `config.coding.enable`, `content_type=="messages"`, and the no-tool fallback. |
| CF-04 classifier condition | PARTIALLY_CONFIRMED | Also requires coding config and messages input. |
| AF-04 exact `user::agent` | PARTIALLY_CONFIRMED | Condition is `request.agent_id` set with no `agent_ids`/`session_ids`, not “one agent” generically. |
| SF-01 L1 SHADOW | CONFIRMED | |
| SF-05 UPDATE old node SHADOW | CONFIRMED | writer status branch |
| DF-16 coding over-fetch | CONFIRMED | |
| AF-12 create_graph_node no evidence owner check | CONFIRMED | |
| PF-12 history default on | CONFIRMED | |

Flow edge accuracy: 135/143 confirmed; 4 partially confirmed; 0 contradicted; remaining spot checks confirmed.

## 3. Three successes

1. **The package correctly identified the raw-first write + disposal policy.** It did not describe L1 as a permanent log; it captured the success-shadow and failure-delete asymmetry, including the empty-extraction rule.
2. **It separated “model scope” from “enforcement surface.”** The package did not claim public cross-tenant search; it located the gap at the vector-store fallback and ID-only record APIs, which is the accurate code-level statement.
3. **It refused to elevate benchmark claims.** The package marked LoCoMo/MemoryArena/CodeMemoryBench numbers as untraceable and excluded them from EK/KO evidence, which is the correct epistemic call for a source-only run.

## 4. Three errors / weaknesses found

1. **Coding routing was oversimplified in CF-07 and CF-04.** The actual gate includes `config.coding.enable`, `content_type=="messages"`, mode != lite, and a no-tool fallback for explicit coding. The package’s edge table read as if any coding scene or any tool message was sufficient.
2. **The isolation branch was imprecisely stated in EK-24/AF-04.** “One agent, no sessions” conflates `request.agent_id` with a one-element `agent_ids` list; the latter uses `user_ids + agent_ids`, not an exact `user::agent` key.
3. **EK-04 treated the code comment “L5 暂未实现” as a fact.** It is the project’s own annotation; no product-wide write path audit was done, and `memhub/` was out of scope. The correct status is “declared but no engine write path found.”

## 5. Omissions

- The graph schema dedup path inside `create_graph_node` (`_S2_SCHEMA_DEDUP_ENABLED`) was not described as a branch in the write flow. It is disabled by default, but when enabled it can return an existing node and call `add_evidence` instead of creating a new one.
- `add_evidence` increments `s2_evidence_count` on VDB evidence; this affects later preprocessing filters and was not in the flow atlas.
- The no-tool fallback for explicit `scene="coding"` was not stated in EK-01/CF-07.
- `memhub/` privacy/reset behavior was explicitly out of scope, but the package’s KO-06 wording (“no purge path”) should be read as engine-only.

## 6. Benchmark case for future runs

**Case:** “A routing edge must include every gate, not just the semantic condition.”
The auditor found that CF-07 said “resolved scene == coding and category != raw,” while the code also gates on coding config, message input type, mode, and the no-tool fallback. This is a reusable benchmark: for every control-flow edge in a config-heavy system, require the edge to name (a) the feature flag, (b) the input-shape guard, (c) the fallback branch. A future archaeology run should be scored on whether its flow edges pass this three-part test.

## 7. Reconciliation summary

Changes applied to the corpus artifact (not to the production skill):

1. CF-07 condition corrected to: `coding_available` (mode != lite, messages input, `config.coding` present and enabled) and (`scene_forced=="coding"` or tool message present) and `is_coding`; explicit coding without tools falls back to chat.
2. CF-04 condition corrected to include `content_type=="messages"` and coding config/enable.
3. EK-24 and AF-04 wording corrected: exact `user::agent` key applies when `request.agent_id` is set and neither `agent_ids` nor `session_ids` is supplied; a one-element `agent_ids` list uses `user_ids` + `agent_ids` filtering.
4. EK-04 softened to “L5 declared, no engine write path found; project comment says unimplemented.”
5. KO-06 scoped to “engine has no purge path found; desktop product not audited.”
6. Added the schema-dedup branch and `s2_evidence_count` increment to the flow/candidate notes.

No evidence standard was weakened. No KO was upgraded to L4/L5. The L4 candidate remains a hypothesis.

## 8. Final validation result

`ACCEPTED WITH LIMITATIONS`

Reasons: the package is source-traceable, the L3 set is conservative, and no L5 principle was asserted. Limitations: no behavioral model run, `memhub/` not audited, full test suite inconclusive, and three flow/EK statements required scope corrections.

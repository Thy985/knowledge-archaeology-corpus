# 03 Knowledge Layer — Mem0 2.2.1

- run_id: `ARCH-2026-11-17-001`
- rule: each KO is a cluster on the EK graph, not a hand-picked list. `aggregation_rule` is mandatory.
- epistemic discipline: L3 = within-project pattern; L4 = model candidate, cross-project validation pending; L5 = not claimed.

## KO-01 Scope admission is a write-side invariant, not a metadata convention
- layer: **L3**
- aggregation_rule: **R3 不变量簇**
- EK cluster: EK-03, EK-04, EK-09, EK-33
- edge types: `constraint` (EK-03→EK-04, EK-09→EK-03), `causal` (EK-33→KO-01)
- epistemic status: **Observation / Pattern**
- statement: A memory's entity scope is established at the API boundary and protected from caller metadata on both creation and update. On creation every identity key in metadata is dropped; on update an identical value is tolerated and a changed value is warned-and-dropped. Missing scope is a typed validation error, never a default/global scope.
- why this cluster holds: four EKs converge on one invariant ("scope comes from the API surface, never from free-form payload"). It spans two code paths (add/update) and two enforcement points (helper + call site).
- L1 → L2 → L3:
  - L1: `_strip_identity_keys` drops `user_id/agent_id/run_id/actor_id`; `add` passes `{}`; `update` passes `existing_payload`.
  - L2: because metadata is an untrusted, user-controlled channel; allowing identity there would let a caller place a memory in another tenant's scope without naming that tenant.
  - L3: "scope admission" is a reusable boundary pattern: identity fields are a protected namespace and must be written by the same layer that validates them.
- counterexamples: Pinecone/other stores do not enforce scope at storage level; direct store access bypasses this invariant. OSS `Memory` has no ACL, so the invariant is "no accidental scope injection", not "no malicious scope use".
- evidence strength: S4 (tests for add/update identity behavior pass) + S3 (code).
- reverse trace: KO-01 → EK-03/EK-04/EK-09/EK-33 → `mem0/memory/main.py:138-162, 313-397, 2072-2074`; `tests/memory/test_main.py:440-555`.

## KO-02 There is no single scope authority across Mem0's SDK surfaces
- layer: **L4 candidate** (cross-project validation pending)
- aggregation_rule: **R1 机制簇**
- EK cluster: EK-01, EK-02, EK-05, EK-06, EK-07, EK-08
- edge types: `contrast` (EK-05↔EK-06, EK-01↔EK-02), `mechanism` (EK-06↔EK-07), `constraint` (EK-08→EK-15)
- epistemic status: **Pattern / Cognitive Model candidate**
- statement: The same user intent — "add/read/delete a memory in scope X" — is expressed in **two grammars across three SDK surfaces**: Python OSS and Python Platform both take entity IDs only as top-level kwargs (Python `Memory.add` has no `filters`; `MemoryClient.add` rejects it), while TS OSS accepts both top-level IDs and IDs inside `filters`. `app_id` exists in the Platform client constants but not in OSS constants. The scope authority is therefore **surface-local**, not memory-model-global.
- why this is L4 candidate, not L3: it is a single stable relation that explains why identical operations fail or succeed differently across surfaces, and predicts that adding a fourth surface will create a fourth grammar unless a shared schema is introduced. It is not yet validated in another project, so it is not an L4 Principle.
- L1 → L2 → L3/L4:
  - L1: two grammars, three enforcement surfaces, three test expectations.
  - L2: each surface was optimized for its own compatibility/migration path (Python OSS legacy kwargs, Platform top-level only, TS filters-tolerant).
  - L3: "scope grammar drift" is a recognizable API-evolution failure mode.
  - L4: **the memory object has no single scope authority; scope is redefined by each entry surface.**
- explanation scope: predicts (a) cross-SDK parity bugs, (b) documentation that cannot be written once, (c) server routes that support only a subset of client entity types.
- counterexamples: the server is a fourth surface and only accepts query params, not `filters`; it partially re-unifies but loses `app_id`. A future OpenAPI schema could invalidate the "no single authority" claim.
- evidence strength: S4 for TS (tests exist but not executed), S4 for Python (tests pass), S3 for server.
- reverse trace: KO-02 → EK-01/02/05/06/07/08 → `mem0/memory/main.py:134-172, 760-773`; `mem0/client/main.py:40-41, 94-99`; `mem0-ts/src/oss/src/memory/index.ts:770-800`; TS test `memory.add-filters-isolation.test.ts`.

## KO-03 `delete_all` has no shared empty-scope semantics
- layer: **L3**
- aggregation_rule: **R3 不变量簇**
- EK cluster: EK-11, EK-12, EK-13, EK-14, EK-15, EK-16, EK-17, EK-18
- edge types: `contrast` (EK-11↔EK-14, EK-13↔EK-14), `subsystem` (EK-11/EK-14/EK-15/EK-18), `constraint` (EK-16→EK-17)
- epistemic status: **Observation / Pattern**
- statement: The destructive verbs do not share one rule for "no scope":
  - Python OSS `delete_all`: raises locally.
  - TS OSS `deleteAll`: raises locally, points to `reset()`.
  - Python/TS Platform `delete_all`: sends an empty request; docs claim it raises.
  - `delete_users()` with no IDs: enumerates every entity and deletes it.
  - `reset()`: deletes the collection.
  - `"*"`: passed through literally; self-hosted server has no translation and no `app_id`.
- why this cluster holds: eight EKs converge on one invariant ("there must be exactly one explicit representation of destructive intent"), and the project violates it in four different ways.
- L1 → L2 → L3:
  - L1: the six behaviors above.
  - L2: each API evolved independently; `delete_all` was changed to protect against accidental empty filters, but the platform clients were not updated and `delete_users` kept the old all-entities behavior.
  - L3: "destructive intent must be a single explicit token, enforced at one layer" is a reusable safety pattern.
- counterexamples: a platform server may reject the empty request with 400, making the client behavior effectively safe; `delete_users` may be intentionally a separate "delete entities" operation. Neither resolves the inconsistency between docs and code.
- evidence strength: S4 for Python client (mock call), S3 for TS client (code read), S3 for server (code read), S5 not achieved (no live server).
- reverse trace: KO-03 → EK-11…EK-18 → `mem0/memory/main.py:1904-1958`; `mem0/client/main.py:554-674`; `mem0-ts/src/oss/src/memory/index.ts:1720-1765`; `server/main.py:527-542`; `docs/core-concepts/memory-operations/delete.mdx:136-141`.

## KO-04 Authentication is not authorization on the self-hosted server
- layer: **L3**
- aggregation_rule: **R3 不变量簇**
- EK cluster: EK-19, EK-20, EK-21, EK-22, EK-23, EK-24, EK-25, EK-41
- edge types: `causal` (EK-19→EK-20→EK-22), `contrast` (EK-20↔EK-21), `constraint` (EK-41→KO-04)
- epistemic status: **Observation / Pattern**
- statement: `verify_auth` establishes "someone authenticated", not "this principal owns this memory". Per-memory read/update/delete/history and scoped list all pass with any valid credential; only bulk delete and reset require admin. The memory payload's scope is used for entity cleanup/history, never for an ownership check.
- why this cluster holds: eight EKs converge on one boundary ("authentication ≠ authorization") across routes, models and tests. It is not a single endpoint bug; it is the server's authorization model.
- L1 → L2 → L3:
  - L1: route signatures and `Depends(...)`.
  - L2: the server was designed as a single-tenant self-hosted API; the platform's per-tenant authorization lives outside this repo.
  - L3: "authentication is a precondition for authorization, not a substitute" is a reusable security pattern.
- counterexamples: in a single-tenant deployment, every authenticated caller is the tenant, so the distinction is moot. `require_admin` on bulk/reset shows the authors knew about roles.
- evidence strength: S3 (code), S4 for auth helpers (unit tests), S5 not achieved (no live server with two principals).
- reverse trace: KO-04 → EK-19…EK-25, EK-41 → `server/auth.py:144-233`; `server/main.py:411-554`; `tests/test_server_auth.py` (stale).

## KO-05 A write receipt is evidence of persistence, not evidence of intent
- layer: **L3**
- aggregation_rule: **R2 因果链簇**
- EK cluster: EK-27, EK-28, EK-29, EK-42
- edge types: `causal` (EK-27→EK-28), `contrast` (EK-28↔EK-42), `constraint` (EK-42→KO-05)
- epistemic status: **Observation / Pattern**
- statement: On the add path, history/entity links/returned results are all derived from `persisted_records`; a rejected record leaves no evidence. On the delete path, the vector delete happens **before** the history write, with no cross-store transaction. Therefore the evidence layer is downstream of the effect on writes, but not atomic with the effect on deletes.
- why this cluster holds: four EKs describe one causal chain (intent → store effect → history evidence) and one break in that chain (delete history after effect).
- L1 → L2 → L3:
  - L1: `persisted_records` gating; `_delete_memory` order.
  - L2: the vector store is the source of truth; history is a derived audit trail; there is no 2PC across SQLite and the vector store.
  - L3: "report only what persisted" and "audit after effect" are separate engineering choices; conflating them creates false receipts or missing evidence.
- counterexamples: a store may acknowledge an insert that later fails durability; a delete history failure may be repaired by reconciliation. Neither was tested.
- evidence strength: S4 for add path (two tests pass), S3 for delete path.
- reverse trace: KO-05 → EK-27/EK-28/EK-29/EK-42 → `mem0/memory/main.py:1047-1110, 2126-2144`; `tests/memory/test_main.py:1215-1259`.

## KO-06 The update receipt has three values but only two caller outcomes
- layer: **L3**
- aggregation_rule: **R1 机制簇**
- EK cluster: EK-35, EK-36, EK-37, EK-38, EK-39, EK-40
- edge types: `mechanism` (EK-35↔EK-39), `contrast` (EK-36↔EK-37, EK-38↔EK-39), `constraint` (EK-37→EK-35)
- epistemic status: **Observation / Pattern**
- statement: The contract distinguishes `False` (missing), `True` (updated), `None` (unknown). Python `_update_memory` raises only on `False`; `None` proceeds to write an `UPDATE` history row. Pinecone always returns `None`. Qdrant falls back to `upsert` on older clients, which can create a missing point. Thus "update never creates" is true for the contract and false for some implementations/fallbacks, and "unknown" is operationally indistinguishable from "success".
- why this cluster holds: six EKs converge on one mechanism (tri-state receipt) and show three implementation paths that weaken it.
- L1 → L2 → L3:
  - L1: `base.py` docstring; Qdrant `UPDATE_ONLY` + fallback; Pinecone `None`; Python/TS caller behavior.
  - L2: vector stores differ in whether they can report existence; the contract chose `None` to avoid breaking adapters.
  - L3: "a tri-state receipt needs a fourth rule for how unknown is treated" is a reusable API-contract lesson.
- counterexamples: some stores genuinely cannot distinguish; the caller could pre-read (Python does `get` before update, so Pinecone's `None` is partly mitigated by the earlier existence check — but a delete can race between `get` and `update`).
- evidence strength: S4 for Qdrant targeted tests (8 passed), S3 for Pinecone.
- reverse trace: KO-06 → EK-35…EK-40 → `mem0/vector_stores/base.py:33-40`; `qdrant.py:518-586`; `pinecone.py:315-331`; `mem0/memory/main.py:2052-2105`; `mem0-ts/src/oss/src/memory/index.ts:1990-2001`.

## KO-07 LLM output is untrusted at the identity boundary
- layer: **L3**
- aggregation_rule: **R4 主题簇**
- EK cluster: EK-26, EK-49, EK-50
- edge types: `subsystem` (EK-26→EK-49), `mechanism` (EK-49↔EK-50)
- epistemic status: **Fact / Pattern**
- statement: Mem0 treats the extraction model as an unreliable narrator of IDs. Existing memories are mapped to integer aliases; returned aliases are remapped and invalid ones discarded; facts are hashed to deduplicate. The model may decide *what* is a fact, but it must not decide *which row* to overwrite or *which scope* to write into.
- why this cluster holds: three EKs cover complementary defenses around one theme (constrained LLM authority).
- L1 → L2 → L3:
  - L1: UUID mapping, hash dedup, single-pass extraction.
  - L2: LLMs hallucinate IDs and duplicate facts; the write path must be deterministic after extraction.
  - L3: "separate semantic authority (what) from identity authority (which)" is reusable in any LLM-to-store pipeline.
- counterexamples: `memory_id` on update is caller-supplied, not LLM-supplied; this KO applies to extraction decisions, not to direct API calls.
- evidence strength: S4 (2.1.0 tests; re-read code).
- reverse trace: KO-07 → EK-26/EK-49/EK-50 → `mem0/memory/main.py` V3 pipeline; old EK-05/EK-02.

## KO-08 Deletion has four units and no shared invariant
- layer: **L3**
- aggregation_rule: **R4 主题簇**
- EK cluster: EK-11, EK-14, EK-15, EK-18, EK-41, EK-42
- edge types: `subsystem` (EK-11/EK-14/EK-15/EK-18), `contrast` (EK-18↔EK-41), `causal` (EK-42→KO-08)
- epistemic status: **Observation / Pattern**
- statement: "Delete" can mean one memory (`delete`), scoped memories (`delete_all`), every entity (`delete_users`), or the whole collection (`reset`). Each unit has its own guard, its own evidence trail, and its own failure mode. There is no `delete(scope)` primitive that can be reasoned about once.
- why this cluster holds: six EKs cover four units and two evidence paths; the theme is not "deletion" but "the absence of a single deletion algebra".
- L1 → L2 → L3:
  - L1: four public methods and four routes.
  - L2: product needs differ (single-memory UI vs tenant offboarding vs local reset), so separate methods were added incrementally.
  - L3: "destructive operations should be modeled as one algebra with explicit scope and unit" is a reusable design rule.
- counterexamples: keeping `reset` separate is reasonable because it is a collection operation, not a memory operation. The KO is about the lack of a shared explicit-scope rule, not about having four methods.
- evidence strength: S3/S4 mixed.
- reverse trace: KO-08 → EK-11/EK-14/EK-15/EK-18/EK-41/EK-42 → `mem0/memory/main.py:1883-1958, 2116-2144`; `mem0/client/main.py:528-674`; `server/main.py:515-554`.

## KO-09 History is a derived byproduct, not a transactional audit log
- layer: **L4 candidate** (cross-project validation pending)
- aggregation_rule: **R2 因果链簇**
- EK cluster: EK-28, EK-29, EK-32, EK-42, EK-43
- edge types: `causal` (EK-29→EK-28→EK-32), `contrast` (EK-28↔EK-42), `mechanism` (EK-43↔EK-32)
- epistemic status: **Pattern / Cognitive Model candidate**
- statement: Mem0's history table answers "what changed" but not "prove that the change happened atomically". On add, history is written only after persistence, and a full failure raises. On delete, the vector disappears before history is written, with no transaction. Entity links are cleaned from stored scope. The stable relation is: **the audit trail is a downstream projection of the store, not a peer transaction.**
- why L4 candidate: it is one sentence that explains add/delete/update evidence behavior across three stores and predicts reconciliation gaps after partial failures. Not yet cross-project validated.
- L1 → L2 → L3 → L4:
  - L1: `persisted_records`; `_delete_memory` order; entity cleanup from payload.
  - L2: SQLite history and vector store are separate systems; adding 2PC would hurt availability and adapter simplicity.
  - L3: "projection, not transaction" is a known event-sourcing trade-off.
  - L4: **memory audit trails are projections; their completeness depends on the write order of the primary store.**
- counterexamples: the add path is effectively "persist then project", which is safe; the delete path is "effect then project", which can lose evidence. The model must be conditioned on write order, not stated as a blanket law.
- evidence strength: S3 + S4 (add tests).
- reverse trace: KO-09 → EK-28/EK-29/EK-32/EK-42/EK-43 → `mem0/memory/main.py:1047-1110, 2107-2144`.

## KO-10 Product surface and security surface diverge
- layer: **L3**
- aggregation_rule: **R4 主题簇**
- EK cluster: EK-08, EK-17, EK-19, EK-20, EK-21, EK-51, EK-52, EK-53
- edge types: `contrast` (EK-08↔EK-17), `causal` (EK-19→EK-20), `subsystem` (EK-51/EK-52/EK-53)
- epistemic status: **Observation / Pattern**
- statement: 2.2.1 spends most of its delta on product surfaces (profiles, SDK parity, docs) while the self-hosted security surface is unchanged and partially inconsistent: the Platform client's entity vocabulary includes `app_id` while the self-hosted bulk-delete route does not; docs claim a guard that clients do not implement; tests still encode 2.1.x auth semantics. Growth and governance are not moving at the same speed.
- why this cluster holds: eight EKs cover product additions and unchanged/contradictory governance surfaces.
- L1 → L2 → L3:
  - L1: profiles added; server code unchanged; docs changed; tests stale.
  - L2: the refresh window prioritized SDK features and dependency security, not server authorization.
  - L3: "product-surface velocity can outrun security-surface coherence" is a reusable release-review heuristic.
- counterexamples: the security dependency upgrades in the same window show security was not ignored; it was dependency security, not authorization design.
- evidence strength: S3 + S4.
- reverse trace: KO-10 → EK-08/EK-17/EK-19/EK-20/EK-21/EK-51/EK-52/EK-53.

## Layer distribution

| Layer | Count | KOs |
|---|---|---|
| L3 Pattern | 8 | KO-01, KO-03, KO-04, KO-05, KO-06, KO-07, KO-08, KO-10 |
| L4 candidate | 2 | KO-02, KO-09 |
| L5 | 0 | not claimed |
| L1/L2 only | 0 | retained in EK layer |

## Aggregation coverage

- R1 mechanism cluster: KO-02, KO-06
- R2 causal chain: KO-05, KO-09
- R3 invariant cluster: KO-01, KO-03, KO-04
- R4 theme cluster: KO-07, KO-08, KO-10
- Coverage: 10/10 = 100%
- KOs with <3 EK: none
- KOs whose only reason is "same subsystem": none

## Reverse trace integrity

Every KO lists its EK cluster and the EK section lists evidence paths. The corpus schema check will verify that every `KO-xx` referenced here appears in `02_engineering_knowledge.md` and every `EK-xx` in a cluster exists. No KO is allowed to cite only itself or only a document.

## Explicit non-claims

- No L5 methodology is claimed. Mem0 is one project; "scope has no single authority" and "history is a projection" are L4 **candidates**, not principles.
- "Authentication ≠ authorization" is stated as a server observation, not a universal law; single-tenant deployments are a legitimate countercase.
- "delete_all is unsafe" is not claimed. The claim is narrower: **the empty-scope rule is inconsistent across surfaces**.

# Reconciliation — Mem0 2.2.1 refresh

- run_id: `ARCH-2026-11-17-001`
- input: `06_validation/independent-validation.md` (32 findings)
- rule: the independent report does not edit the original package; this file records the adjudication, and the package is then corrected. Corrections never change the production skill.

## 1. Adjudication table

| Finding | Status | Decision | Package action |
|---|---|---|---|
| IF-01 commit/version/files | CONFIRMED | accept | none |
| IF-02 Python OSS `add` has no filters | CONFIRMED | accept | none |
| IF-03 TS `filters` is not the only source | **CONTRADICTED** | accept; EK-06/EK-07 over-broad | rewrote EK-06 and EK-07 to "two grammars / `filters` is a second channel" |
| IF-04 Platform add rejects filters | CONFIRMED | accept | none |
| IF-05 `app_id` client vs OSS | CONFIRMED | accept | none |
| IF-06 Platform vs self-hosted are two API families | **PARTIALLY_CONFIRMED** | accept; EK-17/C-10 framed as one call path | rewrote EK-17; rewrote 01 §1 and §7.4; rewrote C-10 |
| IF-07 OSS delete_all raises | CONFIRMED | accept | none |
| IF-08 TS deleteAll raises | CONFIRMED | accept | none |
| IF-09 Platform delete_all sends empty | CONFIRMED | accept | none |
| IF-10 docs claim raise | CONFIRMED | accept | none |
| IF-11 delete_users no-arg all | CONFIRMED | accept | none |
| IF-12 wildcard literal | PARTIALLY_CONFIRMED | accept; keep as hypothesis | C-04 unchanged (already hypothesis) |
| IF-13 Phase 6 persisted records | CONFIRMED | accept | none |
| IF-14 all-fail raises | CONFIRMED | accept | none |
| IF-15 history from persisted | CONFIRMED | accept | none |
| IF-16 delete then history | CONFIRMED | accept | none |
| IF-17 tri-state base | CONFIRMED | accept | none |
| IF-18 Pinecone None | CONFIRMED | accept | none |
| IF-19 Qdrant fallback | CONFIRMED | accept | none |
| IF-20 per-memory auth only | CONFIRMED | accept | none |
| IF-21 bulk/reset admin | CONFIRMED | accept | none |
| IF-22 ADMIN_API_KEY empty | CONFIRMED | accept | none |
| IF-23 server tests stale | CONFIRMED | accept | none |
| IF-24 server app code unchanged | CONFIRMED | accept | none |
| IF-25 retrieval unchanged | CONFIRMED | accept | none |
| IF-26 counts | CONFIRMED | accept | none |
| IF-27 KO rules / no L5 | CONFIRMED | accept | none |
| IF-28 "three grammars" | **DOWNGRADED** | accept; change to two grammars, three surfaces | rewrote KO-02 statement and L1 ladder; fixed 00 and XP-01 |
| IF-29 no single authority | PARTIALLY_CONFIRMED | accept as L4 candidate only | KO-02 already labeled candidate; wording tightened |
| IF-30 Platform/self-hosted API family omission | **MISSING** | accept as structural omission | added to 01 §1 and EK-17 |
| IF-31 Facts=47 unsourced | **MISSING** | accept; remove the unverified metric | 00 now reports "EK statements = 55" |
| IF-32 app_id runtime evidence | **MISSING** | accept; keep as runtime gap | C-10 remains "Needs Owner"; no runtime claim |

## 2. Decisions that changed the package

### D-01 EK-06 / EK-07 — scope grammar
- Before: "three SDK surfaces define scope with different grammars"; "TS filters is the only scope source".
- After: "two grammars across three surfaces"; TS accepts top-level **or** filters; Python accepts top-level only; Platform rejects filters explicitly.
- Reason: `mem0-ts/src/oss/src/memory/index.ts:781-787` writes top-level `userId` into `filters`, so top-level is a source; `index.ts:788-793` reads `filters.user_id` as a source. Both are true.

### D-02 EK-17 / 01 / C-10 — API families
- Before: "platform client call to the self-hosted route has no contract".
- After: "Platform and self-hosted are two API families"; `MemoryClient` calls `/v1/...`; self-hosted serves `/memories`; the `app_id` gap is a cross-contract parity gap.
- Reason: `server/main.py:322-557` has no `/v1` prefix; `mem0/client/main.py:1360` uses `/v1/ping/`. The original wording implied one client speaking to one server, which is not supported.
- Impact: KO-10's "product vs security surface" claim is narrowed, not invalidated.

### D-03 KO-02 — L4 candidate
- Keep L4 candidate, not L3 and not Principle.
- Change "three grammars" to "two grammars across three surfaces".
- Keep the prediction: a fourth surface will create a fourth grammar unless a shared schema exists.
- This is a **scope tightening**, not a downgrade to L2.

### D-04 Metrics
- Remove "Facts / observations = 47" because no enumeration existed.
- Report "EK statements = 55" (each EK is a fact or observation).
- Update independent validation counts to 24/3/1/1/3.

### D-05 Minor additions
- EK-42 now records that `delete_all` emits N DELETE rows, not one aggregate event.
- Candidate C-10 risk changed from "client-valid call degrades to 400" to "API-family parity gap".

## 3. What was not changed

- No EK or KO was deleted.
- No L4 candidate was promoted to Principle.
- No Hypothesis (C-01, C-04, C-06, C-07, C-11) was converted to Fact.
- The validation results (including contradictions) are preserved in `independent-validation.md`; reconciliation did not rewrite them.
- The production skill was not modified.

## 4. Post-reconciliation consistency check

| Check | Result |
|---|---|
| every CONTRADICTED finding has a package edit | yes (IF-03 → EK-06/07) |
| every DOWNGRADED finding has a package edit | yes (IF-28 → KO-02, 00, XP-01) |
| every MISSING finding is either added or left as a declared gap | yes (IF-30 added; IF-31 metric removed; IF-32 left as runtime gap) |
| no "three grammars" text remains | yes (grep clean after edit) |
| no "Facts=47" metric remains | yes |
| KO layer counts still 8 L3 + 2 L4 | yes |
| EK count still 55, links still 157 | yes |

## 5. Final validation verdict

**PARTIALLY_CONFIRMED → reconciled to ACCEPTED WITH LIMITATIONS.**

- Core E/F findings survive: scope hardening, destructive empty-scope divergence, authn ≠ authz, receipt tri-state, history projection.
- Two wording defects are corrected; one structural omission is added; one metric removed.
- Limits remain: no live server/vector store/LLM/Postgres; TS tests not executed; 23 old EKs not re-evidenced.

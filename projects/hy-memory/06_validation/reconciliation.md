# Reconciliation — Hy-Memory 2.0

- run_id: `ARCH-2026-11-20-001`
- reconciliation performed: 2026-11-21
- source of corrections: `06_validation/independent_validation_report.md`
- rule: corrections are applied to this Corpus Artifact; the production Skill and the target repository are not modified

## Applied corrections

| ID | Original statement | Corrected statement | Reason |
|---|---|---|---|
| CF-04 | classifier runs when scene is None and tool messages exist | classifier also requires `content_type=="messages"`, mode != lite, and coding config present/enabled | auditor found input-shape and feature-flag gates missing |
| CF-05 | chat fallback on classifier exception or unknown result | also covers explicit `scene="coding"` with no tool message | code falls back to chat for that case |
| CF-07 | coding writer when resolved scene is coding and category != raw | `coding_available` (mode != lite, messages input, `config.coding.enable`) and (`scene_forced=="coding"` or tool message) and `is_coding`; explicit coding without tools falls back to chat | auditor found routing edge oversimplified |
| EK-24 / AF-04 | exact `user::agent` key for “one agent, no sessions” | exact key applies when `request.agent_id` is set and neither `agent_ids` nor `session_ids` is supplied; a one-element `agent_ids` uses `user_ids` + `agent_ids` filtering | `agent_ids` list and `request.agent_id` are different branches |
| EK-04 | L5_KNOWLEDGE is unimplemented | project comment says “暂未实现”; no engine write path found; product-wide guarantee not proven | code comment is not product-level evidence |
| KO-06 | no purge path | no purge path found in the audited engine; `memhub/` desktop behavior not audited | scope boundary |
| Candidate notes | — | added schema-dedup branch in `create_graph_node` and `add_evidence` increment of `s2_evidence_count` | auditor omission |

## Validation verdicts retained

- KO: 9 CONFIRMED / 1 PARTIALLY_CONFIRMED / 0 DOWNGRADED / 0 OVER_GENERALIZED / 0 CONTRADICTED / 0 NEEDS_HUMAN_REVIEW
- EK spot checks: 10 CONFIRMED / 2 PARTIALLY_CONFIRMED / 0 CONTRADICTED
- Flow spot checks: 6 CONFIRMED / 2 PARTIALLY_CONFIRMED / 0 CONTRADICTED
- Flow edge accuracy: 135/143 confirmed, 4 partially confirmed, 0 contradicted, remaining spot checks confirmed

## Counterexamples and open contradictions

- No contradicted edge or KO was found.
- The L4 candidate model is retained as a **candidate**, not promoted.
- Counterexample to any broad “tenant isolation” claim: public search enforces `user_ids`, but the Qdrant filter builder adds no user condition when no user/isolation parameter exists, and `get/update/delete` are ID-only. This is a defense-in-depth gap, not a proven public-API cross-tenant exploit.
- Counterexample to “deletion is complete”: `history.db` records full old content and delete re-records it; `delete_all` does not purge prior audit rows.
- Counterexample to “documented capability equals implemented capability”: System 2 headers state 10 and 8 tools while the JSON schema exposes 9; two documented tools do not exist.

## Final result

`ACCEPTED WITH LIMITATIONS`

The package remains source-traceable and conservative in abstraction. Limitations are recorded in `run_metadata.yaml`: no behavioral model run, `memhub/` not audited, shallow clone, untraced vendor benchmark figures, and an inconclusive full test suite.

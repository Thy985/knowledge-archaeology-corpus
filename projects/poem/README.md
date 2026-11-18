# PoEM — Proof-of-Execution Memory

PoEM is a research prototype and paper artifact asking one question: **should an agent trust a memory claim that a safety step was already executed, or require an execution receipt?**

Its answer is a separate HMAC-chained `ExecutionLedger`, written only after the runtime selects the safe action. A decision gate vetoes a configured unsafe action when the ledger has no matching `(action, subject)` event. The repository also contains a FARMA-comparison harness (six defenses × three scenarios × two attack families), a PoEM-aware attack script, a false-positive experiment, an overhead benchmark, and a LangChain port. `injection_lab/` is a separate research line and is out of scope for this run.

## Archaeology snapshot

| Field | Value |
|---|---|
| Repository | `https://github.com/bithabib/ai_security.git` |
| Commit | `4021642395daad997786cfbbd04b64924e8a46bd` (main, 2026-08-28) |
| Mode | initial |
| Run | `ARCH-2026-11-19-001` |
| Skill | knowledge-archaeology v3.2 |
| Scope | `farma_repro/` + `poem-paper/` (design intent); `injection_lab/` excluded |
| License | MIT |

## Quality metrics

| Metric | Value |
|---|---|
| Engineering Knowledge | 49 (EK-01…EK-49) |
| EK links | 156; avg 3.18; isolated 0; dangling 0 |
| Knowledge objects | 10 (9 L3 + 1 L2 observation); 0 L4; 0 L5 |
| Flow edges | 49 (7/7 classes) |
| Candidates | 14 |
| Counterexamples | 10 |
| Independent verdict | 7 CONFIRMED / 2 DOWNGRADED / 1 NEEDS_HUMAN_REVIEW (KO); 11 CONFIRMED / 1 PARTIALLY_CONFIRMED (EK sample) |
| Overall | **ACCEPTED WITH LIMITATIONS** |

## What survives

1. **Separation of stores.** Reasoning memory and execution accounting are different objects with different writers and integrity rules (EK-01…EK-14).
2. **Gate coverage is the security boundary.** PoEM checks the wired unsafe action only; out-of-scope actions pass. A narrow gate set cannot protect an action it does not name (KO-03, KO-04).
3. **Proof identity matters.** `(action, subject)` binding plus chain verification blocks the cross-subject replay that defeats a naive "any event with this step" check (KO-08).
4. **Integrity is key custody plus write authority, not the hash alone.** The default key, public `record()`/`clear()`, and acceptance of duplicate/out-of-order events widen the trust boundary (KO-02, EK-49).
5. **Verification cost is O(N).** Measured 18.3 µs at N=10 and 173.8 ms at N=100,000; the paper's microsecond claim is only true at small history (KO-10).

## What does not survive as generalized knowledge

- **KO-02 and KO-05 were L4 candidates; both are L3 after independent blind reconstruction.** The code supports the mechanisms, but there is no second project or production call site supporting a cognitive-model claim.
- **KO-09 remains an L2 observation.** Model-capability-vs-ASR numbers are author-recorded S3 files; no live model was rerun. It needs human review, not promotion.
- **Mock numbers are plumbing evidence.** The README says explicitly they are not scientific results; mock behavior is constructed, so mock ASR/FP cells cannot validate the research claim.
- **The LangChain port is not equivalent to the harness.** It retrieves `page_content` only and hardcodes `source_label="AGENT"`, dropping the EXTERNAL signal (EK-47); it was not run.

## Deliverables

- `00_overview.md` — one-page project and findings
- `01_project-layer/project_layer.md` — architecture, lifecycle, entry points, data structures
- `02_engineering-knowledge/ek_graph.md` — 49 EK with six edge types
- `03_knowledge-layer/knowledge_layer.md` — 10 KO with R1–R4 aggregation rules
- `04_flow-atlas/flow_atlas.md` — 49 edges across Control/State/Data/Evidence/Authority/Memory/Policy
- `05_candidates/candidates.md` — 14 unverified hypotheses
- `06_validation/` — self-validation, independent blind reconstruction, reconciliation, evidence index
- `archaeology-runs/ARCH-2026-11-19-001/` — immutable run copy (package + job manifest + snapshot)

## Reproduction boundary

All local probes ran on copies in `/tmp`; the target repository was not modified. Real-model reproduction requires Ollama + `llama3.1:8b` or an OpenAI key, neither available in this environment. The `results*.json` files are treated as author evidence (S3), never as S4/S5.

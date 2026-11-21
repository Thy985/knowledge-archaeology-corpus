# 04 Flow Atlas — PoEM

- run_id: `ARCH-2026-11-19-001`
- source: code at `4021642395daad997786cfbbd04b64924e8a46bd`; not inferred from paper alone
- edge schema: `id | from → to | condition | symbol / file | evidence`
- seven classes: Control / State / Data / Evidence / Authority / Memory / Policy

## 1. Control Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| CF-01 | `Agent.run_task` → `defense.filter_memory` | defense has `filter_memory` | `agent.py:49-53` | code |
| CF-02 | `Agent.run_task` → `memory.as_prompt_text` | no `filter_memory` | `agent.py:54-58` | code |
| CF-03 | `Agent.run_task` → `scenario.build_prompt` | always, after memory text chosen | `agent.py:58`; `scenarios.py:42-50` | code |
| CF-04 | `scenario.build_prompt` → `llm.decide` | always | `agent.py:60`; `llm.py:114` | code |
| CF-05 | `Agent.run_task` → `defense.authorize_decision` | defense has the method | `agent.py:62-66` | code |
| CF-06 | `PoEM.authorize_decision` → `safe_action` | `action == unsafe_action` and `contains_event(...)` false | `defenses.py:83-89` | code |
| CF-07 | `Agent.run_task` → `_execute_safety_step` | final action == `scenario.safe_action` | `agent.py:67-70` | code |
| CF-08 | `_execute_safety_step` → `ledger.record` | safe action selected | `agent.py:42-44`; `memory.py:97` | code |
| CF-09 | `Agent.run_task` → caller | always, returns final action | `agent.py:71` | code |

## 2. State Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| SF-01 | `Agent.step = 0` → `step + 1` | a safe action executes | `agent.py:36, 43` | code |
| SF-02 | empty ledger → first event | first `record()` call | `memory.py:97-101` | code |
| SF-03 | last event MAC / `"GENESIS"` → `prev` | every append | `memory.py:99` | code |
| SF-04 | LLM-proposed action → final action | `authorize_decision` returns a value | `agent.py:62-66` | code |
| SF-05 | memory text → `entries` append | `write()` called | `memory.py:47-57` | code |
| SF-06 | subject with no hit → all entries | `retrieve()` fallback | `memory.py:59-64` | code |
| SF-07 | fresh Agent per trial → empty memory/ledger | each experiment trial | `experiment.py` trial loop | code |

## 3. Data Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| DF-01 | reasoning entry text → prompt memory block | always (filtered or raw) | `memory.py:66-70`; `agent.py:54-58` | code |
| DF-02 | scenario task + subject → prompt | always | `scenarios.py:42-50` | code |
| DF-03 | LLM raw content → parsed action | response received | `llm.py:92-112` | code |
| DF-04 | parse failure → `safe_action` | JSON error / missing / unknown action | `llm.py:112` | code |
| DF-05 | all entries → subject-matched subset | `retrieve(subject)` | `memory.py:59-64` | code |
| DF-06 | `action, subject, step, prev` → MAC | every `record()` | `memory.py:93-95` | code |
| DF-07 | event list → `True/False` | `verify_chain()` | `memory.py:110-119` | code |
| DF-08 | per-cell successes / trials → Wilson interval | experiment reporting | `experiment.py:41-49` | code |

## 4. Evidence Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| EF-01 | clean run → safe action | empty memory, baseline behavior | `experiment.py` attack-success loop | code |
| EF-02 | poisoned run → unsafe action | attack succeeds | `experiment.py` attack-success loop | code |
| EF-03 | genuine execution event → legitimate skip | benign agent has a real ledger event | `fp_experiment.py:45-53` | code |
| EF-04 | forged event appended without valid chain → `verify_chain() == False` | attacker edits/append without key | `memory.py:110-119`; `attack_poem.py:155-164` | code + local run |
| EF-05 | `parse_stats` → coercion rate | unparseable/unknown replies | `llm.py:83-90`; `experiment.py` | code |
| EF-06 | results JSON → Markdown table / figure | post-processing script | `compare_models.py`, `figures.py` | code |
| EF-07 | paper claim (µs overhead) → measured 173.8 ms at 100k | benchmark at large N | `overhead_test.py`; paper §Overhead | code + local run (contradiction surfaced) |

## 5. Authority Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| AF-01 | `AUTHORIZED_WRITERS["AGENT"]` → default memory signer | `signer="default"` | `memory.py:19-25, 45` | code |
| AF-02 | `AUTHORIZED_WRITERS["PLUGIN"]` → attack note signature | `inject_variant(..., signer="plugin")` | `agent.py:13-27` | code |
| AF-03 | `signer="unauthorized"` → unsigned entry | explicit attack option | `agent.py:21-25` | code |
| AF-04 | any holder of ledger object → `record()` | no capability check | `memory.py:97-101` | code |
| AF-05 | execution ledger → PoEM veto authority | `contains_event` true/false | `defenses.py:83-89` | code |
| AF-06 | writer + HMAC signature → SignedMemory keep/drop | signature valid and writer authorized | `defenses.py:193-204` | code |
| AF-07 | constructor/default literal → ledger MAC authority | no explicit key passed | `memory.py:89-90`; `agent.py:37-39` | code + local default-key forgery run |

## 6. Memory Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| MF-01 | `write(text, label, signer)` → `entries` | any caller | `memory.py:47-57` | code |
| MF-02 | `entries` → `retrieve(subject)` result | lowercase substring match | `memory.py:59-64` | code |
| MF-03 | `clear()` → empty `entries` | explicit call | `memory.py:72-74` | code |
| MF-04 | `record(...)` → append `events` | any caller | `memory.py:97-101` | code |
| MF-05 | `clear()` → empty `events` | explicit call | `memory.py:121-122` | code |
| MF-06 | process/Agent exit → all memory and ledger state lost | no persistence code | `agent.py:30-40`; `memory.py:34-122` | code |

## 7. Policy Flow

| id | from → to | condition | symbol / file | evidence |
|---|---|---|---|---|
| PF-01 | scenario policy text → prompt | each scenario | `scenarios.py:20-50` | code |
| PF-02 | PoEM policy (memory cannot authorize skip) → decision veto | unsafe action without ledger event | `defenses.py:76-89`; paper §Method | code + design intent |
| PF-03 | Sentinel threshold 0.4 → entry dropped / empty marker | score ≥ threshold / all dropped | `defenses.py:41-74` | code |
| PF-04 | gate set → protected action coverage | action in `unsafe_action` only | `defenses.py:83-89`; `policy_test.py:71-95` | code + deterministic run |
| PF-05 | genuine ledger + genuine note → legitimate skip allowed | benign experiment | `fp_experiment.py:45-105` | code |

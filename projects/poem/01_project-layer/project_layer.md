# 01 Project Layer — PoEM

- run_id: `ARCH-2026-11-19-001`
- scope: `farma_repro/` + `poem-paper/` evidence relevant to PoEM; `injection_lab/` excluded
- rule: Project Layer records project facts (L0). Design intent from paper/notes is marked separately from implementation.

## 1. What the project is

PoEM is a research prototype of a memory-defense mechanism for LLM agents. The agent’s ordinary reasoning memory (`ReasoningMemory`) can be poisoned with forged notes claiming a required safety step was already done. PoEM adds a second store, `ExecutionLedger`, that records safety steps that actually executed, and a decision-time gate that refuses an “unsafe skip” unless the ledger contains a matching event.

The repository is not a packaged library. It is a collection of research artifacts:
- a minimal, dependency-free FARMA/PoEM harness;
- a paper source tree;
- a separate prompt-injection research line;
- reference papers and working notes.

## 2. Repository map

| Path | Files / lines | Role | Used as PoEM evidence? |
|---|---|---|---|
| `farma_repro/memory.py` | 122 | Two stores: `ReasoningMemory`, `ExecutionLedger` | Yes |
| `farma_repro/agent.py` | 71 | Scenario-driven agent; decision → gate → execute | Yes |
| `farma_repro/defenses.py` | 204 | NoDefense, Sentinel, PoEM, NaivePoEM, AlwaysExecute, AnomalyDetection, SignedMemory | Yes |
| `farma_repro/attack.py` | 102 | 3 basic + 7 adaptive forged-reasoning variants | Yes |
| `farma_repro/scenarios.py` | 238 | EHRAgent / ReAct-QA / RAP; prompts, safe/unsafe actions, benign/reference notes | Yes |
| `farma_repro/llm.py` | 208 | mock / ollama / openai clients; action coercion and parse stats | Yes |
| `farma_repro/experiment.py` | 155 | 3 scenarios × 2 attack families × 6 defenses; ASR + Wilson CI + coercion | Yes |
| `farma_repro/fp_experiment.py` | 138 | Benign legitimate-skip / false-positive experiment | Yes |
| `farma_repro/attack_poem.py` | 178 | PoEM-aware attacks: ledger forgery, cross-subject replay, reframing, out-of-scope (doc only) | Yes |
| `farma_repro/policy_test.py` | 114 | Judgment/reframing test + coverage demonstration | Yes |
| `farma_repro/overhead_test.py` | 138 | Grounding-check / verify-chain latency at ledger sizes 10…100000 | Yes |
| `farma_repro/langchain_port.py` | 242 | Real LangChain vector-store + ChatOpenAI port | Yes (code read; not executed) |
| `farma_repro/auto_attacker.py` | 158 | LLM-driven adaptive attacker loop | Yes (code read; mock pool) |
| `farma_repro/calibrate_anomaly.py` | 37 | Threshold calibration on genuine notes only | Yes |
| `farma_repro/run_sweep.py` | 48 | Multi-model sweep driver | Yes |
| `farma_repro/figures.py` / `baseline_figures.py` / `compare_models.py` / `fp_figures.py` / `auto_attacker_summary.py` | 152/141/142/104/75 | Figure/table generation | Yes (secondary) |
| `poem-paper/` | TeX/PDF | Design intent, threat model, results claims | Intent only |
| `injection_lab/`, `injection-paper/` | 1,028 Python lines + TeX/PDF | Separate prompt-injection study | No |

## 3. Architecture

```
                       attacker-writable
                     ┌──────────────────┐
                     │ ReasoningMemory  │  entries: text, source_label, writer, sig
                     └────────┬─────────┘
                              │ retrieve / as_prompt_text
                              ▼
   Scenario.policy + task + memory text  ──►  LLMClient.decide
                              │
                              ▼ proposed action
                     ┌──────────────────┐
                     │ defense          │
                     │ filter_memory?   │  (NoDefense / Sentinel / Anomaly / Signed)
                     │ authorize_decision? │ (PoEM / NaivePoEM / AlwaysExecute)
                     └────────┬─────────┘
                              │ final action
                 safe_action ─┴─ unsafe_action / other
                     │                    │
                     ▼                    ▼
            _execute_safety_step      return without execution
                     │
                     ▼
             ExecutionLedger (HMAC chain)   ← trusted action layer only (design intent)
```

Key ordering: memory filtering happens **before** prompt construction; decision-time authorization happens **after** the LLM proposes an action and **before** any execution (`agent.py:46-71`).

## 4. Core abstractions

| Abstraction | Responsibility | Key fields / methods |
|---|---|---|
| `Scenario` | Bundles policy prompt, task template, safe/unsafe action names, safety-step name, variant families, mock hints, benign/reference notes | `build_prompt()`, `attack_families()` |
| `ReasoningMemory` | Attacker-writable “notebook”; optional HMAC signature naming the writer | `entries`, `write()`, `retrieve()`, `as_prompt_text()`, `clear()` |
| `ExecutionLedger` | Append-only, HMAC-chained record of executed safety steps | `events`, `record()`, `contains_event()`, `verify_chain()`, `clear()` |
| `Agent` | Runs one task; applies defense; executes safe action; writes ledger | `run_task()`, `_execute_safety_step()` |
| Defense | Either `filter_memory(memory, subject)` or `authorize_decision(...)`, or both | `NoDefense`, `Sentinel`, `PoEM`, `NaivePoEM`, `AlwaysExecute`, `AnomalyDetection`, `SignedMemory` |
| `LLMClient` | Backend abstraction (mock/ollama/openai); action parsing and coercion accounting | `decide()`, `_coerce_action()`, `parse_stats` |
| Experiment driver | Creates fresh Agent per trial; injects variants; aggregates ASR/CI/coercion | `experiment.py`, `fp_experiment.py`, `attack_poem.py` |

## 5. Lifecycle

1. `Agent.__init__` constructs a fresh `ReasoningMemory` and `ExecutionLedger`; binds scenario-aware defenses (`AnomalyDetection`).
2. Optional attack injection writes forged notes into `ReasoningMemory` before `run_task`.
3. `run_task(subject)`:
   - retrieve memory (or defense-filtered memory);
   - build prompt from scenario policy + task + memory text;
   - call `llm.decide(...)`;
   - optional `authorize_decision(...)`;
   - if final action == safe action, increment `step` and append ledger event;
   - return action.
4. The ledger and memory die with the Agent instance. There is no persistence, recovery, concurrency control, or cross-session reuse.

## 6. State stores

| Store | Location | Format | Writer | Persistence |
|---|---|---|---|---|
| Reasoning memory | `Agent.memory.entries` | Python list of dicts `{text, source_label, writer, sig}` | Agent + attacker (in threat model); `write()` is public | In-memory, per Agent |
| Execution ledger | `Agent.ledger.events` | Python list `{action, subject, step, prev, mac}` | `record()`; design says trusted action layer only | In-memory, per Agent |
| LangChain vector store | `InMemoryVectorStore` in `langchain_port.py` | Embeddings + documents | `remember()` / `add_texts()` | In-memory, per agent |
| Experiment outputs | `results*.json`, `fp_results*.json`, `*_table.md` | JSON / Markdown | Experiment scripts | Files in repo (author-produced) |

## 7. Entrypoints and run modes

| Entrypoint | Backend | Writes | Purpose |
|---|---|---|---|
| `python3 experiment.py` | mock (default) | `results.json` (unless `--out`) | 36-cell ASR matrix, Wilson CI, coercion |
| `python3 experiment.py --backend ollama --model llama3.1:8b` | local Ollama | results JSON | real-model ASR |
| `python3 experiment.py --backend openai --model gpt-4o-mini` | OpenAI | results JSON | real-model ASR |
| `python3 fp_experiment.py` | mock/ollama/openai | `fp_results.json` | benign legitimate-skip / FP |
| `python3 attack_poem.py` | mock/ollama/openai | `poem_adversarial_table.md`, figures | PoEM-aware attacks |
| `python3 policy_test.py` | mock/ollama/openai | stdout only | judgment claims + coverage demo |
| `python3 overhead_test.py` | mock/openai | `overhead_table.md`, figures | ledger cost |
| `python3 langchain_port.py` | OpenAI + LangChain | `langchain_results.json`, table, figure | real framework port |
| `python3 auto_attacker.py` | mock/ollama/openai | `auto_attacker_results.json` | optimizing attacker |

## 8. Test system (what exists, what does not)

- **No** `pytest`/`unittest` suite, no `tests/` directory, no CI workflow, no coverage configuration.
- The scripts are research experiments, not assertions. They print tables and write JSON.
- Deterministic checks that do exist:
  - `attack_poem.py` checks `verify_chain()` detects a forged event (attack A) and that a cross-subject genuine ledger remains valid.
  - `policy_test.py` Part B is a deterministic gate-coverage demonstration.
  - `experiment.py` records `parse_stats` and coercion rate.
- `README.md` explicitly says the mock backend is a “logic/plumbing demonstration, NOT scientific results”; real evidence needs a model.

## 9. Configuration and dependencies

| Item | Fact |
|---|---|
| Core runtime | Python standard library (`hashlib`, `hmac`, `json`, `re`, `urllib`, `time`) |
| Optional figures | `matplotlib` |
| Ollama backend | `http://localhost:11434`, default model `llama3.1:8b` |
| OpenAI backend | `OPENAI_API_KEY` env var or `open_ai_api_key` / `openai_api_key` file found by walking up from cwd/file dir (`llm.py:52-68`) |
| LangChain port | `langchain_core`, `langchain_openai`, `OpenAIEmbeddings(text-embedding-3-small)`, `ChatOpenAI(gpt-4o-mini)`, Python 3.12 |
| Lockfile / version pinning | None |
| Container / deployment config | None |

## 10. Permission and governance mechanisms

| Mechanism | Implementation | Boundary |
|---|---|---|
| Memory writer identity | `AUTHORIZED_WRITERS = {"AGENT": b"agent-writer-key", "PLUGIN": b"plugin-writer-key"}`; `sign_entry()` HMAC over `writer|text` | Proves who wrote, not whether content is true |
| Ledger write | `ExecutionLedger.record()` | No capability token; any caller with the object can append |
| Ledger integrity | `verify_chain()` recomputes `HMAC(action|subject|step|prev)` and compares with `hmac.compare_digest` | Detects byte-level edits; does not protect against a key-holder or trusted-process attacker |
| Ledger key | Default literal `b"trusted-runtime-key"`; constructor allows override | Caller must inject and isolate a real secret; code does not require it |
| Decision gate | `PoEM.authorize_decision()` only checks `unsafe_action` | Other harmful actions are out of scope until gated |
| Threat model | Paper §Threat Model: attacker can write memory, cannot modify code or observe execution | If attacker can invoke genuine action path or obtain key, guarantee collapses (paper limitation 5) |

## 11. External dependencies and scope boundaries

- The PoEM harness has no vector DB, no server, no network service; `ollama`/`openai` are the only network clients.
- The LangChain port is an adapter, not a middleware: it calls `ledger.contains_event()` manually and does not wrap arbitrary tool calls.
- `injection_lab/` has its own README and experiment scripts. It is not imported by `farma_repro`; its results must not be attributed to PoEM.
- Paper PDFs and working notes are design-intent evidence; implementation facts come from code.

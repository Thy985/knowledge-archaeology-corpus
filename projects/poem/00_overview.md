# 00 Overview — PoEM (ARCH-2026-11-19-001)

## One-line positioning

PoEM（Proof-of-Execution Memory）是一套针对 LLM Agent “伪造已完成推理”（forged reasoning）的防御：它不检查攻击者可写的推理记忆文本，而是维护一份仅由可信动作层写入的 HMAC 链式执行台账；只有当台账中存在“该 subject 的安全步骤确实执行过”的事件时，Agent 才被允许跳过该步骤。

仓库 `bithabib/ai_security` 同时包含 PoEM/FARMA 复现 harness（`farma_repro/`）、IEEE Access 论文源（`poem-paper/`）、另一条 prompt-injection 研究线（`injection_lab/`、`injection-paper/`）与参考论文。本 run 的 scope 只覆盖 **PoEM/FARMA 研究线**；`injection_lab` 不作为 PoEM 的实现证据。

## Why this project was selected

- KnowlegeMap 11-19 雷达把它列为 **A 级新锚点**：F 层“执行回执/执行事实”防御，直接对应 Memory 第二周的“从检测记忆到校验执行”。
- 论文 `2608.16032v2` 与 MIT 代码同时存在；`farma_repro/*.py` 2,767 行，核心 harness 仅依赖 Python 标准库，可在无 GPU/无 API key 条件下完成逻辑与路径取证。
- Corpus 此前无 `poem` 项目；本轮为 initial 考古。
- 与既有 Corpus 的连接：Mem0（记忆层 scope/写路径）、AMG（记忆投毒检测）提供 E/F 层对照；PoEM 把问题从“识别恶意文本”推进到“验证真实执行”。

## Repository facts

| Item | Value |
|---|---|
| repository | `https://github.com/bithabib/ai_security.git` |
| branch | `main` |
| HEAD | `4021642395daad997786cfbbd04b64924e8a46bd` |
| commit time | `2026-08-28 23:30:36 +0900` |
| commit subject | `Checkpoint: baseline defenses, multi-model results, and injection paper` |
| license | MIT（Copyright (c) 2026 Md Habibur Rahman） |
| tracked files | 253 |
| Python | 49 files / 4,712 lines |
| PoEM harness | `farma_repro/*.py` 2,767 lines |
| version / tag | 无 `pyproject.toml` / `requirements.txt` / 版本号 / tag |
| skill | knowledge-archaeology v3.2 |

## Core claim (as stated by the repository)

> Do not trust a memory that says a safety step happened; verify it against an independent, tamper-evident record of what actually executed.

Three-step narrative in `farma_repro/README.md`:
1. **VERIFY** — basic FARMA succeeds under No Defense; SENTINEL stops it.
2. **BREAK** — adaptive/reworded attack gets past SENTINEL.
3. **FIX** — PoEM blocks basic and adaptive attacks.

## Top findings

1. **The defense’s core separation is real in code, not just prose.** `ReasoningMemory` is attacker-writable and has no access control; `ExecutionLedger` is a separate append-only HMAC chain; `PoEM.authorize_decision()` consults only the ledger for the `unsafe_action`, never the memory text (`memory.py:34-122`, `defenses.py:76-89`, `agent.py:46-71`).

2. **The tamper-evidence guarantee is a deployment invariant the code does not enforce.** `ExecutionLedger.__init__` ships `secret_key=b"trusted-runtime-key"` as a literal default; `record()` is a public method with no capability check; `clear()` is public; the ledger is in-memory per trial. A byte-level chain rewrite is detected by `verify_chain()`, but an attacker who can execute in the trusted process or use the default key can mint a valid chain. The paper names the key-isolation limitation; the implementation does not enforce it (`memory.py:89-121`, `poem-paper/body.tex:637-639`).

3. **PoEM is a veto on one action, not a policy engine.** `authorize_decision` only triggers when `action == unsafe_action`; every other action passes through. `policy_test.py` Part B demonstrates that an ungated harmful action (`export_to_partner`) succeeds with narrow gates and is blocked only when added to the gate set (`defenses.py:83-89`, `policy_test.py:71-95`).

4. **The proof identity is `(action, subject)` only — no step/task/nonce/expiry.** `record()` stores `step`, but `contains_event()` does not compare it; a past `validate` event for subject `X01` authorizes a later, unrelated skip for the same subject. The paper explicitly lists freshness as a limitation; the code has no freshness mechanism (`memory.py:97-108`, `poem-paper/body.tex:626-628`).

5. **The evaluation separates attack success from parser failure — a genuine methodological strength.** `LLMClient._coerce_action()` forces unparseable replies to the safe action and records `json_error` / `missing_action` / `bad_action`; `experiment.py` reports a coercion rate alongside ASR. Without this, weak-model formatting failures would be misread as resistance (`llm.py:92-112`, `experiment.py:107-118`).

6. **The default mock path is plumbing, not evidence.** `README.md` states “mock numbers are a logic/plumbing demonstration, NOT scientific results.” The mock returns unsafe iff a scenario hint phrase appears; `policy_test.py` Part A’s judgment claims contain none of the mock hints, so the default mock run reports 0% ASR even for No Defense. Judgment-claim results require a real model (`README.md:26-31`, `llm.py:127-136`, `policy_test.py:28-52`).

7. **Recorded results show the capability paradox, but it is not independently reproduced in this run.** Repository `results_gpt4o.json` / `results_llama8b.json` record adaptive ASR 94–100% on GPT-4o/GPT-4o-mini vs 40–56% on Llama-3.1-8B; PoEM records 0% in all cells. These are author-produced result files, not live reruns in this archaeology (`farma_repro/results_gpt4o.json`, `farma_repro/results_llama8b.json`).

## Three-layer shape

```
Project Layer  (01)   project map, two stores, lifecycle, entrypoints, tests, dependencies
Engineering    (02)   49 EK, 156 link declarations, six edge types
Knowledge      (03)   10 KO = 9 L3 patterns + 1 L2 observation (NEEDS_HUMAN_REVIEW); 0 L4; 0 L5
Flow Atlas     (04)   49 edges across 7 classes
Candidates     (05)   14 candidates (incl. 4 cross-project hypotheses)
Validation     (06)   self-validation now; independent blind reconstruction appended in phase 5
```

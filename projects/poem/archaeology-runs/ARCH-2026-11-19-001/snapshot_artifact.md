# Repository Snapshot — PoEM

- run_id: `ARCH-2026-11-19-001`
- repository: `https://github.com/bithabib/ai_security.git`
- local_path: `archaeology-jobs/ARCH-2026-11-19-001/repo`
- branch: `main`
- HEAD: `4021642395daad997786cfbbd04b64924e8a46bd`
- commit_time: `2026-08-28 23:30:36 +0900`
- commit_subject: `Checkpoint: baseline defenses, multi-model results, and injection paper`
- shallow_depth: 50
- license: MIT（`LICENSE`，Copyright (c) 2026 Md Habibur Rahman）

## 仓库规模

- tracked files: 253
- Python: 49 个文件，4,712 行
- `farma_repro/*.py`: 2,767 行（PoEM/FARMA 主 harness）
- `injection_lab/*.py`: 1,028 行（另一条 injection 研究线）
- Markdown: 33；JSON: 63；PDF: 36；TeX: 15；PNG: 16
- 无 `pyproject.toml`、`requirements.txt`、版本号或 git tag；仓库是多研究产物集合，不是单一 Python 包

## 顶层结构

| 路径 | 内容 | 与 PoEM 关系 |
|---|---|---|
| `farma_repro/` | FARMA/PoEM harness、攻击、防御、实验、结果表、出图脚本 | 主对象 |
| `poem-paper/` | IEEE Access 论文 LaTeX/PDF、图表、bib | 设计意图来源，不等同实现事实 |
| `injection_lab/` | 另一组 prompt-injection 实验脚本 | 非同一条研究线，本轮不作为 PoEM 证据 |
| `injection-paper/` | 第二篇论文 LaTeX/PDF | 同上 |
| `papers/` | 参考论文 PDF/文本 | 背景资料 |
| 顶层 `*.md` | 分析笔记：`problem-and-solution.md`、`policy-and-overhead-analysis.md`、`poem-adversarial-analysis.md`、`auto-attacker-analysis.md`、`langchain-port-analysis.md`、`memory-problem-timeline.md`、`paper-draft.md`、`references-verified.md`、`related-work-draft.md` | 作者自述，作为设计意图/结果声称 |

## 入口与运行方式

- 主实验：`farma_repro/experiment.py`
  - `python3 experiment.py`：mock backend，无依赖
  - `python3 experiment.py --backend ollama --model llama3.1:8b --trials 20 --temperature 0.7`：真实模型
  - `python3 experiment.py --backend openai --model gpt-4o-mini ...`：OpenAI 后端
- 假阳性实验：`farma_repro/fp_experiment.py`
- PoEM-aware 对抗：`farma_repro/attack_poem.py`
- 判断/覆盖测试：`farma_repro/policy_test.py`
- 开销：`farma_repro/overhead_test.py`
- LangChain 端口：`farma_repro/langchain_port.py`（需 Python 3.12、`langchain_core`、`langchain_openai`、OpenAI key）
- 自动攻击器：`farma_repro/auto_attacker.py`

## 核心模块与数据结构

| 模块 | 关键符号 | 作用 |
|---|---|---|
| `memory.py` | `ReasoningMemory` | 攻击者可写的“笔记本”；`entries` 列表；`write()` 可选 HMAC 签名；`retrieve()` 子串匹配，无命中回退全量 |
| `memory.py` | `ExecutionLedger` | 追加式 HMAC 链台账；`events` 列表；`record()` 追加；`contains_event()` 先 `verify_chain()` 再匹配 action+subject；`verify_chain()` 从头重算 |
| `agent.py` | `Agent.run_task()` | 检索记忆 → LLM 决策 → `authorize_decision` 门控 → 若 safe action 则 `_execute_safety_step()` |
| `agent.py` | `inject_variant()` | 以 PLUGIN 授权签名或未签名方式写入伪造笔记 |
| `defenses.py` | `NoDefense` / `Sentinel` / `PoEM` / `NaivePoEM` / `AlwaysExecute` / `AnomalyDetection` / `SignedMemory` | 七种对照/防御 |
| `attack.py` | `BASIC_VARIANTS` / `ADAPTIVE_VARIANTS` | 3 条显式伪造 + 7 条改写伪造 |
| `attack_poem.py` | 4 类 PoEM-aware 攻击 | 台账伪造、跨 subject 重放、重构判断、越界动作（D 仅文档） |
| `scenarios.py` | `EHRAGENT` / `REACT_QA` / `RAP` | 三场景：validate / verify / confirm |
| `llm.py` | `LLMClient` | mock / ollama / openai；`_coerce_action()` 将不可解析回复强制 safe action 并计入 `parse_stats` |
| `experiment.py` | `wilson()` / `DEFENSES` | 3 场景 × 2 攻击族 × 6 防御 = 36 格；ASR 与 Wilson CI；coercion 统计 |
| `fp_experiment.py` | `build_benign_agent()` | 真实台账事件 + 真实自述日志，测合法跳过率/假阳性 |
| `langchain_port.py` | `LangChainAgent` | 真实 `InMemoryVectorStore` + `OpenAIEmbeddings` + `ChatOpenAI` + PoEM 台账 |

## 状态与生命周期

`Agent.__init__` 创建 `ReasoningMemory` 与 `ExecutionLedger`；每个 trial 新建 Agent，因此台账是**进程内、按 trial 独立**的。`Agent.run_task` 单次调用完成“检索 → 决策 → 门控 → 执行（或跳过）”。安全步骤执行时 `step += 1` 并 `ledger.record(safety_step, subject, step)`。无持久化、无恢复、无并发控制、无跨会话共享。

## 测试体系

- 无 pytest/unittest 测试目录；无 CI 配置；验证由可执行实验脚本承担：
  - `experiment.py`（36 格矩阵 + CI + coercion）
  - `fp_experiment.py`（3 场景 × 6 防御 × 3 真实日志风格）
  - `attack_poem.py`（4 攻击 × 4 防御）
  - `policy_test.py`（5 判断伪造 + 覆盖演示）
  - `overhead_test.py`（5 个台账规模）
- 结论：测试是“研究脚本 + 结果 JSON”，不是自动化回归测试；`README.md` 明确 mock 结果不是科学结果。

## 配置与外部依赖

- 核心 harness 仅 Python 标准库 + `matplotlib`（出图可选）。
- `ollama` 后端：本地 `http://localhost:11434`，默认 `llama3.1:8b`。
- `openai` 后端：`OPENAI_API_KEY` 环境变量或 `open_ai_api_key` / `openai_api_key` 文件（`llm._load_api_key`）。
- LangChain 端口：`langchain_core`、`langchain_openai`、`OpenAIEmbeddings(text-embedding-3-small)`、`ChatOpenAI(gpt-4o-mini)`。
- 无锁文件/版本固定；无容器/部署配置。

## 权限与治理机制

| 机制 | 位置 | 事实 |
|---|---|---|
| 记忆写入者签名 | `memory.AUTHORIZED_WRITERS` | `AGENT` 与 `PLUGIN` 各持硬编码 HMAC key；`SignedMemory` 只验证 WHO，不验证内容真假 |
| 台账写入 | `ExecutionLedger.record()` | 无 capability/token 校验；任何能拿到对象引用的代码都可调用 |
| 台账完整性 | `ExecutionLedger.verify_chain()` | HMAC-SHA256(action|subject|step|prev)，`compare_digest` 比较；失败 fail-safe 拒绝跳过 |
| 默认密钥 | `ExecutionLedger.__init__(secret_key=b"trusted-runtime-key")` | 源码内明文默认值；调用方可覆盖 |
| 决策门控 | `PoEM.authorize_decision()` | 只对 `action == unsafe_action` 生效；只检查 action+subject 是否存在，不检查 step/task/nonce/时效 |
| 攻击模型 | `README.md` / `poem-paper/body.tex` §Threat Model | 攻击者可写记忆，不能改代码/不能观察执行；不变量：台账仅可信动作层可写 |

## 外部依赖与边界

- `farma_repro` 是 PoEM 主 harness；`injection_lab`/`injection-paper` 是独立研究线，本轮**不纳入** PoEM 知识结论。
- 论文数字与仓库 `results*.json` 均为作者自报/已存结果；本轮只独立复现了 mock 逻辑路径，未重跑真实模型。

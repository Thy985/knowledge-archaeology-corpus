# RAMPART — 项目考古总览（00 Overview）

> run_id: ARCH-2026-09-04-001 ｜ skill_version: 3.2 ｜ commit: 125595cb2dc53ed81ac4da0f30a7042d6951e897
> 目标仓库: https://github.com/microsoft/RAMPART.git（main，2026-09-04 浅克隆快照）

## 一句话定位
RAMPART（Risk Assessment & Measurement Platform for Agentic Red Teaming）是微软 AI Red Team 出品的 **pytest 原生 Agent 安全测试框架**：把红队技术变成可嵌入 CI/CD 的回归测试，让 AI 安全成为"持续工程纪律"而非"周期性检查点"。

## 核心命题（本项目让我们认识到了什么）
**"安全"与"不确定"必须严格区分——一个 agent 安全测试框架，最重要的不是断言有多强，而是它对"看不到的部分"有多诚实。**
RAMPART 用三值结果（SAFE / UNSAFE / **UNDETERMINED**）+ 可观察性声明（ObservabilityLevel）+ 未确定操作数追踪（undetermined_operands），把"无证据"与"证据为无"结构性区分开。这是一个安全工具对自身可信度边界的工程化回答。

## 关键数据（可追溯）
| 项 | 值 | 证据 |
|---|---|---|
| 语言/运行 | Python ≥3.11，pytest 插件 | pyproject.toml |
| 规模 | 209 文件，rampart+tests ≈ 25,793 行 | git ls-files / wc |
| 依赖 | jinja2 / pydantic / **pyrit==0.13.0(git rev 6dc8b94)** / pytest / pytest-asyncio / pyyaml | pyproject.toml |
| 许可 | MIT，Alpha 状态 | pyproject.toml / LICENSE |
| 架构分层 | PyRIT(L1) → RAMPART(L2) → Consumer(L3) → Agent(L4) | docs/concepts/overview.md |
| 核心抽象 | BaseExecution(ABC) + 4 协议(Adapter/Session/Surface/InjectionHandle) + Evaluator/Driver/Sink 协议 | core/*.py |
| 执行策略 | XPIA(attack) + SingleTurn(probe) | attacks/_xpia.py, probes/_single_turn.py |
| 结果类型 | 唯一 Result（bool=result.safe） | core/result.py |
| 质量机制 | 6 层 CI（ruff/ty/flake8-rampart/4 版本矩阵/codeql/coverage） | .github/workflows/ |

## 与已有 Corpus 的连接
- **deepseek-harness**（ARCH-2026-09-03-001，已考古）：DeepSeek Harness 是"agent 运行时/控制面"，RAMPART 是"agent 安全测试面"——二者构成 **运行时 harness ↔ 安全 CI** 的对照；RAMPART 的 `agent-harness-control-plane` 候选卡同域。
- **KnowlegeMap**：candidates/[cand]rampart-clarity-2026.md（S 级），连接 dsh-pentest / silver-shield / E2E-CLI。

## 产物清单
```
00_overview.md            本文件
01_project-layer/         项目地图（架构/模块/入口/配置/依赖）
02_engineering-knowledge/ EK Graph（宽底座，24 EK + links）
03_knowledge-layer/       Generalized KO（8 KO + aggregation_rule）
04_flow-atlas/            七类流（Control/State/Data/Evidence/Authority/Memory/Policy）
05_candidates/            未验证假说 / 跨项目候选
06_validation/            Validation 报告（Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic）
run_metadata.yaml         run 元数据
```

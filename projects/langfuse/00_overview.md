# langfuse — Project Archaeology Overview

- **run_id**: ARCH-2026-10-01-001
- **project**: langfuse
- **repository**: https://github.com/langfuse/langfuse.git
- **commit**: a48751c99841ea1f88f214af234b458aba16c5a9（main）
- **skill_version**: knowledge-archaeology v3.2
- **timestamp**: 2026-10-01（UTC+8）
- **mode**: initial

## 一句话定位

**Langfuse 是开源 LLM 工程平台**（v4.48.0，MIT，TypeScript monorepo，35k★）：以 OpenTelemetry GenAI 语义约定为摄取基线的 trace / evaluate / improve 一体平台——外部 OTel span 经适配层归一化为内部 Observation 模型，事件化写入 ClickHouse；eval 系统以 DecisionModel（结构化问题定义）为规格、LLM-as-judge/代码 eval 为执行体、LIVE/MANUAL + blocked 规则为治理门；worker 侧以 25+ BullMQ 队列异步执行全部耗时路径。

## 为什么值得考古（Knowledge Value）

1. **Agent/AI Engineering 相关性（S 级）**：这是"LLM/Agent 可观测性平台"的标准实现样本——OTel GenAI semconv 消费（OtelIngestionProcessor/ObservationTypeMapper）、事件化摄取管道（17 种事件 → Redis → ClickHouse）、eval 系统（DecisionModel 三类问题 + blocking 治理 + LLM-as-judge + 双调度器代码 eval）、模型匹配与定价（modelMatch 缓存链）。corpus 27 项尚无观测平台类项目，本轮填补主题空白。
2. **Ecosystem Connection**：与已考古 agentevals（trace→评分，同 eval 域不同实现）、deepseek-harness（运行时遥测）、webwright（run-artifact-first）三方对照——"观测平台 vs 评测工具 vs agent 运行时"三形态。
3. **Novelty**：①DecisionModel 把 eval 定义成"结构化问题集"（CHOICE/SCORE/NOUL + state 变量 + 概率输出）而非自由文本 prompt——eval 规格化 ②EvalExecutionMode MANUAL 绕过 live toggle 但不绕过 blocked（治理分层）③确定性 SHA-256 trace 采样（无效配置保守保留）④跨日边界事件延迟防乱序 ⑤供应链治理（pnpm 5 天新依赖延迟 + allowBuilds 白名单 + 4 个上游未修丁）⑥AGENTS.md Agent 协同治理（subagent 委派/测试纪律/种子优先）。
4. **Benchmark 学习价值**：①"外部语义 → 内部模型"适配器注册表（ObservationTypeMapper priority 排序）——多源协议归一化的活教材 ②LIVE/MANUAL/blocked 三层 eval 治理门——"可暂停的自动化"设计 ③LLM-as-judge 的输出定义编译+校验（结构化解耦 judge 输出）④25+ 队列的 worker 装配（WorkerManager/DeadLetterRetry）——异步平台架构模板。

## 认知核心（一句话）

**"可观测性平台 = 语义适配 + 事件化存储 + 可治理的执行管道"**：Langfuse 用 OTel 适配层把异构 trace 归一为内部模型，用事件化管道保证摄取可靠（延迟/采样/去重），用 DecisionModel + 治理门让"评估"成为可暂停、可审计、可规格化的工程对象——观测、评测、治理三者在同一数据底座上闭合。

## 产物清单
- 01_project-layer/project-layer.md — 项目地图
- 02_engineering-knowledge/ek-graph.md — EK Graph（44 条，六类边）
- 03_knowledge-layer/knowledge-layer.md — Generalized KO 层（8 个，R1-R4）
- 04_flow-atlas/flow-atlas.md — 七类流
- 05_candidates/candidates.md — 未验证假说（6 条）
- 06_validation/validation.md — Blind Validation + 反例 + Reconciliation
- archaeology-runs/ARCH-2026-10-01-001/run_metadata.yaml — run 元数据

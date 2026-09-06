# 00 · Overview — Omnigent（ARCH-2026-09-07-001）

> 核心问题：**"Omnigent 让我们认识到了什么？"** —— 一个 meta-harness 如何在不重写任何 harness 的前提下获得统一编排、策略治理与实时协作。

## 一句话定位

Omnigent 是开源 **meta-harness**（Databricks 出品，Apache 2.0，alpha，v0.12.0@2026-09-01）：在 Claude Code / Codex / Cursor / OpenCode / Hermes / Pi / 自研 agent 之上提供统一编排层——组合（swap/combine harness 不重写）、控制（策略/沙箱/花销上限）、协作（任意设备实时共享会话）。

## 为什么值得考古（对个人知识系统的价值）

1. **控制面策略的工程化实现**：Policies 系统（声明式 PolicySpec → 纯求值器 Policy → engine 组合）与候选卡 [cand]agent-harness-control-plane 的"策略在 harness 层而非 prompt 层"判断同构——可直接对照五维模型/EP-002。
2. **异构 harness 适配的完整样本**：11 个 vendor harness 的 native bridge + 能力声明模型（5 种集成模式、4 种 elicitation 通道）——"适配层如何不腐烂"的活教材。
3. **失败与自愈的密集证据**：bundle 自愈、runner 重连、孤儿收割、诊断改进——大型 alpha 项目的可靠性工程实践。
4. **诚实的设计-实现差距审计**：OBSERVABILITY.md 主动披露 trace 不传播、dead code、未 wire 的 instrumentor——"设计意图 vs 实现现实"的罕见书面样本。

## 认知核心（详见 03）

- 执行点跟随控制权：策略必须附着在被执行的位置（runner 拥有 MCP dispatch → function policy 下移 runner）
- 审批是人际闸门：ASK 默认等待一天而非自动拒绝——超时语义是产品决策
- 资源可替换、身份持久：sandbox 可死，会话绑定存活
- 失败可解释性是工程投资：wedged LLM → connectivity、crash → issue、dark trace → tracing 计划

## 关键数字（全部可追溯到仓库）

| 项 | 值 | 来源 |
|----|-----|------|
| Python 规模 | 760 文件 / ~437k 行 | find + wc |
| 测试文件 | 1681 个 py | find tests |
| 最新 release | v0.12.0（2026-09-01） | CHANGELOG.md |
| HEAD | 381bf638fb31e6a51990d9dab54ea9ef4b933711 | git rev-parse |
| harness 适配 | 11 个（claude/codex/cursor/hermes/pi/kimi/kiro/qwen/goose/opencode/antigravity） | omnigent/ 顶层文件 |
| 集成模式 | 5 种（SDK_IN_PROCESS/CLI_SUBPROCESS/ACP_SUBPROCESS/NATIVE_TUI/NATIVE_SERVER） | harness_capabilities.py |
| 托管沙箱后端 | ≥10 种（Modal/Daytona/Blaxel/Islo/E2B/CoreWeave/K8s/OpenShell/Boxlite/microsandbox/Databricks） | README + managed_hosts.py |
| ASK 默认超时 | 86400s（旧 120s 会静默拒绝） | pending_approvals.py |

## 产物清单

| 文件 | 内容 |
|------|------|
| 01_project-layer.md | 项目地图（架构/模块/生命周期/依赖/入口） |
| 02_engineering-knowledge.md | EK Graph（30 EK + 六类边） |
| 03_knowledge-layer.md | Generalized Knowledge（7 KO + 4 CM + 4 M + 聚合规则） |
| 04_flow-atlas.md | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| 05_candidates.md | 未验证假说与跨项目候选 |
| 06_validation.md | Truth/Coverage/Flow/Abstraction/Counterexample/Epistemic + 质量指标 |
| run_metadata.yaml | run_id / commit / skill_version / 质量指标 / validation result |

## 免责声明

- 本次考古基于浅克隆 HEAD（381bf63），未追踪该 SHA 之后的更新。
- 测试环境缺重型测试依赖（playwright 视觉快照插件等），1681 测试未运行；"测试揭示行为"类证据为**测试意图阅读**（S3 级），非运行验证（S4 级）。已在 06 中如实标注。
- 项目为 alpha（v0.12.0），机制可能在后续版本变化；所有结论锚定快照 SHA。

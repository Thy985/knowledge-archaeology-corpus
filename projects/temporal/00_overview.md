# 00 · Overview — Temporal（ARCH-2026-11-02-001）

## 项目一句话

**Temporal = durable execution 的开山鼻祖与工业标准**：一个面向分布式应用/workflow 的持久化执行引擎。核心命题是"长时程任务跨进程崩溃/重启后从最后完成步骤恢复，不重放已完成副作用"——通过 **Event History 追加 + 确定性重放（Replay-based）** 实现，另以 **Chasm（ASM 状态机框架）** 开辟 Workflow 之外的第二个执行模型。

## 为什么值得考古（本轮选中理由）

1. **品类鼻祖**：durable execution 一词的实践源头（2019 年起），Temporal Server 1.32.1（2026-10-09 发布）是工业级参考实现；与已考古的 **Restate（journal 步骤持久化派）/ agent-substrate（挂起恢复派）** 构成 durable execution **三大执行模型对照**——Temporal 是其中唯一"运行前就已完全确定代码路径、用 Event History 重放恢复"的系统。
2. **Agent-AI 相关性**：2026-10-15 上线 llm-info agent 适配页；SDK 层（Java/TS/Python）是长时程 agent 任务（durable workflows、human-in-the-loop、signal-driven）的成熟载体，直接服务用户 TeamMind（Java 多 Agent 运行时）选型。
3. **Benchmark 学习价值**：确定性重放约束（Workflow.Sleep/Timer/外部 I/O 门控）、事件历史分支、RangeID/shard 所有权 CAS、VersionedTransition 幂等——这些是"如何设计不悬挂、可恢复、可审计的长时程执行"的核心机制。
4. **新引擎信号**：Chasm（10.2 万行）把 Workflow 降格为"一个 Application State Machine"，是 2026 年 Temporal 的重大架构转向，具有独特 Novelty。

## 考古范围与证据边界

- **证据面**：shallow clone（depth=1）HEAD `e0233ff`（2026-10-31），静态源码考古；总 1,124,761 行 Go（含 api/proto 生成代码 101,047，业务代码 1,023,714）。
- **聚焦**：核心执行模型（History/Workflow Task/Mutable State/Persistence CAS）+ Chasm 新引擎 + 动态配置/Worker Versioning/治理机制。
- **证据上限**：无 Docker 环境，未跑集成测试；测试代码（tests/ 1,072 个 *_test.go）仅作为"测试揭示的行为"（S4 参考），不升级为运行验证（S5）。
- **未覆盖**：SDK 仓库（temporalio/sdk-*，外部仓库）、完整 git 历史（浅克隆）、全量 1,023,714 行业务代码（超大项目，按核心机制裁剪采样）。

## 产物结构

```
package/
├── 00_overview.md          ← 本文件
├── 01_project_layer.md     ← 项目地图
├── 02_engineering_knowledge.md  ← EK Graph（宽底座，40 EK）
├── 03_knowledge_layer.md   ← Generalized KO（窄尖顶，9 KO，R1-R4 聚合）
├── 04_flow_atlas.md        ← 七类流
├── 05_candidates.md        ← 未验证假设
├── 06_validation.md        ← Validation + Reconciliation + 质量指标
└── run_metadata.yaml
```

## 核心结论（TL;DR）

- Temporal 用**"命令→事件"的两阶段闭环**实现确定性持久化：worker 的每个副作用意图都作为 Command 提交，History 服务校验后生成 Event 追加到 Event History 并更新 Mutable State，崩溃恢复时新 worker **重放 Event History**（不是重放 worker 本地逻辑）——重放是"读历史"，副作用"只执行一次"。
- **确定性重放不是没有代价**：它要求 workflow 代码完全确定性（时间/随机/异步全部注入抽象），因此 workflow 中不能做任意副作用；**Activity 是唯一的副作用出口**（带 retry/heartbeat/Cancel 状态机）。
- **Chasm 是"Workflow 太重"的答案**：把持久化状态机抽象成 Field[T]/Map/ParentPtr + Task，复用 sharding/路由/原子存储但不承担 workflow 的完整语义负担——Workflow 本身也实现为一个 ASM，证明"持久化执行引擎"可以在 workflow 之上再做一层正交抽象。
- **横向启示（对 TeamMind）**：Java 栈 durable execution 的首选仍为 Restate（已考古），但 Temporal 的**版本化 worker 部署（build ID / deployment / versioning search attributes）** 与 **signal/query 机制** 是多 Agent 运行时"热升级/查询正在运行 agent"的参考样板。

## 质量指标速览（详见 06_validation.md）

- EK：40 条（全部带 links，平均出边 2.1，游离 0）
- KO：9 个（R1×3 / R2×2 / R3×2 / R4×2，全部可回溯）
- Flow Atlas：7 类流 42 条 Edge（关键 Edge 全部回溯 symbol/file）
- IVR：盲重建判定（详见 06）
- 反例：9 KO 每个 ≥1 定向反例攻击（详见 06）

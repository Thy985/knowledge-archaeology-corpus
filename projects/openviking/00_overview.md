# 00 Overview · OpenViking

> **一句话**：OpenViking 把 agent 的上下文（知识 / 记忆 / 技能）做成一个 **`viking://` 虚拟文件系统**，用"目录 + 摘要 + 向量索引"同时满足**可读、可编辑、可检索、可审计**——它是 corpus 中第一个把"文件系统语义"当作记忆数据库主接口的工业实现。

- **run_id**: `ARCH-2026-11-11-001`
- **repository**: `volcengine/OpenViking` @ `f7e83b2d739905b50215cd7739199a564b9b044b`（main，2026-11-10 22:17 +0800）
- **license**: AGPL-3.0（只读分析）
- **mode**: initial
- **scope**: Python 核心 + `docs/en/concepts/`；Rust `crates/ragfs` 仅接口边界（见 `01_project-layer/architecture.md` §Scope）

## 1. 为什么考古这个项目

Memory 周前四天已在 corpus 建立四个正交点：

| 项目 | 记忆范式 | 核心命题 |
|---|---|---|
| mem0 | 抽取式写入 | 从对话抽事实、存入向量库 |
| graphiti | 时序知识图谱 | 实体/关系 + 时间有效性 |
| letta-code | 自编辑记忆 | agent 自己改写记忆块 |
| hy-memory 2.0 | 写入策略模型化 | 把"何时写/写什么"变成可训练策略 |
| **openviking** | **上下文数据库（文件系统即接口）** | **记忆/知识/技能同构为文件树，摘要分层，检索按目录 scope** |

它的独特性不是"又一个向量记忆"，而是**把存储抽象从"扁平索引"换成"可导航的文件系统"**，并围绕这个抽象重建了写入、检索、生命周期、权限与一致性。

## 2. 五条最值得带走的认识（详见 03）

1. **FS 是事实源，向量索引是派生品**：删除先删索引再删文件、写后重读再向量化、崩溃后靠队列重建派生层——所有不一致都朝"宁可漏检，不可返回坏结果"收敛。
2. **删除是写入协议的最后一步，且有 6 道前置守卫**：写失败、链接迁移失败、替换源失效、同批 unresolved upsert 都会让删除跳过；"Replace 同名"被显式降级为 Update。
3. **渐进披露不是 UI 技巧，是 token 经济学**：`abstract` 来自向量 payload（零读），`overview` 读 sidecar，`full` 才读原文；默认 tier 是 abstract，只有 `events` 允许升到 full。
4. **记忆是可声明 schema 的一等对象**：memory_type 由 YAML 定义目录、文件名、embedding 模板、字段与 `merge_op`；"经验"被强制写成 Situation / Approach / Reflect 三段式机器指令。
5. **权限与一致性都是"显式降级"而非"假装强保证"**：ACL 随索引更新、承认最终一致；trusted 模式必须 root key，但 root key 即跨租户 impersonation 能力。

## 3. 产物索引

| 文件 | 内容 |
|---|---|
| `01_project-layer/architecture.md` | 系统分层、模块、入口、部署拓扑、Scope |
| `01_project-layer/data_model_and_lifecycle.md` | URI / ContextType / L0-L2 / 会话与队列生命周期 |
| `01_project-layer/config_testing_governance.md` | 配置、测试、CI、许可与治理 |
| `02_engineering-knowledge/evidence-index.md` | 36 条事实的证据登记（S0–S5） |
| `02_engineering-knowledge/ek.md` | 36 条 Engineering Knowledge（含 links） |
| `02_engineering-knowledge/ek-graph.md` | 边表、出入度、簇与质量指标 |
| `03_knowledge-layer/ko.md` | 8 个 Generalized KO（R1–R4 + 反例） |
| `04_flow-atlas/flow-atlas.md` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05-candidates/candidates.md` | 未验证假设 / 跨项目候选 / 待人工审查 |
| `06-validation/self-validation.md` | 自校验（Blind Reconstruction 前的内部对照） |
| `06-validation/independent-validation.md` | 独立 Auditor 盲重建结果（阶段⑤） |
| `06-validation/reconciliation.md` | 合并修正（阶段⑥） |
| `run_metadata.yaml` | run 元数据与质量指标 |

## 4. 证据与认知状态声明

- 本文所有"代码行为"均落到 `file:line`；文档声明单独标注 `doc_claim`，不自动升为实现事实。
- 厂商自报 benchmark（LoCoMo 80–83%、token −34.3~91.0%、tau2 +6.87/+11.87pp）**只作为 Observation 记录**，不进入 Fact 层，不参与 KO 归纳。
- 未读 Rust `ragfs` 内部实现；凡依赖 Rust 的结论标 `boundary: Rust-unread`，在 Validation 中按 `PARTIALLY_CONFIRMED` 处理。

## 5. 一句话结论（先给，后文论证）

> OpenViking 的真正贡献不是"把记忆存成文件"，而是**把数据库的一致性责任重新分配给文件系统与派生索引**：内容写入是同步的、可解释的；索引是异步的、可重建的；删除与移动则用"顺序 + 守卫 + 锁"把最坏情况限制在"索引多一条脏记录"，而不是"源文件丢失"。

# 00 Overview — MemMachine Archaeology

- **run_id**: `ARCH-2026-11-14-001`
- **project**: MemMachine / MemMachine
- **slug**: `memmachine`
- **repository**: `https://github.com/MemMachine/MemMachine.git`
- **commit**: `377510d436dc81c3651502f2d5e36ff245650ce7`（2026-11-02T11:35:15-08:00）
- **license**: Apache-2.0
- **skill_version**: knowledge-archaeology v3.2
- **mode**: initial
- **priority**: A−
- **evidence ceiling**: 静态代码 + 测试代码 + 文档；**未运行测试/服务**，不主张 S4/S5

## 核心问题

> **这个项目让我们认识到了什么？**

MemMachine 不是一个“向量库 + 摘要”的简单包装，而是一个**把 agent memory 拆成多种物理表示并分别治理**的系统：原始 episode、派生片段/事件、语义 feature、短期摘要，各自有独立的存储、生命周期和失败模式。它最大的工程价值不在于“支持多少后端”，而在于它被迫回答了一个更根本的问题：

> **当一份记忆同时存在于 3~5 个存储/索引中时，系统如何保证“可检索的副本”和“不可丢的原文”不互相背叛？**

本轮考古得到的主要认知：

1. **“保留原文”不是一种存储选择，而是一条贯穿写/读/删的次序契约。**
   declarative backend 先写原始 `Episode`，再写 `Derivative` 与 `DERIVED_FROM` 边；event backend 先写 segment store，再写 vector index；删除时 event backend 反向先删索引、再删 segment。**原文与索引不是对等副本，而是“权威 + 可重建视图”**。但这条契约没有事务保护，所以它同时暴露了系统的真正脆弱点（EK-16~EK-23、KO-01、KO-02）。

2. **MemMachine 的“记忆质量”主要由写入路径决定，而不是检索模型。**
   同一个检索接口下，declarative 路径做“派生片段召回 + 原文回填 + 邻域扩展 + reranker”，event 路径做“segment 召回 + segment 窗口 + 可选 reranker”；semantic 路径则由 LLM 做 ADD/DELETE 与破坏性 consolidation。**不同后端的差异不在 embedding，而在“写什么、以什么粒度写、何时删、删了能否追溯”**（EK-20~EK-31、KO-02、KO-06、KO-07）。

3. **系统的默认姿态是“宁可关闭，也不半开”。**
   `episodic_memory.enabled` 默认 false；LTM 缺少 embedder/reranker/vector_graph_store 或 vector_store/segment_store 时整块关闭；semantic 缺少 LLM/embedding 凭证时自动关闭。**这是一种诚实的可用性策略：不假装具备记忆能力**。代价是“部署成功”与“记忆生效”之间存在一条容易被忽略的配置鸿沟（EK-07~EK-09、KO-03）。

4. **多租户身份是“命名空间约定”，不是认证边界。**
   `org_id/project_id` 由请求体提供；MCP 从 `org-id/proj-id/user-id` header 取值；REST 路由没有认证依赖。**租户隔离依赖调用方自律**。这不是“漏洞利用”结论，而是架构事实：MemMachine 把身份交给上层网关，自己只做命名空间分区（EK-05、EK-10、EK-11、KO-05）。

5. **最危险的失败不是“查不到”，而是“悄悄少了一部分”。**
   1~3 路条件并行写用 `return_exceptions=True` 后统一抛错，但没有补偿日志（源码留 TODO #1738）；semantic consolidation 先删后加，失败会丢记忆；STM 摘要失败只记录异常并保留旧摘要；event 索引与 episode store 漂移只 warning 并丢弃。**这些失败都不会让服务崩溃，只会让记忆变少/变旧**（EK-17~EK-19、EK-30、EK-48、KO-02、KO-07）。

## 产物清单

| 文件 | 内容 |
|---|---|
| `README.md` | 项目考古索引与三条核心发现 |
| `metadata.yaml` | 项目元数据、层数与验证判定 |
| `archaeology-runs/ARCH-2026-11-14-001/job-manifest.md` | 候选池、排序、选中理由、边界 |
| `archaeology-runs/ARCH-2026-11-14-001/snapshot-artifact.md` | commit、规模、基础项目地图、取证入口 |
| `01-project-layer/architecture.md` | 进程/模块/入口/部署拓扑 |
| `01-project-layer/data-model-and-lifecycle.md` | 核心数据结构与写/读/删/驱逐生命周期 |
| `01-project-layer/config-testing-and-governance.md` | 配置门控、测试体系、权限与治理 |
| `02-engineering-knowledge/evidence-index.md` | EV-001~EV-092 |
| `02-engineering-knowledge/ek.md` | EK-01~EK-52（L1/L2，宽底座） |
| `02-engineering-knowledge/ek-graph.md` | 六类边、聚合规则、质量指标 |
| `03-knowledge-layer/ko.md` | KO-01~KO-08（L3 candidate，窄尖顶） |
| `04-flow-atlas/flow-atlas.md` | 七类流 72 Edge（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `05-candidates/candidates.md` | C-01~C-12（含被推翻的 C-06） |
| `06-validation/self-validation.md` | 六类 Validator 自检 |
| `06-validation/independent-validation.md` | 独立 Auditor 盲重建判定 |
| `06-validation/reconciliation.md` | 审计后的合并/降级决定 |
| `06-validation/coverage-gaps.md` | G-01~G-10 覆盖缺口 |
| `archaeology-runs/ARCH-2026-11-14-001/run-metadata.yaml` | 运行元数据与质量指标 |

## 质量概览（最终数值见 run_metadata.yaml）

- **Evidence**：92 条（S3 90、S2 2；无 S4/S5）
- **Engineering Knowledge**：52 条；全部声明 `links`；平均出边 2.79、游离 EK 0
- **Generalized KO**：8 个 L3 candidate；全部声明 `aggregation_rule`；每个至少 3 条反例
- **Flow Atlas**：7 类流；关键 Edge 带 symbol/file/line/condition
- **独立验证**：CONFIRMED 13 / PARTIALLY 3 / DOWNGRADED 1 / OVER_GENERALIZED 2 / CONTRADICTED 1 / MISSING 5
- **关键未决**：API `score_threshold=-inf` 与 LTM “避免 -inf”的冲突；README 拓扑描述与默认部署不一致；多租户无认证的边界需部署方确认

## 阅读顺序

1. `01_project_layer/architecture.md` — 先建立“它由哪些进程/存储组成”
2. `01_project_layer/data_model_and_lifecycle.md` — 理解“一份记忆有几个副本、按什么顺序写/删”
3. `02_engineering_knowledge/ek.md` — 宽底座（52 条）
4. `03_knowledge_layer/ko.md` — 窄尖顶（8 条，先看 scope 与反例）
5. `04_flow_atlas/flow-atlas.md` — 用流校验结论
6. `06_validation/`（self / independent / reconciliation） — 看哪些被降级/反驳

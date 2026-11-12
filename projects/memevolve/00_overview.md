# 00 Overview — MemEvolve / EvolveLab Archaeology

- **run_id**：`ARCH-2026-11-13-001`
- **repository**：`https://github.com/bingreeky/MemEvolve.git`
- **commit**：`6035d5659d7a092dbfa6a87b1a32a3cee652ba54`（2026-05-05）
- **license**：Apache-2.0
- **skill_version**：knowledge-archaeology v3.2
- **mode**：initial
- **priority**：A−

## 核心问题
> **这个项目让我们认识到了什么？**

MemEvolve 把“记忆系统设计”从人工调参变成**代码级搜索**：用 LLM 读取失败轨迹 → 生成新的 provider 实现 → 直接改写注册表与配置 → 隔离验证 → 在真实任务上与基座做锦标赛。它同时提供了一个少见的横向对照底座：**13 套记忆系统共享同一套 3 方法接口**。

本轮考古得到的最重要认知不是“它有 13 个 provider”，而是：

1. **记忆系统的差异不在存储层，而在“写入门控 + 阶段策略 + 变异/剪枝规则”**——13 个 provider 都用 JSON，但 7 个只在 BEGIN 提供记忆，成功门控有 5 种写法，写入有 append/merge/revise/consolidate/prune 五种形态。
2. **元进化系统的真正瓶颈是“变异事务性”与“验证独立性”**：Creator 非原子写代码（E-063/E-064），Validator 同时是修复者并把修复回写真实目录（E-067/E-068）。这使搜索过程可能污染自己的基因型，也让质量门失去独立性。
3. **接口的窄化会反向塑造实现**：只有 3 个方法、2 个阶段、2 种载荷，导致删除/合并/失效只能藏在 `take_in_memory` 内部，无法被上层审计（E-001/E-002）。

## 产物清单

| 文件 | 内容 |
|---|---|
| `job_manifest.md` | 候选排序、选中理由、边界 |
| `snapshot_artifact.md` | commit、规模、基础项目地图 |
| `package/01_project_layer/` | 架构、数据模型与生命周期、配置/测试/治理 |
| `package/02_engineering_knowledge/` | evidence-index（100 条）、EK 50 条、EK Graph |
| `package/03_knowledge_layer/` | KO-01~KO-09 + 16 个 Pattern candidates |
| `package/04_flow_atlas/` | 七类流（Control/State/Data/Evidence/Authority/Memory/Policy） |
| `package/05_candidates/` | 9 条未验证假说（含跨项目候选） |
| `package/06_validation/` | 自验证、Reconciliation |
| `validation/independent_validation.md` | 独立 Auditor 盲重建结果 |
| `package/run_metadata.yaml` | 运行元数据与质量指标 |

## 质量概览（详见 run_metadata.yaml 与 06_validation）
- Evidence：100 条（S3×96，S0-vendor×4；无 S4/S5）
- Engineering Knowledge：50 条；平均出边 ≥1；游离 EK 0（每条至少 1 条 link）
- Generalized KO：9 个（L3×9，L4×0；原申报 L4×2 经独立验证降级）；全部声明 aggregation_rule；L3+ 均附 ≥3 反例
- Flow Atlas：7 类流；关键 Edge 均带 symbol/file/condition
- Independent Validation：见独立报告（CONFIRMED / PARTIALLY_CONFIRMED / DOWNGRADED / CONTRADICTED 统计）
- 证据上限：静态代码 S3；论文数字仅作 S0-vendor，未进入 Fact/KO

## 阅读顺序建议
1. `01_project_layer/architecture.md` — 先建立地图
2. `02_engineering_knowledge/ek.md` — 宽底座
3. `03_knowledge_layer/ko.md` — 窄尖顶（先看每个 KO 的 scope 与反例）
4. `04_flow_atlas/flow-atlas.md` — 用流校验前面的结论
5. `06_validation/` + `validation/independent_validation.md` — 看哪些结论被降级

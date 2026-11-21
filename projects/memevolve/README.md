# MemEvolve / EvolveLab — 知识考古索引

> 一句话定位：**把“记忆系统的实现差异”压缩为可比较的设计维度，并暴露“记忆架构自我进化”的工程风险。**

- 仓库：https://github.com/bingreeky/MemEvolve
- Commit：`6035d5659d7a092dbfa6a87b1a32a3cee652ba54`（2026-05-05，branch `main`）
- Skill：`knowledge-archaeology` v3.2 · Run：`ARCH-2026-11-13-001` · 模式：initial
- 考古日期：2026-11-13
- 证据强度：S3×96 + S0-vendor×4（无运行验证，S4/S5 缺失）

## 产物

| 层 | 文件 | 内容 |
|---|---|---|
| 00 | `00_overview.md` | 核心问题、三条认知、质量概览 |
| 01 | `01-project-layer/` | 架构、数据模型与生命周期、配置/测试/治理 |
| 02 | `02-engineering-knowledge/` | 100 条 Evidence、50 条 EK、EK Graph（134 边） |
| 03 | `03-knowledge-layer/ko.md` | 9 个 KO（L3×9）+ 16 个 Pattern 候选 |
| 04 | `04-flow-atlas/flow-atlas.md` | 七类流、68 条 Edge |
| 05 | `05-candidates/candidates.md` | 14 条 Hypotheses / 跨项目候选 |
| 06 | `06-validation/` | 自检、独立验证、Reconciliation |
| run | `archaeology-runs/ARCH-2026-11-13-001/` | Job Manifest、Snapshot、run metadata |

## 三条核心认知

1. **绑定“记忆”抽象的不是存储，而是写入形态、成功门控与阶段能力**——13 个 provider 的差异集中在三处，而不是嵌入模型。
2. **元进化的瓶颈不是生成质量，而是变异的事务性与验证的独立性**——Creator 非原子、Validator 默认自修并回写真实目录。
3. **窄接口（3 方法、2 状态）把生命周期语义压进每个实现**——导致失败摄取、剪枝、审计等语义在 13 个 provider 中各不相同。

## 验证状态

`PARTIALLY_VALIDATED`：静态机制链完整；独立审计 21 项中 8 CONFIRMED、4 PARTIALLY、3 DOWNGRADED、1 CONTRADICTED、1 NEEDS_HUMAN、2 MISSING；KO-06/07 已从 L4 降为 L3。无性能结论。

## 边界

- 只读考古：未修改目标仓库。
- 飞书落盘未授权，未执行。
- 性能数字（含最高 +17.06%）仅作作者自报（S0-vendor），未复现。

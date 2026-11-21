# EK Graph

## 图统计（脚本实测，非手工估计）

| 指标 | 值 | 质量门 |
|---|---:|---|
| EK 节点 | **33** | — |
| 有向边（links 声明） | **95** | — |
| 平均出度 | **2.88** | ≥1 ✅ |
| 出度范围 | 2 ~ 5 | — |
| 孤立 EK | **0**（0%） | <20% ✅ |
| 连通分量 | **1**（33/33 节点连通） | 无碎片 ✅ |
| 聚合规则覆盖率 | 100%（见 03 层矩阵） | 100% ✅ |

## 六类边分布

| 边类型 | 数量 | 语义 |
|---|---:|---|
| `causal` | 20 | 触发/导致 |
| `contrast` | 23 | 同一问题的两种解法 |
| `dependency` | 15 | 运行时/前置依赖 |
| `constraint` | 15 | 边界/前置条件 |
| `mechanism` | 12 | 共享实现机制 |
| `subsystem` | 10 | 同子系统 |

> `contrast` 占比最高（24%），说明该项目的知识主要靠"对照"组织：时间 vs 系统时间、LLM vs 确定性、开源 vs 托管、串行 vs 并发、多后端差异。这是本项目知识结构的一个特征，不是噪声。

## 主要连通骨架

```
写入管线：      EK-01 ──causal──► EK-13 ──causal──► EK-04
                    │ subsystem                    ▲ dependency
                    ├──► EK-02 ──constraint──► EK-12 │
                    └──► EK-07 ──causal──► EK-08 ──constraint──► EK-09
时间模型：      EK-03 ──causal──► EK-04 ──causal──► EK-11
                    └──contrast──► EK-10
作用域：        EK-05 ──constraint/mechanism──► EK-06 ──causal──► EK-33
                    └──causal──► EK-21
能力边界：      EK-22 ──causal──► EK-26 ──causal──► EK-27 ──causal──► EK-28
                                                      └──causal──► EK-29
驱动体系：      EK-17 ──mechanism──► EK-08
                    ├──dependency──► EK-18 ──causal──► EK-19
                    └──constraint──► EK-31
```

## 反退化检查

- 每条 EK 至少 2 条出边，无"模块说明"式孤立条目；
- 不存在"仅因为同属某子系统"的聚合（见 03 层：每个 KO 的聚合规则均为 R1 机制簇 / R2 因果链 / R3 不变量 / R4 主题，且写明跨实例证据）；
- `subsystem` 边仅用于表达结构归属，不单独构成 KO 理由。

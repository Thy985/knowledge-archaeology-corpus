# 02.3 EK Graph · 节点、边、质量

## 1. 节点（30）

| 组 | EK |
|---|---|
| 协议迁移 | EK-01~04 |
| 会话模型 | EK-05~10 |
| 门禁/通知 | EK-12~14 |
| 多轮交互 | EK-15~16 |
| 版本三平面 | EK-17~18 |
| 事件/重放 | EK-19~21 |
| 传输加固 | EK-22~25 |
| 授权 | EK-26~27 |
| 缺口显性化 | EK-28~30 |

> EK-11（MCPGODEBUG）横跨会话与治理，归入"会话模型"组。

## 2. 边（37）

| # | From | To | 类型 |
|---|---|---|---|
| 1 | EK-01 | EK-02 | mechanism |
| 2 | EK-01 | EK-04 | causal |
| 3 | EK-01 | EK-05 | contrast |
| 4 | EK-02 | EK-03 | dependency |
| 5 | EK-03 | EK-18 | subsystem |
| 6 | EK-04 | EK-05 | constraint |
| 7 | EK-05 | EK-06 | mechanism |
| 8 | EK-05 | EK-07 | contrast |
| 9 | EK-05 | EK-10 | mechanism |
| 10 | EK-06 | EK-12 | constraint |
| 11 | EK-07 | EK-08 | subsystem |
| 12 | EK-07 | EK-09 | contrast |
| 13 | EK-26 | EK-08 | dependency |
| 14 | EK-09 | EK-27 | constraint |
| 15 | EK-10 | EK-11 | mechanism |
| 16 | EK-11 | EK-30 | subsystem |
| 17 | EK-12 | EK-13 | contrast |
| 18 | EK-13 | EK-14 | causal |
| 19 | EK-14 | EK-15 | mechanism |
| 20 | EK-15 | EK-16 | mechanism |
| 21 | EK-16 | EK-18 | contrast |
| 22 | EK-17 | EK-18 | mechanism |
| 23 | EK-17 | EK-05 | dependency |
| 24 | EK-18 | EK-03 | constraint |
| 25 | EK-19 | EK-20 | subsystem |
| 26 | EK-19 | EK-21 | dependency |
| 27 | EK-20 | EK-21 | subsystem |
| 28 | EK-22 | EK-23 | subsystem |
| 29 | EK-22 | EK-24 | subsystem |
| 30 | EK-22 | EK-26 | constraint |
| 31 | EK-23 | EK-24 | subsystem |
| 32 | EK-24 | EK-25 | subsystem |
| 33 | EK-25 | EK-18 | constraint |
| 34 | EK-26 | EK-27 | mechanism |
| 35 | EK-27 | EK-28 | subsystem |
| 36 | EK-28 | EK-29 | mechanism |
| 37 | EK-29 | EK-30 | subsystem |

## 3. 质量统计

| 指标 | 值 | 门槛 |
|---|---|---|
| 节点数 | 30 | — |
| 边数 | 37 | — |
| 平均度数（无向计） | 2×37/30 = **2.47** | ≥1 |
| 孤立 EK | **0** | <20% |
| 有向边 | 13 | — |
| 无向边 | 24 | — |
| 覆盖 KO 簇 | 7 个簇 / 30 EK = 100% | 100% |

## 4. 边类型分布

| 类型 | 数量 | 说明 |
|---|---|---|
| mechanism | 12 | 共享实现机制（双栈、合成状态、MRTR、token 顺序） |
| subsystem | 11 | 同属一个子系统（重放、传输加固、治理） |
| causal | 3 | 探测→回退、通知→新协议禁令 |
| dependency | 4 | discover 依赖 transport 能力、auth 依赖会话、重放依赖存储 |
| constraint | 5 | 版本列表约束回退、兼容层约束门禁、会话绑定约束 token 校验 |
| contrast | 5 | 无状态 vs 有状态、门禁 vs 通知、header vs `_meta` |

## 5. 图的形状判断

- **两条主轴**：
  1. `EK-01 → 02 → 03 → 18`：协议迁移与版本三平面；
  2. `EK-19 → 20 → 21`：重放与存储契约。
- **一个横切簇**：`EK-05/06/07/09/10/11` 围绕"无状态 vs 有状态"形成对比星形。
- **一个安全簇**：`EK-22/23/24/25/26/27` 从传输加固连到授权。
- **无孤立节点**：说明没有为凑数写"模块说明"。

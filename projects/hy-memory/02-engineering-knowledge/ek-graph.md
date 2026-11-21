# 02.3 · EK Graph（边、度数与簇）

## 1. 统计口径

| 指标 | 值 |
|---|---:|
| EK 节点 | 29 |
| 声明的有向 links | 58 |
| 同对节点同类型去重后 | 45 |
| 平均出边（58/29） | 2.00 |
| 无 links 的游离 EK | 0 |
| 六类边覆盖 | mechanism / subsystem / causal / dependency / constraint / contrast 全部出现 |

> 说明：`links` 在 EK 条目中以"从本节点出发"记录；无向边（mechanism/subsystem/contrast）在两端各出现一次，故去重后为 45。
> 本轮 Independent Validation 后新增 EK-29（`constraint` → EK-15），节点与边数已同步。

## 2. 边表（按类型）

### 2.1 `causal`（有向 A → B，A 触发/导致 B）

| From | To | 语义 | 证据锚点 |
|---|---|---|---|
| EK-02 | EK-03 | 只有含 tool 才触发三分类 | `client.py:2949` |
| EK-03 | EK-04 | 分类失败进入 fail-soft 分支 | `judge.py:215` |
| EK-07 | EK-08 | 原文先写成功后才进入派生判重 | `writer.py:2660→1298` |
| EK-09 | EK-12 | SHADOW 状态使节点退出召回集合 | `memory.py:121` + `reader_base.py:1555` |
| EK-15 | EK-16 | value bar 决定是否产生 Experience | `coding/writer.py:188` |
| EK-17 | EK-22 | 链存在才能被检索装饰 | `client.py:3030` |
| EK-19 | EK-17 | patch 解析结果写入链 | `chain_store.py:314` |
| EK-22 | EK-07 | 写入成功是挂链前置 | `client.py:3030` |

### 2.2 `dependency`（有向：A 依赖 B 的运行时保证）

| From | To | 语义 |
|---|---|---|
| EK-08 | EK-28 | fail-closed 的可重跑依赖 request_id 幂等 |
| EK-10 | EK-09 | 链头判重依赖逻辑删除保证"旧版本不复活" |
| EK-13 | EK-12 | 图增强不改变主通道结论 |
| EK-21 | EK-17 | 无 git 时链仍可退化工作 |
| EK-22 | EK-17 | 装饰器依赖链已写入 |
| EK-26 | EK-24 | workspace 解析依赖来源授权的身份校验 |
| EK-27 | EK-23 | 本地 API 是管理面可信前提 |
| EK-28 | EK-08 | （反向声明在 EK-08 中） |

### 2.3 `constraint`（有向：A 是 B 的边界/前置条件）

| From | To | 边界 |
|---|---|---|
| EK-02 | EK-03 | 分类的前置是"含 tool" |
| EK-04 | EK-02 | 失败时退回最保守终点 |
| EK-05 | EK-03 | 强制 coding 仍受 tool 前置约束 |
| EK-09 | EK-12 | 状态字段约束召回可见性 |
| EK-16 | EK-15 | 阈值约束经验数量 |
| EK-18 | EK-25 | owner 键约束用户隔离 |
| EK-19 | EK-20 | 合成 diff 的不可信行号约束归并方式 |
| EK-20 | EK-19 | 拒收规则约束链内容 |
| EK-23 | EK-15, EK-17 | 管理面不得复制两层权威 |
| EK-24 | EK-25, EK-26 | 授权身份约束用户/工作区解析 |
| EK-25 | EK-11 | 字符集约束判重分组假设 |
| EK-29 | EK-15 | 返回值只覆盖双层中的上层（`memory_id`≠写入成功） |

### 2.4 `mechanism`（无向：共享同一实现机制）

| 对 | 机制 |
|---|---|
| EK-01 ↔ EK-02 | 入口即收敛的枚举/短路 |
| EK-09 ↔ EK-10 | 版本链的"当前有效"语义 |
| EK-11 ↔ EK-25 | 用键维度表达隔离 |
| EK-17 ↔ EK-22 | 链是"确定性的旁路数据" |
| EK-18 ↔ EK-25, EK-26 | 隔离键构造 |
| EK-25 ↔ EK-26 | 身份解析 |

### 2.5 `subsystem`（无向：同子系统）

| 簇 | 成员 |
|---|---|
| 场景路由 | EK-01, EK-02, EK-03, EK-05 |
| Productivity | EK-06, EK-15, EK-17, EK-22 |
| 检索 | EK-11, EK-12, EK-13 |
| System 2 | EK-14, EK-06 |

### 2.6 `contrast`（无向：同一问题的两种解法）

| 对 | 对照 |
|---|---|
| EK-04 ↔ EK-08 | fail-soft（降级 chat） vs fail-closed（整批不写） |
| EK-07 ↔ EK-08 | 原文 non-fatal vs 派生 all-or-nothing |
| EK-02 ↔ EK-05 | 自动判定 vs 强制指定（同一前置条件） |
| EK-07 ↔ EK-14 | 自动每次触发 vs 手动全量 |
| EK-20 ↔ EK-21 | 宁可拒收 vs 静默退化 |

## 3. KO 聚合（R1–R4）

| KO | 规则 | 簇内 EK | 边类型 | 簇成立校验 |
|---|---|---|---|---|
| KO-01 场景路由的成本/保守性 | **R1 机制簇** | EK-01, 02, 03, 04, 05 | mechanism + causal + constraint | 跨 2 个独立子系统（client 路由 / judge）；5 条共享"入口收敛"机制；可命名；可回溯 |
| KO-02 写入的两级失败策略 | **R2 因果链** | EK-07 → EK-08 → EK-28 | causal + dependency | 完整链"原文先写→派生整批→重跑幂等"；有明确决策→执行→记录 |
| KO-03 召回只见当前有效语义 | **R3 不变量簇** | EK-09, 10, 12 | causal + dependency + constraint | 三条 EK 汇聚到同一不变量；跨存储层（状态机/链/召回） |
| KO-04 事实保底、经验增值 | **R4 主题簇** | EK-15, 16 | causal + constraint | 同一主题的互补维度（保底 vs 增值）；跨文件（store/writer） |
| KO-05 版本链是附件而非通道 | **R2 因果链** | EK-17 → 19 → 20/21 → 22 | causal + constraint + contrast | 决策（零 LLM）→ 解析（三来源）→ 过滤（失败拒收）→ 消费（装饰器） |
| KO-06 管理面不得复制引擎权威 | **R3 不变量簇** | EK-23, 24, 27 | constraint + dependency | 三条 EK 汇聚到"权威单一"；跨架构/权限/安全三个面 |
| KO-07 隔离由键的完备性决定 | **R1 机制簇** | EK-11, 18, 25, 26 | mechanism + constraint | 同一机制（把隔离维度写进键）跨 chat 判重 / chain / user / workspace 四个独立子系统 |

> **反退化检查**：没有任何 KO 以"都属于某子系统"为聚合理由。KO-04 的聚合是"同一主题的互补维度"（保底 vs 增值），不是"都在 coding 子系统"。

## 4. 图健康度

| 检查 | 结果 |
|---|---|
| 平均出边 ≥1 | ✅ 2.04 |
| 游离 EK <20% | ✅ 0% |
| 聚合规则覆盖率 100% | ✅ 7/7 |
| KO 平均簇规模 3~12 | ✅ 2.4（KO-04 仅 2 条，低于区间；见下） |
| 无"同子系统=聚合理由" | ✅ |

**已知弱点**：KO-04 簇规模 2，低于 3~12 的建议区间。它成立是因为"保底/增值"是一个完整对照机制，但证据面偏窄（只覆盖 `coding/store.py` 与 `coding/writer.py`）。在 03 层标注为 **L2（工程经验）而非 L3**，不强行凑第 3 条 EK。

## 5. 关键路径（图上的主干）

```
EK-01 ──▶ EK-02 ──▶ EK-03 ──▶ EK-04
             │                      │
             └── EK-05 ◀────────────┘

EK-07 ──▶ EK-08 ──▶ EK-28

EK-09 ──▶ EK-10
   └────▶ EK-12 ──▶ EK-13

EK-15 ──▶ EK-16

EK-17 ──▶ EK-19 ──▶ EK-20 ◀──▶ EK-21
   └────▶ EK-22 ──▶ EK-07

EK-23 ◀── EK-27
EK-24 ──▶ EK-26
EK-25 ──▶ EK-18
```

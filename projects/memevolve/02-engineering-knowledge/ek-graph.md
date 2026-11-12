# EK Graph — MemEvolve / EvolveLab

## 1. 图统计（由 `ek.md` 机器复算）

| 指标 | 值 | 口径 |
|---|---|---|
| 节点（EK） | **50** | `### EK-xx` 块 |
| 边（links） | **134** | 六类边正则抽取 |
| 平均出度 | **2.68** | 134/50 |
| 游离节点 | **0**（0%） | 无 links 的 EK |
| 自环 / 非法目标 | **0 / 0** | 校验目标存在 |
| 连通分量 | **1 个（50 节点全连通）** | 无向化后并查遍历 |
| 边类型分布 | contrast 40 / causal 39 / mechanism 20 / constraint 16 / dependency 14 / subsystem 5 | — |

> 达标：平均出边 ≥1、游离 EK <20%、目标全部可解析。

## 2. 枢纽节点（入度）

| EK | 入度 | 为什么成为枢纽 |
|---|---|---|
| EK-17 Evolver 原则库/打分/合并 | 8 | 多数 provider 对照都指向它（append vs merge） |
| EK-07 阶段支持 7/1/5 | 7 | 一切“为什么 IN 没记忆”的解释入口 |
| EK-14 Agent-KB append + 全量重建 | 7 | append 模式的基准样本 |
| EK-33 五种写入形态 | 6 | 横向对照的汇聚点 |
| EK-39 Creator 写入顺序 | 6 | 元进化缺陷链的上游 |
| EK-09 384 维嵌入底座 | 5 | 检索/存储机制的共同依赖 |
| EK-43 验证器默认自动修复 | 5 | 治理/验证缺陷链的汇聚点 |

## 3. 关键因果链（R2 簇的基础）

### Chain A：元进化 → 代码变异 → 验证/修复 → 真实环境
```
EK-35 四阶段状态机
  → EK-37 生成（全文注入 + 高温度/大 token）
  → EK-38 字符串解析
  → EK-39 Creator 顺序写入
  → EK-40 非原子无回滚
  → EK-41 config 失败仅告警
  → EK-43 验证器默认自动修复
  → EK-45 修复结果 copy2 回真实目录
```
解释力：一条“搜索会污染自己基因型”的完整机制故事；对应 KO-06。

### Chain B：接口窄化 → 写入形态内聚 → 门控漂移
```
EK-01 三方法接口
  → EK-33 五种写入形态
  → EK-34 成功门控 5 种写法
  → EK-21 MemP 缺失默认成功
  → EK-32 Lightweight 索引位移
```
解释力：为什么“同一个 TrajectoryData 在不同 provider 中得到相反结果”；对应 KO-03/KO-04。

### Chain C：两阶段协议 → 注入点 → 阶段能力差异
```
EK-02 只有 BEGIN/IN
  → EK-08 user 消息注入（planning + 每步）
  → EK-07 7/1/5 阶段分布
  → EK-27 Cerebra IN 独立阈值
```
解释力：为什么 7/13 系统在任务中途“失忆”；对应 KO-05。

### Chain D：公平性重置 → 锦标赛 → 选择偏差
```
EK-46 storage 重置
  → EK-47 x/m/t/y 流程
  → EK-48 字典序/Pareto
  → EK-50 无测试、简单均值
```
解释力：选择结果的可重复性风险；对应 KO-08。

## 4. 机制簇（R1 候选）

| 簇 | 成员 | 共享机制 | 跨子系统性 |
|---|---|---|---|
| M1 “成功后抽象再入库” | EK-14, EK-15, EK-16, EK-22 | LLM 摘要 + JSON + 成功门控 | Agent-KB / Agent-Workflow / Dilu/Gen/Voyager / Expel 四个独立文件 |
| M2 “打分/容量驱动的剪枝” | EK-17, EK-31, EK-28 | 打分或容量触发删除/合并 | Evolver / Lightweight / Cerebra |
| M3 “API 型记忆” | EK-25, EK-26 | 记忆=可调用工具 | SkillWeaver / Cerebra |
| M4 “隔离环境验证” | EK-44, EK-42, EK-45 | 复制目录 + import + 行为快检 | Validator 内部三处 |

## 5. 对照边（contrast）的三个主轴
1. **写入**：append（EK-14）↔ merge（EK-17）↔ revise（EK-20）↔ consolidate（EK-28）↔ rewrite（EK-23）。
2. **阶段**：BEGIN-only（EK-07）↔ 双阶段（EK-27）↔ IN-only（EK-23）。
3. **表示**：embedding（EK-09）↔ TF-IDF（EK-22）↔ 词法 token（EK-24）↔ 图（EK-26）↔ 可执行代码（EK-25）。

## 6. 图的使用方式
- KO 只能从上述簇/链生成（见 `03_knowledge_layer/ko.md` 的 aggregation_rule）。
- 任何 KO 若无法回指到本图节点，视为悬空升维，进入 `05_candidates/`。

# 03 · Knowledge Layer — CLEAR（Generalized Knowledge）

> KO 必须声明 aggregation_rule（R1-R4）并回溯 EK/Evidence；Cross-project 一律标注 pending。

## KO-01 · 评估是证据生产管线（R2 因果链簇）
- **aggregation_rule**：R2 因果链簇——EK-02→EK-03→EK-04→EK-05→EK-06 沿 causal 边形成完整链（归一→压缩→评估→聚合→模式）。
- **L3 Pattern**：agent 评估系统不应是"单个 judge 打分"，而应是分阶段证据管线：**trace → 中间表示 → 压缩 → 多粒度评估 → 聚合 → 统计模式**；每阶段消费前一阶段产物。
- **证据**：EK-02/03/04/05/06 全部 S3 + S4（211 测试）。
- **解释范围扩大**：与个人 Validation 编译器（Claim→Operationalization→Evidence→Judgment→Done）同构——评估是编译器不是打分器。Cross-project validation pending（CLEAR + Validation 编译器 2 例）。

## KO-02 · 中间表示解耦：异构输入先归一，下游只面对稳定契约（R4 主题簇）
- **aggregation_rule**：R4 主题簇——EK-02（IR 定义）+ EK-16（消息块归一）+ EK-26（provider 无关）+ EK-03（压缩依赖 IR）。
- **L3 Pattern**：评测系统面对异构框架/观测平台时，先建立统一中间表示（IR），所有下游（评估/聚合/可视化）只消费 IR；新增来源 = 新增预处理器，评估层零改动。
- **证据**：EK-02/16/26（S3/S4）。
- **解释范围扩大**：与数据工程 ETL（staging layer）同族；CLEAR 把 IR 作为唯一稳定契约（docs 明示不支持的平台自产 IR 即可）。Cross-project validation pending。

## KO-03 · judge 输入纯度是评估可信度的第一前提（R3 不变量簇）
- **aggregation_rule**：R3 不变量簇——EK-11（系统消息纯度约束）作为不变量，约束 EK-04 全部评估器；EK-24（分数与文本绑定）支撑可核查性。
- **L3 Pattern**：LLM-as-judge 的系统提示必须显式限定"只依据评估对象本身内容、禁止外部元数据影响判断"，且分数必须与理由（critique）绑定——评估输出可核查。
- **证据**：EK-11（S3）、EK-24（S3）。
- **解释范围扩大**：声明 ≠ 证据（与 Dogwood/Codex 验证纪律同族）——judge 的输入纯度 = 评估的证据纪律。Cross-project validation pending。

## KO-04 · 失败模式挖掘需要统计控制（R3 不变量簇）
- **aggregation_rule**：R3 不变量簇——EK-08（统计纪律）作为不变量约束 EK-06（模式挖掘）；effect/lift/BH 校正共同构成"模式显著"的判定。
- **L3 Pattern**：从评估数据中挖"预测成功/失败的路径模式"时，必须做显著性检验（Fisher exact）+ 多重检验校正（BH）+ 效应量化（effect/lift），否则是模式幻觉；过滤过稀有/过常见模式。
- **证据**：EK-06/08（S3 + S4 test_static_dashboard_recs）。
- **解释范围扩大**：把科学方法（假设检验）内嵌到工程 dashboard——评估洞察的可信度由统计纪律保证。Cross-project validation pending。

## KO-05 · 评测工程化是可复用基建：确定性、可恢复、可替换（R1 机制簇）
- **aggregation_rule**：R1 机制簇——EK-07（温度探测）、EK-10（断点续跑/缓存/并发）、EK-09（外部 judge 插件）、EK-25（结果落盘）共享"评测运行工程化"机制，跨 ≥2 独立子系统。
- **L3 Pattern**：生产级评测系统需要与评估方法论同等级的工程化：确定性保证（温度/随机性控制 + 探测回退）、可恢复性（checkpoint/缓存）、可替换性（judge 插件化）、并发控制——"评测运行"本身是工程产品。
- **证据**：EK-07/09/10/25（S3 + S4）。
- **解释范围扩大**：评测基建模式（与 OpenCode 的 harness 基建、Dogwood 的状态化治理同族）。Cross-project validation pending。

## KO-06 · 多粒度评估互补：成本投向失败样本（R4 主题簇）
- **aggregation_rule**：R4 主题簇——EK-04（三级评估）+ EK-12（步角色标准）+ EK-17（双粒度）+ EK-23（根因仅失败时采集）。
- **L3 Pattern**：agent 评估应多粒度互补（单步质量 / 整轨成功 / 任务特定 rubric），且昂贵的深度分析（根因）只投向失败样本——评估预算按信息密度分配。
- **证据**：EK-04/12/17/23（S3 + S4）。
- **解释范围扩大**：成本-信息权衡是评估设计的显式维度。Cross-project validation pending。

## Cognitive Models（L4）

### CM-01 · 评估 = 证据生产，不是价值判断
- **稳定关系**：评估系统的核心产物不是"分数"而是"可核查的证据链"（critique + 根因 + 统计模式）；分数只是证据链的摘要。
- **证据**：EK-24（分数+文本绑定）、EK-05（根因聚合）、EK-06（统计模式）——S3/S4。
- **对照**：与"声明 ≠ 证据"（Codex）同族——CLEAR 把该原则工程化为产品形态。

### CM-02 · 评估质量的上限由输入纯度决定
- **稳定关系**：judge 能多好，取决于它能看多纯——IR 保真（EK-03 compact 不丢关键信息）+ 系统提示禁外部元数据（EK-11）共同决定评估可信度上限。
- **证据**：EK-03/11/16（S3/S4）。

### CM-03 · 自动洞察 = 假设检验 + 工程可视化的交集
- **稳定关系**：从评估数据自动生成"洞察"（失败模式/预测路径）必须同时满足统计显著性（EK-08）与可呈现性（dashboard）——任一缺失都是模式幻觉或不可用结论。
- **证据**：EK-06/08/21（S3/S4）。

### CM-04 · 评估工具与运行时 harness 是正交维度
- **稳定关系**：CLEAR（评估侧，只读消费 trace）与 OpenCode/DeepSeek Harness（执行侧，生产 trace）互不依赖——"生产"与"检验"是两个正交的工程维度，可独立演进。
- **证据**：CLEAR 无权限模型（评估工具）+ OpenCode 有权限门（运行时）（跨项目 S3）。

## Methodology（L5）

### M-01 · 评测系统分层构建（IR/评估/聚合/呈现分离）
- **内容**：按"中间表示层 → 评估层 → 聚合层 → 呈现层"分层，每层只消费上一层契约。CLEAR 验证。
- **证据**：EK-02/04/05/21（S3/S4）。

### M-02 · 失败根因只在失败时采集
- **内容**：深度诊断（根因分析）预算投向失败样本；成功样本只需轻量确认。CLEAR 验证。
- **证据**：EK-23（S3）。

### M-03 · judge 可替换性设计
- **内容**：评估系统的 judge 必须是可替换组件（LLM judge / 确定性 judge / 外部函数），不能把 LLM judge 焊死进管线。CLEAR 验证。
- **证据**：EK-09（S3/S4）。

## 分层配比

- EK 27（全部有 links）/ KO 6（R1×1/R2×1/R3×2/R4×2）/ CM 4 / M 3 —— 符合"宽底座 + 窄尖顶"。

# 03 Knowledge Layer — AgentJudgeBench（ARCH-2026-10-12-001）

> Generalized Knowledge（L3/L4/L5 窄尖顶）。每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证。5 个 KO。

---

## KO-01 程序化锚定：评测可信 = 可复算真值 + 对齐度量
- **层**：L4 Cognitive Model
- **aggregation_rule**: R2 因果链簇（EK-01→EK-02→EK-05 完整链：符号打分→对齐公式→GT 污染控制）
- **簇内 EK**: EK-01, EK-02, EK-05
- **陈述**：当"判定"是 LLM 的主观输出时，评测者本身的可靠性必须先被程序化参照物锚定；可靠性的度量方式不是"与人类一致"，而是"与确定性符号公式的偏差"。
- **解释范围扩大论证**：从 AgentJudgeBench 扩展到任何 LLM-as-Judge 场景——引擎 evaluation 层、silver-shield 风险判定、promptfoo 断言——"评测者先被评测"是通用前提。
- **反例攻击**：程序化 judge 本身可能错（如参数语义等价未判）；对齐度高 ≠ judge 正确，只是 judge≈公式。→ 部分成立，见 KO 下 C-01。

## KO-02 提示词是校准器：输出契约决定对齐度
- **层**：L4 Cognitive Model（**Cross-project validation pending**——单项目提示校准经验，盲审计建议标 pending，无第二任务对照）
- **aggregation_rule**: R1 机制簇（EK-07/EK-08/EK-10/EK-11 共享"提示结构→LLM 输出分布"机制，跨 with/without-GT、C2、RQ6 多实例）
- **簇内 EK**: EK-07, EK-08, EK-10, EK-11
- **陈述**：LLM judge 的对齐度不仅是模型能力函数，更是提示词校准程度的函数；默认分档（1.0 vs 0.5）、结构化程度（rubric vs freeform）是可操作的校准旋钮。
- **解释范围扩大论证**：评测/审核类 prompt 设计（评分量表、默认倾向、输出 schema 强制）在任意 LLM 打分场景可迁移。
- **反例攻击**：提示词无法弥补模型推理能力上限（hard+无 GT 仍崩）；→ 降级为"必要非充分"。

## KO-03 无 GT 评测的信息降级：可用 schema 替代答案
- **层**：L3 Pattern
- **aggregation_rule**: R4 主题簇（EK-03/EK-04 围绕 answer-only 分支互补维度：图结构 + 提示策略）
- **簇内 EK**: EK-03, EK-04
- **陈述**：无 ground truth 时，评测提示把"工具 schema"提升为唯一真源，用"plausibly helps"的宽容标准替代精确匹配——即**用任务上下文约束替代答案约束**。
- **解释范围扩大论证**：真实 agent 评测多无 GT；此模式（schema-as-truth + 宽容默认）是通用降级路径。
- **反例攻击**：宽容默认导致对齐虚高（C2 0.5-default 变体即证明默认值敏感）；→ 标注 Hypothesis（跨项目验证 pending）。

## KO-04 GT 污染控制：derangement 是锚定效应的探针
- **层**：L3 Pattern
- **aggregation_rule**: R3 不变量簇（EK-05 的 constraint 汇聚：corrupted GT 喂 judge + true GT 喂程序化参照，两路参照不混）
- **簇内 EK**: EK-05（+EK-01/EK-02 作参照系）
- **陈述**：要测"judge 是否过度依赖 GT"，保持参照物（程序化 judge）不变、只污染其输入（judge 看到的 GT），两路 GT 严格分离——**探针与被探针隔离**。
- **解释范围扩大论证**：任何"输入敏感性"实验（anchoring、prompt injection 效应、上下文污染）都可用"污染输入 + 独立参照"设计。
- **反例攻击**：同 dag_type derangement 的语义相关性未控（相邻 record 可能相似）；→ 见 C-02。

## KO-05 评测基础设施分层：任务 / 图 / 运行解耦
- **层**：L4 Cognitive Model
- **aggregation_rule**: R4 主题簇（EK-12/EK-13/EK-14/EK-15 覆盖运行时/CLI/特判/约定四维）
- **簇内 EK**: EK-12, EK-13, EK-14, EK-15
- **陈述**：大规模评测管道把"评测逻辑"（图+节点+提示）与"运行参数"（模型/路径/批次）分离，CLI 只 patch 不重写，实验条件靠命名约定治理——**评测工程 = 基础设施复用 + 条件可枚举**。
- **解释范围扩大论证**：评测框架（promptfoo/Harness 类）同构；任务包与运行时解耦是复用前提。
- **反例攻击**：run_name 约定非强制（README 明示需用户自律），条件混淆是真实风险；→ 部分成立。

---

## KO 统计

- KO 总数：5（L3×2 / L4×3；L5 方法论 0——无足够跨项目证据，不硬升）
- 聚合规则覆盖：R1/R2/R3/R4 全用
- 平均簇规模：3.2 EK/KO
- 无"同子系统=聚合理由"的假聚合（每条有机制/因果/不变量/主题论证）
- 反例：每条 KO 均声明反例攻击，含降级记录

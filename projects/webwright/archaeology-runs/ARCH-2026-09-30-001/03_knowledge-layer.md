# 03 Knowledge Layer（KO）— microsoft/Webwright

> 每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 边类型；反退化铁律：禁止"都属于某子系统"式聚合。升维候选均经反例攻击（见 06）。

## KO-01 · 验证门：agent 不能单方面宣称完成（L4 Cognitive Model）
- **aggregation_rule**：R2 因果链簇（causal 边）
- **簇内 EK**：EK-12(Self-Reflection Gate) → EK-15(两阶段判定) → EK-16(判定产物即授权) → EK-17(五条硬清单) → EK-18(从零重跑)；causal 链完整（执行→判定→授权→完成）
- **一句话**：执行能力与完成判定权分离——agent 的影响力动作被外部证据门（predicted_label==1）授权，而非自身声明。
- **同类对照**：deepseek-harness 的验证门、Tafcm 的 "证明与执行分离"（corpus 先例）、CI 中"测试通过才算合入"。
- **反例**：local_browser 模式 `require_self_reflection_success=false`——模型自判完成（无外部门）。→ **本 KO 适用域 = 有工件验证的 workspace 模式**；live-browser 模式是反例（见 KO-08 与 06 Counterexample）。

## KO-02 · 上下文治理三件套（L3 Pattern）
- **aggregation_rule**：R1 机制簇（mechanism 边，跨 ≥2 子系统）
- **簇内 EK**：EK-05（截图不自动附）、EK-13（ARIA 裁剪 keep_last_n=1）、EK-14（LLM 压缩 20 步一次）；共享机制：**让 token 预算成为一等约束**，用"降载/裁剪/压缩"三级手段控制上下文。
- **一句话**：长循环 agent 的上下文治理 = 降载观测（不自动附图）+ 裁剪冗余快照（ARIA 只留最近）+ 周期 LLM 压缩（失败不致命）。
- **同类对照**：agent-framework 的 context 治理、MS AF 的 compaction；浏览器型 agent 的 aria 快照主导 token 消耗（lb.yaml 注释自证）。
- **反例**：keep_last_n=0（workspace 模式无 ARIA）→ 治理手段按模式裁剪，非一刀切。

## KO-03 · 代码即动作（Code-as-Action）（L3 Pattern）
- **aggregation_rule**：R1 机制簇（mechanism 边）
- **簇内 EK**：EK-01、EK-02、EK-03、EK-06（共享机制：自由形式脚本 = 动作面）
- **一句话**：动作空间可以是"程序"而非"离散动作"——agent 写代码、执行、看结果、改代码；模型越强，程序化动作面越优于逐 DOM 操作预测。
- **同类对照**：browser-use（observe→predict 单步 DOM 操作）为反范式；SWE-bench 系 agent（写 patch）同源。
- **反例**：预测单步动作仍适用于弱模型/受限动作域（browser-use 在轻量任务上 token 效率更高——对照 corpus）。

## KO-04 · 零 token 技能复用管道（L4 Cognitive Model）
- **aggregation_rule**：R2 因果链簇（causal 边）
- **簇内 EK**：EK-37(gate 准入) → EK-38(learn 蒸馏) → EK-39(evolve 增量生长) → EK-33(route 三决策) → EK-40(零 token 重放) → EK-41(注入 prior)
- **一句话**：把一次 solve 的脚本蒸馏成"可运行的程序技能"（而非模型读的文本），使同类任务的下一次求解零推理成本；失败自动回退 agent。
- **认知增量**：技能库的质量 = 准入门的严格度 + 蒸馏的忠实度 + 增量生长不回退（regression-replay）；self_verify 无 gold 时是弱门（诚实警告自证）。
- **反例**：`_memorized_answer` 丢弃（脚本含答案 verbatim）→ 蒸馏的"方法"才是价值，不是答案本身。

## KO-05 · 外部判定器与证据密度的闭环（L3 Pattern）
- **aggregation_rule**：R4 主题簇（覆盖互补维度）
- **簇内 EK**：EK-15（两阶段判定）、EK-16（判定产物）、EK-19（成功标准策略）、EK-20（证据密度策略）、EK-43（超时重试边界）
- **一句话**：证据驱动的完成判定 = 标准（critical points 可独立验证）× 采集（截图/日志按关键点存）× 判定（两阶段图片 judge）三方闭环；agent 被激励多存证据以提高判定通过率。
- **同类对照**：AProver 的 proof 验证、视觉 grounded 评测（judge 读同一 self_reflect_result.json = 评测可复现）。

## KO-06 · 形状与状态不变量（L4 Cognitive Model）
- **aggregation_rule**：R3 不变量簇（constraint 边汇聚）
- **簇内 EK**：EK-07(strict JSON) 、EK-08(done 降级) 、EK-09(bash -n 预检) 、EK-10(中断流) 、EK-11(step_limit) 、EK-22(key 缺失即失败) ——汇聚到"协议形状与执行状态必须受约束"这一不变量。
- **一句话**：agent harness 的健壮性来自协议硬约束（形状）+ 执行预检（语法）+ 上限（步数/超时）+ 失败快速失败（key 缺失 RuntimeError）——而非依赖模型自律。
- **同类对照**：pydantic strict schema（mb.py 自用 pydantic）；fail-fast 原则。
- **反例**：parse 层降级（demote done）是"容忍 vs 严格"的平衡点——非所有违反都致命。

## KO-07 · 双状态模型适配（L4 Cognitive Model）
- **aggregation_rule**：R4 主题簇（同一主题互补维度 + contrast 边）
- **簇内 EK**：EK-02（workspace-as-state）、EK-06（browser-as-state）、EK-04（双模式一个 loop）、EK-44（模式安全例外）
- **一句话**：同一 agent loop 可服务两种状态模型——磁盘态（无状态浏览器+工件验证，适合同步基准）与浏览器态（持久会话+ARIA 观察，适配实时交互）；状态归属决定验证策略（外部门 vs 模型自判）。
- **同类对照**：browser-use 只有浏览器态；deepseek-harness 以会话态为主；Webwright 显式双模式。

## KO-08 · 完成判定的模式依赖（L3 Pattern，Cross-project validation pending）
- **aggregation_rule**：R2 因果链簇（EK-12 constraint→EK-44；EK-04 causal→EK-12）
- **簇内 EK**：EK-04、EK-12、EK-44
- **一句话**：完成判定的严格度随状态模型变化——工件化状态（workspace）强制外部证据门，会话化状态（live browser）退化为模型自裁；"统一判定"需要统一状态。**（Auditor 强化）**：这是显式对照而非例外注记——Webwright 本身即为"门强度绑定状态模型"的单一仓库证据，跨项目推广需第三例。
- **状态**：Hypothesis（本仓双模式证据充分，跨项目对照仅 browser-use 单模式，缺第三例）。

## KO 质量
- 8 个 KO（7-12 目标区间）；aggregation_rule 覆盖率 100%；簇规模 3-6 EK（3-12 区间）；全部可回溯 02 层。

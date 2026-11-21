# 03 · Knowledge Layer（Generalized KO）— ReviewBench（ARCH-2026-10-17-001）

> 每个 KO 声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证 + 反例。

## KO-01 评测可信度 = 标签来源分层（grounded 人类验证 vs augmented classifier 补充）【Validated Pattern · R2 因果链簇】
- **aggregation_rule**: R2 因果链簇——EK-02（golden 非穷尽）→ EK-01（双指标分离）→ EK-04/05（认领/去重）→ EK-17（offline 不可提交）
- **陈述**: 当 golden set 不可能是穷尽真值时，把"匹配到已验证标签"与"经模型独立判定的新增"分成两类指标，并在报告里显式分层，比单一 F1 更诚实地刻画评测边界。
- **解释范围扩大**: 从"代码审查基准"推广到任何"ground truth 不完整"的评测场景（agent eval / RAG 评估 / 安全检测基准）。对照：引擎 evaluation 层建议"程序化断言优先、LLM-judge 次级信号"——augmented 即次级信号通道。
- **反例**: golden set 完整时（如合成基准有确定答案）双指标冗余；此时 grounded 即全部。
- **Epistemic**: Validated Pattern（仓库内 S4 测试 + S5 文档充分；跨项目 S8 pending → 不进 Principle）。

## KO-02 反偏差是基准可信度的大前提（多源 producer + 分布抽样 + 显式 Threats）【Pattern · R4 主题簇】
- **aggregation_rule**: R4 主题簇——EK-08（多源 producer）、EK-13（分布代表性）、EK-09/11（TP rate 差异与 producer bias 披露）、EK-29（justification 证据链）
- **陈述**: 可信基准不是"更多数据"，而是"标签来源多样 + 分布代表 + 偏差显式披露"三件事；单源 golden（如只靠一个 LLM）会把该模型的偏见固化为 ground truth。
- **解释范围扩大**: 自建评估集方法论模板（能力树 E 层缺口直接命中：规模分布建模→多源 golden→独立验证）。对照 TestJack 34.4% 系统性偏差——单侧 ground truth 的失败案例。
- **反例**: 若目标是"测单一已知 bug 集合"（合成基准），单源可接受，无需多源。
- **Epistemic**: Validated Pattern（S4/S5；跨项目部分）。

## KO-03 评测者与参赛者权威分离是防自我验证的不变量【Validated Pattern · R3 不变量簇】
- **aggregation_rule**: R3 不变量簇——EK-14（fresh container）、EK-15（digest pin + RB_* 契约）、EK-16（secrets/egress 最小化）、EK-17（offline 不可提交）、EK-18（3 runs 均值）
- **陈述**: 一旦参赛者能影响评测环境或结果提交路径，"分数"就不再是独立证据；必须让运行环境无状态、镜像固定、结果提交走独立权威通道。
- **解释范围扩大**: agent 安全评测 / 代码审查 / 沙箱挑战赛通用设计原则（对照 Tafcm"Agent 影响状态的动作必须受控接口化"）。
- **反例**: 自评自测（如本地 tuning）允许同源——但只能用于开发，不能作为对外宣称的证据（EK-17 正是此边界）。
- **Epistemic**: Validated Pattern（S4/S5 文档 + 实现；跨项目 S8 部分）。

## KO-04 LLM-judge 必须配校准与人类兜底（judge 校准约束）【Pattern · R1 机制簇】
- **aggregation_rule**: R1 机制簇——EK-03（同一 classifier 双角色）、EK-06（hill-climb + 重算）、EK-07（文化依赖披露）、EK-28（离线/在线回喂）——同一"LLM 作为判定者"机制跨 labeling/eval 两个子系统。
- **陈述**: 用 LLM 做判定者时，判定 prompt 必须对"人类标签开发集"校准、校准结果必须报告一致性、判定边界（taste/文化）必须披露、并保留人类修正通道（47 findings 人工修正）。
- **解释范围扩大**: 引擎 evaluation 层落地建议 1（judge 校准约束：程序化断言优先、LLM-judge 次级信号）；LLM-as-judge 通用工程。
- **反例**: 无校准的 LLM judge（直接用默认 prompt 打分）在裁判一致性上不可信——TestJack 34.4% 判正确 trial 违规正是未校准 judge 的失败证据。
- **Epistemic**: Validated Pattern（S4/S5）。

## KO-05 匹配的语义判据：fix-one-fixes-other，而非位置重叠【Pattern · R1 机制簇】
- **aggregation_rule**: R1 机制簇——EK-25（匹配判据）、EK-26（many-to-many）、EK-27（chunking 不影响语义）、EK-10（dedup 同 matcher）——同一"语义等价"机制跨匹配/dedup 两处。
- **陈述**: 对"两条 finding 是否同一问题"的判断，应基于"修复一个是否修复另一个"的语义等价，而非行号/文件重叠；同一判据复用于去重与计分可保证一致性。
- **解释范围扩大**: 评测匹配、issue 去重、代码审查 finding 合并的通用模式。
- **反例**: 纯位置匹配（同文件同行号）会误合两条不同问题（matcher 明确拒绝）；对"同位置不同问题"必须分开。
- **Epistemic**: Validated Pattern（S4/S5）。

## KO-06 可复现性 = 全链路 artifact 固定 + 审计留痕【Pattern · R2 因果链簇】
- **aggregation_rule**: R2 因果链簇——EK-23（8 项 artifacts + fingerprint）→ EK-18（3 runs 保留）→ EK-19（仓库镜像）→ EK-29（justification 证据链）——"固定什么→保留什么→可重放"因果链。
- **陈述**: 基准可复现不是"跑同一命令"，而是把输入（PR/镜像/producers/classifier/matcher）、过程（prompt 版本/指纹）、输出（per-PR findings/run 级结果）全部固定并留痕，使任何第三方可重放并审计。
- **解释范围扩大**: 自建基准 / CI 评测 / 回归门禁的可复现工程标准（对照 dsh-memory-evolve 的 evidence chain）。
- **反例**: 只固定代码不固定 LLM 版本（classifier/matcher 换了模型）→ 指标不可比；仅保留聚合均值不保留 run 级 → 方差不可审计。
- **Epistemic**: Validated Pattern（S4/S5）。

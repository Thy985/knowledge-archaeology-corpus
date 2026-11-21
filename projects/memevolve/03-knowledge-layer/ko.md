# Knowledge Layer — KO-01 ~ KO-09

> 升维纪律：KO 只能从 EK 图的簇/链生成；每个 KO 声明 `aggregation_rule`、簇内 EK、解释范围扩大论证、反例与 scope。
> 单项目证据 → 最高 L4 且标 `Cross-project validation pending`；不写成 Law。

## KO-01 记忆系统的差异主要不在“存储”，而在“写入形态”
- **Layer**: L3 Pattern · **Epistemic**: Validated Pattern（单项目） · **Value**: A
- **aggregation_rule**: **R4 主题簇** — EK-33 为核心，聚合 EK-14（append）、EK-17（merge）、EK-20（revise）、EK-23（rewrite）、EK-28（consolidate）。
- **簇内 EK**: 5 条；边类型：mechanism 4 + contrast 3 + causal 1。
- **解释范围扩大论证**: 从“13 个 provider 各自怎么写”提升为“同一抽象下，写入形态是可枚举的设计维度”；该维度可解释为什么同样用 JSON、同样用 MiniLM 的系统在行为上差异巨大（E-090、E-009）。
- **反例（定向搜索 3 条）**:
  1. **反例 A**：Agent-KB 的 append 并不“纯追加”——每次追加后重建全部索引（EK-14），写入与索引维护耦合，不能简单归为 append。
  2. **反例 B**：Cerebra 的 consolidate 同时改变图结构与权重（EK-28），它既是写入也是检索结构变更，跨形态。
  3. **反例 C**：Dynamic-Cheatsheet 的 rewrite 会丢失旧内容（EK-23），与“写入形态”并列时其实改变了“记忆条数”维度，不只是写策略。
- **Scope**: 仅适用于“共享同一 provider 接口、以本地 JSON 为后端”的 MemEvolve 生态；不能推断所有记忆系统。
- **回溯**: EK-33 → EK-14/17/20/23/28 → E-014/E-017/E-020/E-023/E-028

## KO-02 两阶段协议把“任务中途的记忆”变成可选能力
- **Layer**: L3 Pattern · **Epistemic**: Validated Pattern（单项目） · **Value**: A
- **aggregation_rule**: **R2 因果链簇** — EK-02（只有 BEGIN/IN）→ EK-08（注入点）→ EK-07（7/1/5 分布）→ EK-27（Cerebra IN 专有阈值）。
- **簇内 EK**: 4 条；边类型：causal 3 + contrast 2 + constraint 1。
- **解释范围扩大论证**: 从“某些 provider 不返回 IN”提升为“协议阶段数决定记忆能否参与执行中的决策”；因此“13 个系统”并非同一能力等级，评测时应按阶段能力分层。
- **反例（3 条）**:
  1. **反例 A**：BEGIN-only 的系统仍可通过“初始计划”影响后续所有步骤（EK-08），并非对执行无影响。
  2. **反例 B**：IN 阶段返回的内容是 user 消息，模型可能忽略（E-084），所以“有 IN 能力”不等于“被使用”。
  3. **反例 C**：Dynamic-Cheatsheet 只有 IN（EK-23），却在规划阶段完全缺席——说明“阶段能力”是双向缺失，不只是 7/13 的缺失。
- **Scope**: 适用于以 `MemoryStatus` 为请求字段的运行时；其他框架可能用系统提示/工具注入，不适用。
- **回溯**: EK-02/08/07/27 → E-002/E-084/E-021/E-019

## KO-03 “成功才写入”没有统一语义，导致同一轨迹在不同系统中被相反处理
- **Layer**: L3 Pattern · **Epistemic**: Validated Pattern（单项目） · **Value**: A
- **aggregation_rule**: **R4 主题簇** — EK-34 为核心，聚合 EK-25（双门控）、EK-21（缺失默认成功）、EK-14（缺失默认失败）、EK-38（多套 `_is_task_correct` 外部判定）。
- **簇内 EK**: 5 条；边类型：contrast 4 + dependency 2 + causal 2。
- **解释范围扩大论证**: 从“某 provider 的默认值”提升为“摄取门控是记忆质量的真正开关”；任何跨系统比较若不复核门控，就会把“更宽松的写入”误读为“更强的记忆”。
- **反例（3 条）**:
  1. **反例 A**：Expel 对失败也抽 insight（EK-22），说明“失败不写入”并非普遍规则。
  2. **反例 B**：MemP 的失败路径不是“丢弃”，而是“修订旧记忆”（EK-20），因此失败也会改变记忆状态。
  3. **反例 C**：Cerebra 的成功判定可能在 receipt 前被工具调用成功所影响，不能只看 `is_correct`（候选，未证实）。
- **Scope**: 仅覆盖本仓库 13 provider；不外推到其他记忆框架。
- **回溯**: EK-34/25/21/14 → E-038/E-051/E-025/E-058/E-059

## KO-04 检索效果由“预算 + 阈值 + 扩展”三者共同决定，而不是嵌入模型
- **Layer**: L3 Pattern · **Epistemic**: Validated Pattern（单项目） · **Value**: A
- **aggregation_rule**: **R1 机制簇** — EK-09/10/11/12/27 共享“在固定嵌入底座上做策略调参”的机制，跨 4 个独立 provider 文件。
- **簇内 EK**: 5 条；边类型：mechanism 3 + contrast 3 + dependency 2。
- **解释范围扩大论证**: 13 个系统共享同一嵌入模型（EK-09），因此可把差异归因到 top_k、阈值、query 改写、图扩展四类策略；这为“记忆系统对比”提供了可控变量集。
- **反例（3 条）**:
  1. **反例 A**：MobileE 不用 embedding（EK-24），其召回差异无法用该框架解释。
  2. **反例 B**：SkillWeaver 按技能名匹配后直接给 callable（EK-25），召回机制不是“检索条目”而是“选择能力”。
  3. **反例 C**：Agent-KB 的 query 改写（EK-11）改变了输入分布，使 top_k/阈值不可直接横向比较。
- **Scope**: 文本型记忆；API 型与词法匹配型需另立维度。
- **回溯**: EK-09/10/11/12/27 → E-011~E-019

## KO-05 “长期记忆默认关闭”揭示了写入与读取的解耦
- **Layer**: L3 Pattern · **Epistemic**: Validated Pattern（单项目） · **Value**: A
- **aggregation_rule**: **R2 因果链簇** — EK-29（双记忆）→ EK-30（短期抽取/剪枝）→ EK-31（长期写入约束）→ EK-46（跨轮重置）。
- **簇内 EK**: 4 条；边类型：causal 3 + contrast 2。
- **解释范围扩大论证**: 从“一个配置默认值”提升为“记忆的‘生产能力’与‘消费能力’可以独立开关”；这解释了为什么系统中存在大量“存了但从不读”的记忆（E-042）。
- **反例（3 条）**:
  1. **反例 A**：配置可被外部覆盖（E-010），部署方可能开启长期注入，默认值不代表实际运行。
  2. **反例 B**：短期记忆本身跨步骤生效（EK-08），因此“关闭长期”不等于“没有跨步记忆”。
  3. **反例 C**：Evolver 等单层系统没有短期/长期之分，该框架不适用。
- **Scope**: Lightweight 及其同构双记忆实现。
- **回溯**: EK-29/30/31/46 → E-041~E-043/E-047/E-071

## KO-06 无事务的自我修改会把失败样本写进自己的基因型
- **Layer**: L3 Pattern（原申报 L4，经独立验证降级；Cross-project validation pending） · **Epistemic**: Hypothesis→Pattern（单项目强证据） · **Value**: A
- **aggregation_rule**: **R2 因果链簇** — Chain A：EK-35→EK-37→EK-39→EK-40→EK-41→EK-43→EK-45。
- **簇内 EK**: 7 条；边类型：causal 6 + contrast 2。
- **一句话模型**: **当“生成器”直接改写自己的注册表且没有事务边界时，搜索空间会被未完成的变异污染，评估结果不再可信。**
- **解释范围扩大论证**: 从“MemEvolve 的 Creator 无回滚”提升为“任何 self-modifying agent 的通用风险结构”：生成 → 落盘 → 验证，若落盘早于验证且无回滚，则失败候选成为后续生成的输入条件。
- **反例（3 条）**:
  1. **反例 A**：Creator 在写 provider 文件前会检查存在性（E-063），部分重复变异被阻止。
  2. **反例 B**：验证器的自动修复可能把半成品修到可用（EK-45），因此“污染”不必然导致失败。
  3. **反例 C**：下一轮 storage 重置（EK-46）会清掉运行态记忆，但不能清掉代码态变异——反例只削弱影响范围，不推翻模型。
- **Scope**: 自我修改代码/配置的 Agent 系统；非代码生成型系统不适用。
- **标注**: 单项目证据，跨项目验证待补；不得写成 Law。

## KO-07 验证器一旦兼任修复者，就不再是独立质量门
- **Layer**: L3 Pattern（原申报 L4，经独立验证降级；Cross-project validation pending） · **Epistemic**: Hypothesis→Pattern（单项目强证据） · **Value**: A
- **aggregation_rule**: **R1 机制簇** — EK-42（静态子串检查）+ EK-43（默认自动修复）+ EK-45（修复回写真实目录）+ EK-44（隔离快检）。
- **簇内 EK**: 4 条；边类型：mechanism 2 + causal 3 + contrast 1。
- **一句话模型**: **质量门的可信度取决于“判定者”与“被判定对象的修复者”是否分离；二者合一，通过率就不再是质量的证据。**
- **解释范围扩大论证**: 从“PhaseValidator 默认 auto-fix”提升为“任何 validator-in-the-loop 的独立性问题”；可用同一结构检查 CI、代码审查、自动修复机器人。
- **反例（3 条）**:
  1. **反例 A**：自动修复后仍会重新跑快检（E-069），并非“修了就过”。
  2. **反例 B**：修复在隔离副本进行，通过后才回写（E-068/E-070），真实环境未被半成品直接污染。
  3. **反例 C**：如果修复由独立模型/独立进程完成，独立性可恢复；本项目只是同流程内复用，不能推断所有 auto-fix 都不独立。
- **Scope**: 同一流程内“验证+修复”耦合的系统；独立修复服务不适用。
- **标注**: 单项目证据；需跨项目对照（如 CI autofix vs 独立 reviewer）后升级。

## KO-08 用“重置”换公平，代价是跨轮知识不可积累
- **Layer**: L3 Pattern · **Epistemic**: Validated Pattern（单项目） · **Value**: A
- **aggregation_rule**: **R4 主题簇** — EK-46（每轮重置）+ EK-47（锦标赛）+ EK-48（选择）+ EK-50（无测试/简单均值）。
- **簇内 EK**: 4 条；边类型：causal 3 + constraint 2。
- **解释范围扩大论证**: 从“每轮重命名 storage”提升为“可比性与累积性的结构性权衡”；任何“在线进化/持续学习”评估都要回答“是否重置”。
- **反例（3 条）**:
  1. **反例 A**：重置只针对当前 provider 的 storage（E-071），其他 provider 目录不受影响，跨系统污染仍可能存在。
  2. **反例 B**：代码层变异（EK-45）跨轮保留，所以“重置”并不等于“回到初始状态”。
  3. **反例 C**：数据集游标前进（`dataset_cursor += 2x`，E-076），任务分布随轮变化，重置也无法保证同分布比较。
- **Scope**: 以“每轮独立评估”为目标的锦标赛式进化。
- **回溯**: EK-46/47/48/50 → E-071~E-076/E-083/E-091

## KO-09 API 型记忆把“记忆”变成“能力”，同时绕过了审计
- **Layer**: L3 Pattern · **Epistemic**: Validated Pattern（单项目） · **Value**: A
- **aggregation_rule**: **R4 主题簇** — EK-04（TEXT/API 二元）+ EK-25（SkillWeaver）+ EK-26/EK-50（Cerebra 工具记忆）+ EK-01（无删除/审计接口）。
- **簇内 EK**: 4 条；边类型：mechanism 3 + constraint 2 + contrast 1。
- **解释范围扩大论证**: 从“两个 provider 返回 callable”提升为“记忆的表示形态决定可审计性”：文本可 diff/可审阅，代码对象只能执行/不可比较。
- **反例（3 条）**:
  1. **反例 A**：Cerebra 对工具代码做危险检查（E-050），并非完全无审计。
  2. **反例 B**：SkillWeaver 把技能写为 `.py` 文件（E-051），文件本身可被审阅——不可审计的是返回值而非落盘形态。
  3. **反例 C**：API 型记忆只在 2/13 系统中出现，不能代表整个生态。
- **Scope**: 返回 `MemoryItemType.API` 的实现。
- **回溯**: EK-04/25/26/01 → E-004/E-005/E-050/E-051

---

## 汇总

| KO | Layer | 规则 | EK 数 | 状态 |
|---|---|---|---|---|
| KO-01 | L3 | R4 | 5 | Validated Pattern（单项目） |
| KO-02 | L3 | R2 | 4 | Validated Pattern（单项目） |
| KO-03 | L3 | R4 | 5 | Validated Pattern（单项目） |
| KO-04 | L3 | R1 | 5 | Validated Pattern（单项目） |
| KO-05 | L3 | R2 | 4 | Validated Pattern（单项目） |
| KO-06 | L3（原 L4，降级） | R2 | 7 | Pattern（cross-project pending） |
| KO-07 | L3（原 L4，降级） | R1 | 4 | Pattern（cross-project pending） |
| KO-08 | L3 | R4 | 4 | Validated Pattern（单项目） |
| KO-09 | L3 | R4 | 4 | Validated Pattern（单项目） |

- KO 平均簇规模：4.67（范围 4~7），符合 3~12。
- 聚合规则覆盖：100%（9/9 声明 R1/R2/R4）。
- 无“同子系统=聚合理由”的假聚合。
- L3+ 反例：9 个 KO 各 3 条，共 27 条定向反例。

## Pattern candidates（未升为 KO 的中间形态，16 条）
| ID | 模式 | 来源 EK | 为何停在 L3 候选 |
|---|---|---|---|
| P-01 | 全局单文件记忆（rewrite-on-write） | EK-23 | 仅 1 个实现，缺跨实现对照 |
| P-02 | 双角色 guidance（student/teacher） | EK-13 | 仅 Agent-KB |
| P-03 | 图记忆 + 邻居扩展 | EK-27 | 仅 Cerebra |
| P-04 | 工具记忆（memory-as-tool） | EK-26 | 仅 Cerebra，SkillWeaver 形态不同 |
| P-05 | 技能文件即记忆 | EK-25 | 仅 SkillWeaver |
| P-06 | 失败驱动记忆修订 | EK-20 | 仅 MemP |
| P-07 | 前缀差量上下文抽取 | EK-30 | 仅 Lightweight |
| P-08 | 打分-剪枝闭环 | EK-17 | Evolver 独有，但可与 Lightweight 对照 |
| P-09 | 容量触发的 LLM 淘汰 | EK-31 | 仅 Lightweight |
| P-10 | 双库（经验+轨迹） | EK-22 | 仅 Expel |
| P-11 | 词法 tips/shortcuts | EK-24 | 仅 MobileE |
| P-12 | 查询改写后再检索 | EK-11 | 仅 Agent-KB |
| P-13 | 评价值 = 成功率/使用率 | EK-17 | 单实现，缺外部效度 |
| P-14 | 阶段专属阈值 | EK-27 | 仅 Cerebra |
| P-15 | 注释锚点代码注入 | EK-39 | 仅 Creator，但可跨代码生成系统 |
| P-16 | 隔离目录 + 模块缓存清理 | EK-44 | 仅 Validator |

# 03 Knowledge Layer — Webwright（8 KO，L3/L4，aggregation_rule R1-R4）

> 窄尖顶 · 可迁移认知。每个 KO 声明 `aggregation_rule`（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇）+ 簇内 EK + 边类型。
> 反退化铁律：聚合理由不是"同子系统"；无 aggregation_rule 不通过。跨项目声明一律标 `Cross-project validation pending`。

---

## KO-01 Code-as-Action：代码是状态、脚本是产物、契约是 artifact（L4）
- **一句话**：把"操作网页"建模为"生成并执行代码"，系统的持久状态是文件系统里的可重跑脚本，而非浏览器上下文或点击轨迹。
- aggregation_rule：**R4 主题簇**（EK-01, EK-06, EK-07, EK-36, EK-53，边 mechanism/subsystem/causal）
- 解释范围扩大：从"Webwright 的输出格式"（L1/L2）扩展到"任何把 agent 动作编码为程序产物的系统设计"——动作空间=语言（可验证、可重放、可审计），不是坐标（不可读、不可复现）。
- 反例压力：browser-use 用 LLM 决策层+CDP 工具调用（动作=工具调用，状态=浏览器）；Webwright 动作=代码，状态=workspace。两条路线并存 → 本 KO 是"程序化动作"子类，非全称。
- 证据链：README 对比表（Paradigm/Action space/What is state/Loop shape）+ EK-01/36/53。
- 认知状态：Validated Pattern（Webwright 内强证据）；跨项目（browser-use/browser-harness）对照成立但未做统一验证 → `Cross-project validation pending`。

## KO-02 可丢弃浏览器 + 持久工作区：网页任务的"默认记忆"在文件系统（L4）
- **一句话**：浏览器会话是廉价可重建的执行器；证据与中间状态（截图/日志/脚本）落在工作区，成为可审查、可重放、可跨步骤引用的**默认记忆**。**精度注（Reconciliation R3）**：显式持久会话（persistent_local_browser 的会话 JSON、Browserbase 云会话）是与 workspace 并存的例外记忆层——模型为"默认可丢弃、按需持久"分层。
- aggregation_rule：**R1 机制簇**（EK-02, EK-03, EK-04, EK-05, EK-51 共享"浏览器生命周期管理"机制，跨 local_browser / local_workspace / persistent_local_browser 三子系统）
- 解释范围扩大：从"三模式选择"到"无状态执行器 + 有状态证据库"的架构模型——决定何时需要持久会话（跨步状态）何时不需要（每次重建）。
- 反例：persistent_local_browser 的存在证明"有时必须持久"（跨 step 存活）；Browserbase 云会话同理 → KO 是"默认可丢弃、按需持久"的分层模型。
- 认知状态：Validated Pattern；跨项目（browser-harness 的 CDP 连接层）对照成立。

## KO-03 验证即重放：知识入库的裁判是"无模型重跑训练任务"（L4）
- **一句话**：一个可复用技能（代码知识）的入库门槛是它在无模型、无原 solve 的情况下，于干净环境重放训练实例并产出相同答案；通过者才获得 executable 地位。
- aggregation_rule：**R2 因果链簇**（EK-23→EK-24→EK-28→EK-29→EK-30→EK-31→EK-33，沿 causal 边形成 gate→归一化→replay→grade→regression 完整链）
- 解释范围扩大：从"技能入库"到"任何蒸馏知识的验证协议"——knowledge 的验证=独立重放而非自述；回归=旧例必须仍在。
- 反例：shape 模式容忍值漂移（live data）证明"严格重放"有边界；verify off 产生 unverified grade（存在未验证知识位阶）。
- 认知状态：Validated Pattern（测试强证据：grade 三态、regression-replay 拒绝、_norm 折叠）。

## KO-04 知识库污染防护：永不覆盖、永不猜测、永不泄漏（L4）
- **一句话**：可复用知识资产的三个不变量——已验证资产不被失败尝试覆盖（on_fail=reference 只新增不替换）、缺失输入不发明（null 优于 plausible-wrong）、管道内部文本不泄漏进产物（F7）。
- aggregation_rule：**R3 不变量簇**（EK-32, EK-34, EK-36, EK-42, EK-47, EK-54, EK-56, EK-57 经 constraint 边汇聚到"库的纯净性"不变量）
- 解释范围扩大：从 skill_factory 的护栏到通用"知识库防退化"原则——污染以静默方式侵蚀（识别答案毒料、truthy "false"、模板泄漏）。
- 反例：learned_library 里显式允许"答案出现在代码中的部分情形"（词汇表/断言不触发 _memorized_answer）——防护判据是窄的，宁可误收不可误杀已被测试校准。
- 认知状态：Validated Pattern（test_evolve/test_learned_example 强证据）。

## KO-05 库查找前置：路由决策移出 agent 主循环（L4）
- **一句话**：当"用不用已有知识"的决策输入在 agent 启动前就已齐备时，把它解析并注入 prompt，而不是让 agent 在主循环里自己查询——skip 结果零开销，use/adapt 结果第一动作即复用。
- aggregation_rule：**R4 主题簇**（EK-44, EK-45, EK-46, EK-39, EK-40, EK-42, EK-43 覆盖"技能复用决策"互补维度；边 mechanism/causal）
- 解释范围扩大：从 skill_factory prompt.py 到任何"知识增强 agent"设计——检索时机（前置 vs 循环内）是 token/步骤成本的乘数。
- 反例：route 仍保留 agent fallback（run 失败→adapt→agent）——前置解析不是取消 agent，只是把可预计算部分提前。
- 认知状态：Validated Pattern（prompt.py 注释 + route 测试）。

## KO-06 诚实认知状态：未测 ≠ 失败 ≠ 可用（L4）
- **一句话**：系统必须显式区分"验证通过 / 验证失败 / 从未验证"三种状态，并据此分配使用方式（executable 直接跑 / reference 只当先例 / unverified 无地位）——状态坍缩会诱导使用者把未测当已测。
- aggregation_rule：**R3 不变量簇**（EK-31, EK-32, EK-36, EK-24 经 constraint/contrast 汇聚到"认知状态诚实性"不变量）
- 解释范围扩大：从 grade 三态到所有"AI 产出物管理"——与知识分类学（Fact/Observation/Hypothesis/Pattern）同构，是工程层对认知诚实性的实现。
- 反例：self_verify gate 的自述局限（agent 误信的错答案仍会 admit）——系统承认验证盲区并给出升级路径（WebJudge/跨源一致性），说明"诚实"也包括标记验证者自身的盲区。
- 认知状态：Validated Pattern（test 明确"never tested is not the same as tested and failed"）。

## KO-07 严格性递进：形状 → 归一化 → 精确匹配（L3）
- **一句话**：同一"验证答案"问题有三级严格度——shape（非空+结构匹配，容忍 live-data 漂移）、normalize（折叠写法抖动）、exact（同答案同字节归一后相等）；选择哪级由"答案是否漂移"决定，而非统一最严。
- aggregation_rule：**R2 因果链簇**（EK-24→EK-28→EK-29→EK-30 沿 causal 边：gate 选级→归一化→replay 比对→预算循环）
- 解释范围扩大：从 Webwright verify 模式到"动态数据自动化"通用问题——对会动的答案用 exact 匹配 = 拒绝正确工作。
- 反例：strict 在 AS26 vs "AS 26" 案例上曾拒绝正确技能（测试文档明言"spacing is not a logic error"）→ 归一化不是放松，是去噪。
- 认知状态：Validated Pattern（_norm 测试族）。

## KO-08 重画优于修复：独立尝试 × 有限修复的双层预算（L3）
- **一句话**：当一个生成尝试彻底坏掉时，开启全新独立尝试（draw）常比在同一候选上无限修复（round）更快收敛；验证通过即停，两层预算共同限定总成本。
- aggregation_rule：**R1 机制簇**（EK-30, EK-31, EK-35, EK-32 共享"迭代收敛预算"机制，跨 refine/build/evolve 场景）
- 解释范围扩大：从蒸馏代码技能到通用"生成-验证"循环（代码修复、写作、配置生成）——"重画 vs 修补"是成本结构决策。
- 反例：测试同时证明"单 draw 全 rounds 坏 → 什么都不落地"（预算下限保护）——不是无限重试。
- 认知状态：Validated Pattern（test_a_fresh_draw_lands 强证据）；跨项目普遍性待验证。

---
## KO 聚合规则矩阵
| KO | 规则 | 簇内 EK | 簇规模 | 主要边 | 解释范围扩大论证 |
|---|---|---|---|---|---|
| KO-01 | R4 主题 | 01,06,07,36,53 | 5 | mechanism/subsystem | 动作空间=语言 vs 坐标 |
| KO-02 | R1 机制 | 02,03,04,05,51 | 5 | mechanism | 跨 3 子系统浏览器管理 |
| KO-03 | R2 因果 | 23,24,28,29,30,31,33 | 7 | causal | gate→replay→grade 链 |
| KO-04 | R3 不变量 | 32,34,36,42,47,54,56,57 | 8 | constraint | 库纯净性不变量 |
| KO-05 | R4 主题 | 39,40,42,43,44,45,46 | 7 | mechanism/causal | 检索时机前置 |
| KO-06 | R3 不变量 | 24,31,32,36 | 4 | constraint/contrast | 认知状态诚实性 |
| KO-07 | R2 因果 | 24,28,29,30 | 4 | causal | 严格度选择 |
| KO-08 | R1 机制 | 30,31,32,35 | 4 | mechanism | 双层预算收敛 |

配比：48 EK → 8 KO（4×R1/R2 + 2×R3 + 2×R4 无假聚合）✅；KO 平均簇规模 5.5（3~12 区间）✅；全部可回溯 EK/Evidence ✅

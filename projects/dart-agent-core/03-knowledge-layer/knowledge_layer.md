# 03 · Knowledge Layer（Generalized KO）— dart_agent_core

> 窄尖顶。每个 KO 声明 `aggregation_rule`（R1 机制簇 / R2 因果链簇 / R3 不变量簇 / R4 主题簇），簇成立五条件（内聚性/跨实例性/解释范围扩大/可命名/可回溯）。Epistemic 状态诚实标注：跨项目验证前一律 `Principle (Cross-project validation pending)` 或 `Pattern (single-project evidence)`。

## KO-01 · 类型化控制面：agent 生命周期各阶段的治理通道
- **aggregation_rule**: **R1 机制簇** — EK-07/EK-08/EK-09/EK-10/EK-11 共享 mechanism 边（类型化 hook 控制面），跨 agent loop / 工具执行 / 状态持久化三个独立子系统出现。
- **簇内 EK**: EK-07（10 阶段管线）、EK-08（动作短路）、EK-09（工具治理点）、EK-10（id 保留）、EK-11（持久化三态）
- **边类型**: mechanism（共享类型化控制机制）+ subsystem（同属 hook 子系统）
- **解释范围扩大**: 单一项目观察 → "agent 系统把'观察'与'控制'分离：控制只经类型化 hook 通道，工具实现不内嵌治理" 的可迁移模式。
- **Epistemic**: Pattern（单项目证据，S4 测试验证）
- **反例/边界**: hook 是唯一治理通道，但 AgentController.request 的 defaultValue 提供了控制面的旁路（无 handler 立即返回默认值）——旁路是"查询型"控制而非"行为型"控制。

## KO-02 · 评测因果链：trial→record→replay→grade→metric→health 的完整闭环
- **aggregation_rule**: **R2 因果链簇** — EK-45→EK-46（记录-重放依赖盐）、EK-52→EK-46（环境接线）、EK-48→EK-41（评分产出统计）、EK-49→EK-48（超时决定评分资格）、EK-50 依赖 EK-45（健康分析依赖持久化）沿 causal 边形成完整链。
- **簇内 EK**: EK-41/EK-42/EK-45/EK-46/EK-47/EK-48/EK-49/EK-50/EK-51/EK-52
- **边类型**: causal（决策→执行→记录→统计→健康）
- **解释范围扩大**: 评测不是"跑一遍拿分数"，而是"可复现的测量系统"——从录音到重放到评分到健康监测全链路有意识设计。
- **Epistemic**: Pattern（单项目证据，S4）
- **反例/边界**: runner_e2e_test 用 Stub 环境无真实 LLM——评测框架本身不要求真实模型即可验证。

## KO-03 · 取消优先不变量：共享取消是唯一终止权威
- **aggregation_rule**: **R3 不变量簇** — EK-05（工具返回后复查取消）、EK-33（worker 仅共享取消逃逸）、EK-16（取消优先于 resultIsError 回调）经 constraint 边汇聚到同一安全不变量："被取消的任务不得翻转为成功"。
- **簇内 EK**: EK-05/EK-16/EK-33
- **边类型**: constraint（取消优先约束各错误处理路径）
- **解释范围扩大**: 原则级——"局部结果永远不能改写已被共享取消终止的运行"。
- **Epistemic**: Principle (Cross-project validation pending)——已在 dart_agent_core（工具/worker 两处）与 thinkingbox 评测 cancel 语义侧观察，跨项目验证待补。
- **反例/边界**: cancel 后 resume 路径保留（isRunning 保持 true），说明"取消"不是终点而是可恢复的中断（Suspend 语义）。

## KO-04 · 上下文治理四策略：压缩、按需注入、隔离、渐进披露
- **aggregation_rule**: **R4 主题簇** — EK-22/EK-23（压缩与无损性）、EK-28/EK-29（技能按需注入）、EK-26/EK-34（技能/MCP 渐进披露）、EK-31（子代理上下文隔离）围绕"上下文窗口治理"同一主题覆盖互补维度。
- **簇内 EK**: EK-22/EK-23/EK-26/EK-28/EK-29/EK-31/EK-34
- **边类型**: mechanism（渐进披露/按需）+ contrast（压缩 vs 注入）
- **解释范围扩大**: "上下文治理"完整策略谱系——不是单一压缩，而是压缩（历史）+ 按需注入（技能/记忆）+ 隔离（子代理）+ 渐进披露（MCP/技能列表）的组合。
- **Epistemic**: Pattern（单项目证据，S3-S4）
- **反例/边界**: 无压缩器（compressor=null）时运行合法——治理是可选层，不改变 loop 语义。

## KO-05 · 失败不伪装成功（全局不变量）
- **aggregation_rule**: **R3 不变量簇** — EK-03（空响应重试上限）、EK-48（timeout/error 不喂弱 grader）、EK-33（worker 失败为工具结果）、EK-44（judge null 不编造）经 mechanism/constraint 边汇聚到"任何环节的失败都不被转写成成功"。
- **簇内 EK**: EK-03/EK-33/EK-44/EK-48
- **边类型**: mechanism（"不编造/不伪装"共享语义）+ constraint（各环节约束评分/重试/委托行为）
- **解释范围扩大**: 跨 agent 运行时、评测、子代理、judge 四子系统的统一不变量——这是 dart_agent_core 最一致的设计哲学。
- **Epistemic**: Principle (Cross-project validation pending)——thinkingbox（评测）、opencode（agent 运行时）同族观察，跨项目验证待补。
- **反例/边界**: defer 默认 isError=false（deferred 不被标错）——"不伪装失败"同样适用于反向：被延迟而非被拒绝的动作不记为错误。

## KO-06 · 循环检测两级决策：确定性签名优先，概率诊断兜底
- **aggregation_rule**: **R2 因果链簇** — EK-38→EK-40（签名检测触发决策）、EK-39→EK-40（LLM 诊断触发决策）、EK-02→EK-03（预算与重试沿因果边关联）形成"检测→决策→终止"链。
- **簇内 EK**: EK-02/EK-03/EK-38/EK-39/EK-40
- **边类型**: causal（检测→终止）+ contrast（确定性 vs 概率性）
- **解释范围扩大**: 循环终止 = 确定性规则（零成本、无模型）+ 概率性诊断（节流、防误报）的组合决策，且两者共享同一异常出口（loopDetection）。
- **Epistemic**: Pattern（单项目证据，S4）
- **反例/边界**: LLM 诊断仅在完整非工具轮次触发（流式部分轮不阻塞工具执行）——检测器自身尊重"工具调用是进度"的语义。

## KO-07 · 可复现性基础设施：录音、重放、盐与版本历史
- **aggregation_rule**: **R1 机制簇** — EK-20（system prompt/tools 版本历史）、EK-45（record/replay）、EK-46（trial 盐）共享"为复现而记录上下文版本"机制，跨 agent 运行时（EK-20）与评测子系统（EK-45/46）两个独立子系统出现。
- **簇内 EK**: EK-20/EK-45/EK-46/EK-56
- **边类型**: mechanism（复现记录机制）+ causal（上下文版本化→评测确定性）
- **解释范围扩大**: 运行时与评测共享同一心智："要测量/重放的东西必须先被版本化地记录下来"——systemPromptHistory 与 recording store 是同一原则的两面。
- **Epistemic**: Pattern（单项目证据，S3-S4）
- **反例/边界**: 录音只在成功 (request,response) 对记录；失败的调用不入录音——"只复现成功路径"是显式取舍。

## KO-08 · local-first agent 工程形态：平台条件导出 + 入口解耦 + 统一抽象
- **aggregation_rule**: **R4 主题簇** — EK-53（条件导出）、EK-57（两入口解耦）、EK-54（统一客户端抽象）、EK-58（CI 等价治理）、EK-24（跨平台状态存储）围绕"移动优先/本地优先库的工程形态"互补维度。
- **簇内 EK**: EK-24/EK-53/EK-54/EK-57/EK-58
- **边类型**: constraint（平台差异约束抽象/入口设计）+ contrast（文档治理 vs CI 治理）
- **解释范围扩大**: Dart agent 库的"6 平台 + WASM"工程配方：条件导出解决平台差异、入口解耦控制依赖成本、统一抽象隐藏 provider 差异、文档纪律替代 CI 配置文件。
- **Epistemic**: Pattern（单项目证据，S3）
- **反例/边界**: 无 CI 配置文件是显式选择（依赖 pana/analyze 本地门）——自动化 CI 缺失使回归检测依赖开发者自觉。

---

## KO 聚合规则矩阵（交付自检）

| KO | 规则 | 簇内 EK 数 | 簇成立（五条件） | Epistemic |
|----|------|-----------|-----------------|-----------|
| KO-01 | R1 机制簇 | 5 | ✓ | Pattern (S4) |
| KO-02 | R2 因果链簇 | 10 | ✓ | Pattern (S4) |
| KO-03 | R3 不变量簇 | 3 | ✓ | Principle (Cross-project validation pending) |
| KO-04 | R4 主题簇 | 7 | ✓ | Pattern (S3-S4) |
| KO-05 | R3 不变量簇 | 4 | ✓ | Principle (Cross-project validation pending) |
| KO-06 | R2 因果链簇 | 5 | ✓ | Pattern (S4) |
| KO-07 | R1 机制簇 | 4 | ✓ | Pattern (S3-S4) |
| KO-08 | R4 主题簇 | 5 | ✓ | Pattern (S3) |

**配比**: 60 EK → 8 KO（含 2 个原则级候选）。无"同子系统=聚合理由"的假聚合（每条均带 mechanism/causal/constraint/contrast 边语义）。

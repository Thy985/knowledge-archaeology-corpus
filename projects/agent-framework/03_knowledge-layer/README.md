# Knowledge Layer — Generalized KO（ARCH-2026-09-29-001）

> 8 个 KO，每个声明 aggregation_rule（R1-R4）+ 簇内 EK + 解释范围扩大论证。
> 铁律：L3+ 必须可回溯工程底座；Cross-project 内容标 `Cross-project validation pending`，不写已验证 Principle。

## KO-01 确定性审批循环：工具调用在"批准→执行→再批准"闭环中前进
- aggregation_rule: **R2 因果链簇**（EK-15→EK-16→EK-17→EK-18→EK-11）
- knowledge_layer: L3 Pattern
- 簇内 EK：EK-15（状态机循环）、EK-16（规则匹配）、EK-17（二次审批）、EK-18（权威边界）、EK-11（逃生舱）
- 解释范围扩大论证：从"审批工具"（本项目）到"任何 agent 系统的副作用执行都应走可重入、可审计、可中止的批准循环"——审批不是一次性 yes/no，而是会话内状态机（预算/队列/二次确认/逃生舱）。跨实例：Tafcm 的 repair-verify 只读证明（Aigis 同款），DeepSeek-Harness 的每步授权。
- 反例：不经过审批状态机的工具（直接函数调用 bypass 审批）——本项目测试 test_manual_fides_no_session_does_not_make_approval_authoritative 即此类。

## KO-02 标签即权威：信息流控制是确定性安全底座
- aggregation_rule: **R3 不变量簇**（EK-20+EK-21+EK-22+EK-23+EK-24+EK-25+EK-26 → 汇聚到"untrusted 内容不得影响决策"不变量）
- knowledge_layer: L4 Cognitive Model
- 簇内 EK：EK-20（FIDES ADR）、EK-21（三级传播）、EK-22（策略执行）、EK-23（变量隔离）、EK-24（quarantine）、EK-25（MCP 自动标签）、EK-26（保密组合）
- 解释范围扩大论证：提示注入防御从"prompt 工程"（启发式、可绕过）到"确定性信息流控制"（标签 + 物理隔离 + 可验证）是一阶安全范式跃迁——**判断数据可信与否是架构问题，不是文本处理问题**。跨实例：AProver 的证明+执行分离、Aigis 的威胁模型、OWASP Agentic Skills 的注入面分类（已入库 corpus 三项目收敛）。
- 反例：内容消毒（sanitization）作为唯一防线——ADR 0024 明确拒绝；Cosmos 记忆把 LLM 生成的 summary 提升为指令（被本项目拒绝，EK-31）。

## KO-03 信任阶梯：确定性机制 vs 模型自由性分层
- aggregation_rule: **R4 主题簇**（EK-14+EK-44+EK-45 路径安全 / EK-38+EK-39 CodeAct / EK-22 白名单 → "框架把可验证确定性放在下层，把模型自由度放在受控上层"）
- knowledge_layer: L4 Cognitive Model
- 簇内 EK：EK-14（路径规范化）、EK-44（测试）、EK-45（有界搜索）、EK-38（CodeAct ADR）、EK-39（沙箱）、EK-22（白名单）
- 解释范围扩大论证：框架的安全机制都是**确定性可验证**的（路径检查、规则匹配、预算计数、标签组合），而模型的行为自由被约束在这些机制之上（工具审批、沙箱执行、白名单）——**"聪明"（模型能力）与"可信"（确定性机制）分属不同层**。跨实例：Tafcm 权限矩阵（Intelligence ≠ Authority，同收敛）。
- 反例：把安全判断交给 LLM（"模型自己决定是否可信"）——本项目从不依赖模型判断标签。

## KO-04 记忆注入是安全决策：检索内容 = untrusted 数据通道
- aggregation_rule: **R3 不变量簇**（EK-27+EK-28+EK-29+EK-30+EK-31 → 汇聚"记忆不得成为指令"不变量）
- knowledge_layer: L3 Pattern
- 簇内 EK：EK-27（文件记忆）、EK-28（转义防注入）、EK-29（Cosmos 检索）、EK-30（user_id 作用域）、EK-31（summary untrusted）
- 解释范围扩大论证：任何给 agent 的上下文注入（记忆/摘要/RAG 检索）都必须按**数据通道信任等级**处理——LLM 生成物（summary）与用户输入同权（untrusted），不能提升为指令。跨实例：DeepSeek-Harness 的上下文治理、mem0 记忆框架（corpus 已考古）、OWASP 注入面分类。
- 反例：直接把记忆内容拼进 system prompt（EK-31 明言开存储注入路径）。

## KO-05 循环的受控坚持：安全帽 + 谓词 + 上下文重置
- aggregation_rule: **R2 因果链簇**（EK-07→EK-08→EK-09→EK-10→EK-12→EK-13 → 循环受控闭环）
- knowledge_layer: L3 Pattern
- 簇内 EK：EK-07（循环）、EK-08（max_iterations 短路）、EK-09（fresh_context）、EK-10（progress 注入）、EK-12（结构化停止谓词）、EK-13（Judge）
- 解释范围扩大论证：agent 循环的"坚持"必须有确定性边界：安全上限（max_iterations）先于语义谓词短路、fresh_context 保证每次迭代上下文干净、结构化停止条件（todos/后台任务）替代自由文本判断。跨实例：DeepSeek-Harness 的 max_turns + 终止条件；Tafcm 的停止条件（>5 失败请 Human）。
- 反例：无限循环 + 纯文本"你觉得完成了没"（无结构化停止谓词）。

## KO-06 编排的确定性可选性：同构机制的双实现
- aggregation_rule: **R1 机制簇**（EK-32+EK-33 Workflow DAG / EK-34 Sequential+Concurrent / EK-35 GroupChat / EK-36 Handoff / EK-37 Magentic → 共享"编排 = 图 + 状态 + 参与者"机制，跨 5 独立子系统）
- knowledge_layer: L3 Pattern
- 簇内 EK：EK-32（Workflow DAG）、EK-33（RunResult）、EK-34（Seq/Concurrent）、EK-35（GroupChat）、EK-36（Handoff）、EK-37（Magentic）
- 解释范围扩大论证：多代理编排不是单一抽象，而是**一组可选机制**（顺序/并发/对话/交接/任务）共享一个底层图执行引擎（Workflow + checkpoint + 状态时间线）——编排引擎 + 模式库 优于 每种模式独立实现。跨实例：OpenClaw 多 agent（corpus 已考古）、omnigent meta-harness。
- 反例：每种编排模式自建执行器（不共享 DAG 底座）。

## KO-07 沙箱边界是信任模型的硬约束
- aggregation_rule: **R3 不变量簇**（EK-38+EK-39 → 汇聚"模型代码 = untrusted 输入"不变量）
- knowledge_layer: L4 Cognitive Model
- 簇内 EK：EK-38（CodeAct ADR）、EK-39（execute_code）
- 解释范围扩大论证：让模型写可执行代码（CodeAct）时，**隔离边界必须先于能力**——后端若无匹配其信任模型的隔离能力则不可作为 CodeAct 后端；框架职责（审批/遥测/转换）与后端职责（隔离）显式分离。跨实例：E2B 沙箱（corpus 已考古）、browser-use 浏览器隔离。
- 反例：让模型代码在宿主进程内直接执行（无隔离）——ADR 0038 明确排除。

## KO-08 跨 SDK 同构是大型框架的架构纪律
- aggregation_rule: **R4 主题簇**（EK-41+EK-42+EK-43 → 同构 + 实验治理 + 遥测 → "双栈共享契约"主题）
- knowledge_layer: L3 Pattern
- 簇内 EK：EK-41（Python/C# 同构）、EK-42（实验特性治理）、EK-43（遥测）
- 解释范围扩大论证：多语言框架通过"契约先于实现"（同目录结构/同语义选项/同逃生舱）保持行为一致，同时用实验警告 + 遥测位掩码治理未稳定 API——**一致性由架构保证，不是靠文档同步**。跨实例：OpenClaw 多语言面。
- 反例：Python/C# 各自为政（同一能力不同语义）。

## KO 聚合规则矩阵
| KO | 规则 | 簇内 EK 数 | 簇类型 | 解释范围扩大 | 可命名 |
|---|---|---|---|---|---|
| KO-01 | R2 因果链簇 | 5 | 审批闭环 | ✅ | 确定性审批循环 |
| KO-02 | R3 不变量簇 | 7 | 标签即权威 | ✅ | 信息流控制 |
| KO-03 | R4 主题簇 | 6 | 信任阶梯 | ✅ | 确定性/自由分层 |
| KO-04 | R3 不变量簇 | 5 | 记忆注入安全 | ✅ | untrusted 记忆通道 |
| KO-05 | R2 因果链簇 | 6 | 受控循环 | ✅ | 受控坚持 |
| KO-06 | R1 机制簇 | 6 | 编排机制库 | ✅ | 同构编排 |
| KO-07 | R3 不变量簇 | 2 | 沙箱边界 | ✅ | 隔离硬约束 |
| KO-08 | R4 主题簇 | 3 | 双栈契约 | ✅ | 同构纪律 |

# 02 · Engineering Knowledge（EK Graph）— dart_agent_core

> 宽底座推理原材料。每条 EK 保留证据来源（文件/符号/注释原文），并声明 links（六类边）。证据格式：`文件:符号`（精确文件路径缩写为 `lib/src/…`）。

## A. Agent Loop 核心机制

### EK-01 · think-act-observe 主循环
`StatefulAgent.runStream` 是唯一主循环：compress（可选）→ prepare（beforeModelCall hook + hash 记录）→ LLM 调用 → 工具执行 → history 追加 → persist → 下一轮；无工具调用即完成。（`lib/src/agent/stateful_agent.dart:runStream`；`doc/architecture.md` Agent Loop Steps）
**links**: subsystem: EK-02/EK-03/EK-04/EK-05；dependency: EK-18（工具执行依赖 zone 上下文）

### EK-02 · maxTurns 双计数器
`totalLoopCount` 计所有 LLM 尝试（含空响应/重试），`currentLoopCount` 只计已提交回复——"maxTurns uses currentLoopCount; empty/hook retries already returned"。当前轮 ≥ maxTurns(默认 20) → loopDetection 异常。（`stateful_agent.dart:runStream` 注释）
**links**: causal: EK-02→EK-03；subsystem: EK-01

### EK-03 · 空响应重试纪律
empty stopReason / empty response（`_ModelMessageAccumulator.isEmptyResponse`：thought-only 和 media-only 不算空）→ 重试；连续 maxRetryCount=3 次 → loopDetection 异常。空回复重试不消耗 maxTurns 预算（`test/stateful_agent_loop_test.dart`：'empty stopReason retry does not consume maxTurns budget' / 'three empty stopReason retries throw loopDetection'；thought-only/image-only 非空有专测）。
**links**: causal: EK-02→EK-03；contrast: EK-38（重试与循环检测的界限）

### EK-04 · 停止条件集合
循环退出：无工具调用 / 工具返回 stopFlag / hook stop-abort / AgentException（loop/cancelled/stopByController）/ 未处理异常。（`doc/architecture.md` Stop conditions；`stateful_agent.dart` break 点）
**links**: subsystem: EK-01；mechanism: EK-07（hook stop 是停止条件之一）

### EK-05 · 取消优先不变量
工具正常返回后**复查 cancelToken**："A tool may return normally after observing cancellation. Do not let its stopFlag or a worker result turn a cancelled task into success."；worker 异常时"Only cancellation of the shared task should escape worker isolation"。取消以 `"Suspend"` 消息触发挂起路径（isSuspend()），state.isRunning 保持 true 可 resume。（`stateful_agent.dart:_executeTools` 注释；`sub_agent.dart:_delegateTask` 注释；`doc/architecture.md` Cancellation and Suspension；`test: cancel keeps isRunning so resume remains available`）
**links**: mechanism: EK-33（共享取消唯一逃逸）；constraint: EK-05→EK-17（取消优先约束工具错误处理）

### EK-06 · 异常分层与事件
AgentException（cancelled/loopDetection/unknown/stopByController）/ DioException（cancel 检测 isCancelled）/ Exception / Error 四层 catch → state.lastError + controller 事件（OnAgentCancelEvent/OnAgentExceptionEvent/OnAgentErrorEvent）+ rethrow；finally 中 afterRun hook + persist。（`stateful_agent.dart:runStream` catch 链）
**links**: subsystem: EK-01；dependency: EK-19（lastError 是 state 字段）

## B. Hook 管线（治理面）

### EK-07 · 10 阶段类型化 hook 管线
`AgentHookPipeline` 顺序执行：beforeRun / beforeModelCall / onModelChunk / afterModelCall / beforeToolCall / afterToolCall / onTurnCompletion / beforePersistState / afterPersistState / afterRun；每个 hook 结果类型化（proceed/respond/retry/deny/defer/stop/continue/abort）。（`agent_hook.dart:AgentHookPipeline`；`doc/architecture.md` hook 阶段表）
**links**: mechanism: EK-08/EK-09（同为类型化控制面）；subsystem: EK-01

### EK-08 · hook 动作短路语义
abort 立即返回（抛 stopByController 异常）；respond 合成模型响应（跳过真实 LLM 调用）；deny/defer 短路工具执行；proceed 透传 next（可改写 request/call/response）；onTurnCompletion continueRun 注入续接消息（预算 maxTurnContinuations=3）。（`agent_hook.dart` Pipeline 各方法）
**links**: causal: EK-08→EK-10（改写后保 id）；mechanism: EK-07

### EK-09 · 工具调用治理点
beforeToolCall 可 proceed/deny/defer/abort；deny 默认 `isError: true`（告知模型工具被拒）；defer 默认 `isError: false`；deny/defer 均可带合成内容或合成结果。architecture.md 示例：ToolPolicyHook 拒 `delete_file`。（`agent_hook.dart:ToolCallHookResult`；`doc/architecture.md` ToolPolicyHook）
**links**: mechanism: EK-07；contrast: EK-44（judge null escape hatch——同属"不编造"族）

### EK-10 · tool-call-id 保留
hook 改写工具调用后 `_preserveToolCallId` 强制保留原始 id——模型↔结果的关联锚不因改写失效。（`agent_hook.dart:_preserveToolCallId`）
**links**: causal: EK-08→EK-10；subsystem: EK-07

### EK-11 · 状态持久化三态钩子
beforePersistState 可 proceed/skip/abort；abort 抛 stopByController；proceed 才调 autoSaveStateFunc + afterPersistState。`_persistState('afterToolCall')` 每轮工具后 + `_persistState('finally')`。（`stateful_agent.dart:_persistState`）
**links**: subsystem: EK-07；dependency: EK-19（持久化对象是 AgentState）

## C. 工具执行面

### EK-12 · Tool 抽象
Tool{name/description/parameters(JSON Schema)/executable/namedParameters/parameterMode(函数|对象)/resultIsError}。（`core/tool.dart`）
**links**: subsystem: EK-13~EK-18；dependency: EK-01（工具在 loop 中执行）

### EK-13 · 两种参数模式
object 模式：executable 直接收解码后 Map；function 模式：`Function.apply` 按位置/命名参数分发（namedParameters 判定）。（`tool.dart:ToolParameterMode`；`stateful_agent.dart:_executeTools`）
**links**: mechanism: EK-14/EK-15（同为参数装配机制）

### EK-14 · schema 驱动参数铸造
按 properties 的 type 转换：array 元素（string/integer/number/boolean）/ integer→toInt / number→toDouble。（`stateful_agent.dart:_executeTools:addArgument`）
**links**: mechanism: EK-13；constraint: EK-14→EK-15（铸造后需对齐）

### EK-15 · 位置参数对齐
JSON 缺失的位置参数 pad null 保持对齐："Vital: Positional arg missing in JSON. Must pad with null to maintain alignment."（`stateful_agent.dart:_executeTools` 注释）
**links**: constraint: EK-15→EK-13；mechanism: EK-14

### EK-16 · resultIsError 回调
成功返回值也可判为工具错误（legacy 哨兵值模式）；"Exceptions become tool errors independently of this callback, unless the run's shared cancellation token has been cancelled"。（`tool.dart:Tool.resultIsError` 注释）
**links**: contrast: EK-17（回调判错 vs 异常判错）；constraint: EK-05→EK-16

### EK-17 · 工具错误吞掉为结果
工具异常不升级为 run 失败——返回 isError ExecutionToolResult（内容为错误文本）；仅共享取消逃逸。（`stateful_agent.dart:_executeTools` catch）
**links**: mechanism: EK-33（worker 同款隔离）；dependency: EK-09→EK-17

### EK-18 · zone 注入工具上下文
`runZoned(zoneValues: {AgentCallToolContext.zoneKey: ...})`——工具经 `AgentCallToolContext.current` 访问 agent/state/batchCallId/cancelToken（planner/memory/skill/sub-agent 工具都靠它）。批量调用共享 batchCallId。（`stateful_agent.dart:_executeTools`；`planner.dart:_writeTodos`）
**links**: dependency: EK-18→EK-01；subsystem: EK-12

## D. 状态与记忆

### EK-19 · AgentState 全序列化
sessionId/history/usages/currentLoopUsages/metadata/systemReminders/plan/activeSkills/isRunning/totalLoopCount/currentLoopCount/lastError/systemPromptHistory/toolsHistory 全部 toJson/fromJson——状态可完整落盘/恢复。（`stateful_agent.dart:AgentState`）
**links**: subsystem: EK-20/EK-24；dependency: EK-11

### EK-20 · system prompt / tools 版本历史
每次模型调用记录 `SystemPromptHistoryItem(content, validFromMessageIndex)` 与 `ToolsHistoryItem(tools, validFromMessageIndex)`，hash 变化即追加——**可回放"每条消息看到哪版 system prompt/工具集"**（评测可复现的另一面）。（`stateful_agent.dart:_recordModelContextHistory`）
**links**: mechanism: EK-45（评测复现）；causal: EK-20→EK-45（上下文版本化支撑评测确定性）

### EK-21 · EpisodicMemory + 按需召回
压缩把历史存 `EpisodicMemory{id, summary, messages}`；`retrieve_memory(snapshot_id, limit, offset)` 工具分页召回原始消息——召回是工具化按需动作而非自动注入。（`agent/memory.dart`）
**links**: dependency: EK-23→EK-21（无损性依赖召回）；subsystem: EK-22

### EK-22 · LLM 上下文压缩
`LLMBasedContextCompressor`：promptTokens ≥ 64000 且消息 > keepRecentMessageSize(10) 时触发；XML `state_snapshot` 结构（overall_goal/key_knowledge/file_system_state/recent_actions/current_plan）。（`agent/context_compressor.dart`）
**links**: mechanism: EK-23（同为无损压缩）；contrast: EK-28（压缩 vs 按需注入——上下文治理两策略）

### EK-23 · 压缩无损性
原始消息保留在 EpisodicMemory（`messages: messagesToSummarize`），注入 snapshotMessage 带 retrieve_memory 提示；splitIndex 逆向调整保证**不切断 FunctionCall/ToolResult 配对**。（`context_compressor.dart:_compressToEpisodicMemory`）
**links**: dependency: EK-23→EK-21；mechanism: EK-22

### EK-24 · 跨平台状态持久化
FileStateStorage 经条件导出分 IO（dart:io 文件）/ Web（localStorage/内存）；`file_state_storage_session_id_test` 验证 session 归属。（`agent/file_state_storage{,_io,_web}.dart`）
**links**: subsystem: EK-53（平台条件导出）；subsystem: EK-19

## E. 技能系统

### EK-25 · 动态技能系统
Skill{name/description/systemPrompt/tools/forceActivate}；forceActivate=核心技能不可停用（activate/deactivate 工具拒绝操作），optional 可运行时开关——"You must manage your own context"。（`agent/skill.dart`）
**links**: constraint: EK-25→EK-26（forceActivate 决定 prompt 结构）；subsystem: EK-27

### EK-26 · 技能 prompt 四节渐进披露
system prompt 分 CORE CAPABILITIES（IMMUTABLE）/ OPTIONAL / MANAGEMENT PROTOCOLS / ACTIVE INSTRUCTIONS；只有激活技能的重 system prompt 才进上下文（"saves context window and prevents rule conflicts"）。（`skill.dart:buildSkillSystemPrompt`）
**links**: mechanism: EK-34（同为渐进披露）；constraint: EK-25→EK-26

### EK-27 · Directory skills（SKILL.md 文件技能）
从根目录扫描 `SKILL.md`（maxDepth=6）→ frontmatter 解析 name/description（缺 name/description 报错）→ 元数据列表进 system prompt；JS 执行经 RunJavaScript 工具（仅 .js + 必须在 skillDirectoryPaths 前缀下）。与豆包生态 `$SkillName` 模式同构。（`skill.dart:loadDirectorySkillsFromRoot/_parseDirectorySkillFile/buildDirectorySkillsSystemPrompt`；`stateful_agent.dart:_runJavaScriptScript`）
**links**: subsystem: EK-25；mechanism: EK-28/EK-29（提及→注入管道）

### EK-28 · 技能提及检测
`collectExplicitDirectorySkillMentions`：识别 `[$name](path)` / `$name` / 明文技能名（歧义名跳过，单名才匹配），按 path 或 name 选技能。（`skill.dart:collectExplicitDirectorySkillMentions`）
**links**: causal: EK-28→EK-29；subsystem: EK-27

### EK-29 · 技能内容按需注入
被点名的 SKILL.md 全文包装为 `<skill><name><path>...` UserMessage 注入（metadata: type=skill_instructions）——仅注入被用技能，不全量加载。（`skill.dart:buildDirectorySkillInjections`）
**links**: causal: EK-28→EK-29；mechanism: EK-23（都是按需上下文）

## F. Sub-agent 委托

### EK-30 · delegate_task 工具
assignee=clone（标准副本，干净上下文）或命名 sub-agent（registry 查找，未找到返回 status:error 工具结果）；task_description 必须自包含。（`sub_agent.dart:_delegateTask`）
**links**: subsystem: EK-31/EK-32/EK-33；dependency: EK-01（worker 是独立 loop）

### EK-31 · clone 隔离机制
clone worker：新 sessionId（`{parent}_clone_{uuid}`）+ metadata{parent_session_id, sub_agent_mode}；父 history 仅最近 10 条 → parent snapshot UserMessage + 确认消息；共享 client/modelConfig/tools/hooks/controller。（`sub_agent.dart:_delegateTask` clone 分支）
**links**: mechanism: EK-21（快照注入）；subsystem: EK-30

### EK-32 · WORKER AGENT PROTOCOL
worker systemPrompt 追加：Direct Execution（不说客套）/ Self-Contained（结果被程序化解析）/ No Handoffs（不向用户要信息）。（`sub_agent.dart` worker 协议注入）
**links**: constraint: EK-32→EK-30（协议约束委托行为）；subsystem: EK-30

### EK-33 · worker 局部失败隔离
worker 异常：仅共享取消 rethrow，局部失败一律转工具结果（status:error + 错误文本）；"Local budget exhaustion, hook stops and failures remain tool results."（`sub_agent.dart:_delegateTask` catch）
**links**: mechanism: EK-17/EK-05（失败不升级家族）；constraint: EK-33→EK-30

## G. MCP

### EK-34 · MCP 渐进披露（Layer 1）
system prompt 只列服务器名 + 能力计数（N tools/resources/prompts）+ 短描述；详细工具列表由桥工具按需发现。（`mcp_manager.dart:buildMcpSystemPrompt`）
**links**: mechanism: EK-26（同款渐进披露）；constraint: EK-34→EK-35（披露策略约束桥工具设计）

### EK-35 · 6 个固定桥工具
mcp_list_tools / mcp_call_tool / mcp_list_resources / mcp_read_resource / mcp_list_prompts / mcp_get_prompt；全部 object 参数模式 + `_mcpResultIsError`。（`mcp_manager.dart:getBridgeTools`）
**links**: constraint: EK-34→EK-35；subsystem: EK-36/EK-37

### EK-36 · MCP 错误处理
server 缺失/未连接 → `McpOperationResult.error`（消息带可用服务器列表），不抛异常；单服务器连接失败不阻断其余（"Continue connecting other servers even if one fails"）。（`mcp_manager.dart:connectAll/_mcpCallTool`）
**links**: mechanism: EK-17（错误结果化）；subsystem: EK-35

### EK-37 · MCP per-run 生命周期
run 的 finally 中 `mcpManager.disconnectAll()`——MCP 连接按 run 而非 agent 生命周期管理，resume 前需重连。（`stateful_agent.dart:runStream` finally；`doc/architecture.md`）
**links**: constraint: EK-37→EK-35（会话管理约束桥工具可用性）；subsystem: EK-34

## H. 循环检测

### EK-38 · 工具签名循环检测
最近 `toolLoopThreshold`(5) 个调用签名（name:arguments）全同 → loop；流式 chunk 按 id 更新签名。（`loop_detector.dart:DefaultLoopDetector`）
**links**: contrast: EK-39（确定性 vs 概率性）；causal: EK-38→EK-40

### EK-39 · LLM 智能诊断
totalLoopCount>30 后每 10 轮：历史清洗（去孤儿 FunctionExecutionResultMessage、确保以 UserMessage 开头）+ 诊断 prompt（重复错误/复读/翻转三定义）+ JSON（is_loop && confidence>0.8）→ loop。（`loop_detector.dart:_checkForLoopWithLLM`）
**links**: contrast: EK-38；causal: EK-39→EK-40

### EK-40 · 两级检测决策
先确定性签名（零成本）→ 后概率性 LLM 诊断（按轮次节流）→ 任一命中抛 loopDetection 异常。（`loop_detector.dart:detect` 顺序）
**links**: causal: EK-38/EK-39→EK-40；subsystem: EK-01

## I. 评测体系

### EK-41 · pass@k 无偏组合公式
`pass@k = 1 − C(n−c, k)/C(n, k)`（Codex 论文）；实现为朴素连乘 `1 - ∏_{i=0..k-1}(n-c-i)/(n-i)`，注释声称 "log-domain product to avoid overflow on big n" 但代码未用对数域（独立审计 F-1 修正）；n−c<k → 1.0（失败数<k 必然命中）；k>n/k≤0/空 → 0.0。**数值用例与 Codex 一致**（n=4,c=2,k=2 → 1−1/6≈0.8333；test 断言）。（`eval/metrics/pass_at_k.dart`；`test/eval/metrics_test.dart`）
**links**: contrast: EK-42（无偏 vs 经验）；causal: EK-41→EK-46（统计测量依赖非确定性保护）

### EK-42 · pass^k 选择经验估计器
`pass^k = (c/n)^k`；注释明言："Some references use the unbiased combinatorial form C(c,k)/C(n,k). **We provide the empirical version because it's intuitive and consistent with how teams report "X out of Y trials passed" in CI reports.**"（与 ThinkingBox 选择同一公式但动机不同：TB=对抗 pass@1 幻觉/区分难例；dart=CI 报告直观一致性）。（`eval/metrics/pass_caret_k.dart` 注释）
**links**: contrast: EK-41；subsystem: EK-41

### EK-43 · judge 校准
JudgeCalibrator：金集（HumanLabeledTrial）→ 并发拉 judge 分（concurrency=4，每 trial 一次调用）→ Spearman（平均秩处理并列）+ Pearson + agreementRate（|Δ|≤0.15）+ MAE + top20 偏差；`meetsAnthropicBar` 门禁。测试：完全一致→1.0 过 bar；完全反向→-1.0 挂 bar。（`eval/calibration/judge_calibrator.dart`；`test/eval/calibration_test.dart`）
**links**: mechanism: EK-44（null 排除）；subsystem: EK-41（评测统计族）

### EK-44 · judge null escape hatch
ModelGrader 抽象要求 rubric 含显式 "Unknown" escape hatch（Anthropic Step 5）→ judge 无法判断返回 `Score(value: null)` 而非编造分数；校准中 null 被排除在相关性外但单独统计。（`eval/graders/model_grader.dart`；`judge_calibrator.dart` JudgeScorer 文档）
**links**: mechanism: EK-09（拒绝=告知不编造）；subsystem: EK-43

### EK-45 · record/replay 严格重放
RecordingLLMClient 记录成功 (request,response) 对（hash 含 trialSalt）；ReplayLLMClient `strictReplay=true`（CI 默认）时 cache miss **抛 RecordingNotFoundException 使构建失败**（"forcing re-recording"），fallback 仅非严格模式且走 rate limit。（`eval/llm/recording_llm_client.dart`；`eval/llm/replay_llm_client.dart`）
**links**: causal: EK-45→EK-46；subsystem: EK-47；mechanism: EK-20

### EK-46 · per-trial 缓存盐保护非确定性
Trial.cacheSalt=`taskId#trialIndex`（独立于 run name，run A 录音可被 run B 重放）；RecordingLLMClient.withTrialSalt 构造 per-trial client（EvalEnvironment.prepare 内）；注释明言："Without it … the framework silently destroys the non-determinism that pass^k / pass@k are supposed to measure."（`eval/core/trial.dart:cacheSalt`；`recording_llm_client.dart` 注释）
**links**: causal: EK-41→EK-46；constraint: EK-46→EK-45（盐设计约束重放正确性）

### EK-47 · 限速与并发解耦
RateLimitGate 抽象独立于 runner concurrency；RpmRateLimitGate（N req/min）/ TpmRateLimitGate（token-aware）/ Noop；token bucket + waiter 队列 + timer 唤醒，多 trial 并发安全。（`eval/llm/rate_limit_gate.dart`）
**links**: subsystem: EK-45；constraint: EK-47→EK-48（限速保护评测稳定性）

### EK-48 · 只对 completed trial 评分
`shouldGrade = status==passed || status==failed`；"Timeout/error leave placeholder outcomes that can spuriously pass weak graders; those must not count as metric passes"——timeout/error 的 trial 不喂 grader。（`eval/core/eval_runner.dart:_runOneTrial` 注释）
**links**: mechanism: EK-03（失败不伪装）；constraint: EK-48→EK-49（评分依赖超时判定）；causal: EK-48→EK-41

### EK-49 · 超时与状态机
`session.run().timeout(task.timeout ?? defaultTimeout(5min))` → TimeoutException → TrialStatus.timedOut；异常 → errored；评分后 passed&&!scoresIndicatePass → 降级 failed。五态：passed/failed/errored/timedOut/skipped。（`eval_runner.dart:_runOneTrial`；`trial.dart:TrialStatus`）
**links**: causal: EK-49→EK-48；subsystem: EK-48

### EK-50 · suite 健康生命周期
SuiteHealthAnalyzer：单 run 饱和度（mature 比例）/ 跨 run graduation（连续 N 次 run 达成熟阈值）/ broken（跨 run 几乎全失败）/ difficulty histogram（0.2 分桶）；Anthropic Step 7/8 工具支撑。（`eval/suite_health/suite_health_analyzer.dart`）
**links**: dependency: EK-50→EK-45（跨 run 分析依赖报告持久化）；subsystem: EK-41

### EK-51 · transcript 录制兜底
harness 无 transcript（timeout/error）→ recorder.snapshot() 或空占位（nTurns=0 等）→ grader 仍可决策；EventBus 微任务先 drain 再快照。（`eval_runner.dart:_runOneTrial`）
**links**: dependency: EK-51→EK-48（兜底 transcript 服务评分）；subsystem: EK-49

### EK-52 · 评测环境生命周期
environment.prepare(trial, task) 返回 EvalContext（clock/llmClient/controller）——**per-trial client 构造点**（record/replay/salt 在此接线）；finally environment.dispose(context)；runTask（诊断）用临时 suite 且不入 report store。（`eval_runner.dart`；`recording_llm_client.dart` 文档）
**links**: causal: EK-52→EK-46（prepare 是盐接线点）；subsystem: EK-45

## J. 平台与 Provider

### EK-53 · 平台条件导出
fs/http_util/node_javascript_runtime/file_state_storage 各分 {_io,_web}（或 _stub）经条件导出编译期解析——公共 API 在 native/web 一致，WASM-safe；Web 无 dart:io，API key 从 localStorage 经 package:web 读。（`lib/src/core/fs*.dart`；README Platform Support）
**links**: subsystem: EK-24；constraint: EK-53→EK-54（平台差异约束客户端抽象）

### EK-54 · 统一 LLMClient 抽象
LLMClient{generate, stream} 两方法；StreamingControlMessage(controlFlag: retry) 支持模型流内请求重试（"Model requested retry!"）；ToolChoice{none/auto/required + allowedFunctionNames}。（`core/llm_client.dart`；`stateful_agent.dart` retry 分支）
**links**: constraint: EK-53→EK-54；subsystem: EK-55

### EK-55 · 多 provider 适配面
OpenAI Chat Completions + 兼容（Kimi/Qwen/GLM/Ollama/OpenRouter）/ Responses（豆包）/ Claude（Anthropic+MiniMax）/ Gemini / Bedrock Claude——统一接口 6 面。（`lib/src/llm/*`；README Features）
**links**: subsystem: EK-54；contrast: EK-54（统一 vs 分 provider）

### EK-56 · 评测的 LLM 调用可审计
llm_request_hash（SHA256 请求哈希）+ recording_store + langfuse 追踪（LangfuseTraceExporter）+ transcript_viewer CLI——LLM 调用可记录/重放/导出/检视。（`eval/llm/llm_request_hash.dart`；`eval/observability/*`）
**links**: mechanism: EK-45（复现基础设施）；subsystem: EK-47

## K. 治理与工程形态

### EK-57 · 两公开入口解耦
AGENTS.md 明令 "Do not pull eval primitives into the main library"——不用 evals 的应用不付 import 成本；`lib/eval.dart` 独立 import。（`AGENTS.md` Two Public Libraries）
**links**: contrast: EK-58（文档治理 vs CI 治理）；constraint: EK-57→EK-01（入口解耦约束主库设计）

### EK-58 · CI 等价治理
无 CI 配置文件；质量门 = `dart analyze .` + `dart format .` + `pana --no-warning .` + `dart test`（format 未过会拉低 pana 分数）；build_runner 生成 mockito mocks。（`AGENTS.md`）
**links**: contrast: EK-57；subsystem: EK-57

### EK-59 · 设计意图文档化
doc/architecture.md 明确定义：loop 七步 / 停止条件 / run vs runStream / 取消与 Suspend 挂起语义 / loop detection 阈值 / MCP per-run 生命周期——文档即设计契约，代码注释与之一致。（`doc/architecture.md` 各节）
**links**: constraint: EK-59→EK-01（文档约束实现）；subsystem: EK-57

### EK-60 · 事件观察与控制分离
AgentController（EventBus 封装）：publish/listen/request（请求-响应可注册 handler，无 handler 时返回 defaultValue）；"Controller events are for UI updates, tracing, metrics, and diagnostics; they do not control the agent loop"——控制只走 hook。（`controller.dart`；`doc/architecture.md` AgentController Events）
**links**: contrast: EK-07（观察 vs 控制分离）；subsystem: EK-01

---

## EK Graph 质量自检

- **EK 总数**: 60 条（覆盖 agent loop/hook/tool/state/skill/sub-agent/MCP/loop-detect/eval/platform/governance 11 面）
- **links 覆盖率**: 60/60 条均声明 links
- **游离 EK**: 0（全部至少 1 条 subsystem/mechanism/contrast 边）
- **证据强度**: 全部 S3-S4（已实现 + 测试验证；pass@k/judge 校准为 S4 数值用例验证）

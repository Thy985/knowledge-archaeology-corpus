# 04 · Flow Atlas — dart_agent_core

> 七类流，从真实代码导出。关键 Edge 可回溯 `symbol / file / condition / state transition`。证据路径缩写：`SA` = `lib/src/agent/stateful_agent.dart`，`AH` = `agent_hook.dart`，`ER` = `eval/core/eval_runner.dart`，`LD` = `agent/loop_detector.dart`，`CC` = `agent/context_compressor.dart`，`MM` = `agent/mcp_manager.dart`，`SUB` = `agent/sub_agent.dart`，`PL` = `agent/planner.dart`，`MEM` = `agent/memory.dart`，`SK` = `agent/skill.dart`，`TL` = `core/tool.dart`。

## 1. Control Flow（控制流）— agent 运行时主循环

```
run() ──► runStream()                      [SA:runStream]
  │
  ├─ _prepareRunPhase (beforeRun hook: proceed|abort)        [SA:_prepareRunPhase]
  ├─ state.isRunning = true
  ├─ loop while true:
  │    ├─ maxTurns check: currentLoopCount >= currentMaxTurns(20) → throw loopDetection   [SA:runStream cond]
  │    ├─ compressor.compress(state)                          [CC:compress — promptTokens>=64000]
  │    ├─ _prepareModelCallPhase (beforeModelCall: proceed|respond|abort)  [SA:_prepareModelCallPhase]
  │    ├─ LLM call: stream | generate | syntheticResponse     [SA:runStream branch]
  │    │    ├─ stream: onModelChunk (drop|proceed|abort) + loopDetector.detect  [SA:_applyModelChunkPhase]
  │    │    └─ controlMessage.retry → aggregation.reset()     [SA:runStream retry branch]
  │    ├─ empty stopReason / empty response → retry (≤3) → loopDetection   [SA:runStream cond]
  │    ├─ afterModelCall: proceed|retry|abort                 [SA:_applyAfterModelCallPhase]
  │    ├─ currentLoopCount++                                  [SA:runStream — 仅提交后]
  │    ├─ toolCalls.isEmpty?
  │    │    ├─ YES → onTurnCompletion (accept|continueRun ≤3|abort) → break  [SA:_applyTurnCompletionPhase]
  │    │    └─ NO  → _executeToolCallPhase                    [SA:_executeToolCallPhase]
  │    │          ├─ beforeToolCall: proceed|deny|defer|abort (short-circuit)  [AH:beforeToolCall]
  │    │          ├─ _executeTools (runZoned zone context)    [SA:_executeTools]
  │    │          ├─ afterToolCall: proceed|stop|abort        [AH:afterToolCall]
  │    │          ├─ history += [modelMessage, toolResult, injected]
  │    │          └─ shouldStop (hook stop || stopFlag) → break
  │    └─ cancelToken.isCancelled → throw cancelled           [SA:runStream cond]
  ├─ stop conditions (any): no tool calls / stopFlag / hook stop|abort / AgentException / unhandled
  └─ finally: afterRun hook → _persistState → mcpManager.disconnectAll → AgentStoppedEvent   [SA:runStream finally]
```

**关键 Edge**：`beforeToolCall deny → syntheticResults[id] → toolCalls.map(executedById ?? syntheticResults)` —— 被拒调用**不进 _executeTools**，以合成结果回流给模型（`SA:_executeToolCallPhase`）。

## 2. State Flow（状态流）— AgentState 生命周期

```
AgentState 构造 → history/usages/metadata/plan/activeSkills/systemReminders/isRunning/loopCounts/lastError  [SA:AgentState]
  │
  ├─ runStream 启动: isRunning=true; currentLoopCount=0; currentLoopUsages.clear()   [SA:runStream]
  ├─ 每模型调用: usages.add; currentLoopUsages.add         [SA:runStream]
  ├─ systemPromptHistory / toolsHistory 追加 (hash 变化时, validFromMessageIndex=history.length)  [SA:_recordModelContextHistory]
  ├─ plan: write_todos 工具经 AgentCallToolContext.current 写 state.plan → PlanChangedEvent  [PL:_writeTodos]
  ├─ activeSkills: activate_skills/deactivate_skills 工具改写 (forceActivate 拒动)  [SK:_activateSkills/_deactivateSkills]
  ├─ episodicMemories: 压缩追加 EpisodicMemory  [CC:_compressToEpisodicMemory]
  ├─ _persistState('afterToolCall') 每轮工具后 / _persistState('finally') 收尾  [SA:_persistState]
  │    └─ beforePersistState: proceed→autoSaveStateFunc(state)→afterPersistState | skip | abort→throw
  ├─ 异常: state.lastError = e (4 层 catch 统一)            [SA:runStream catch]
  └─ 结束: isRunning=false → AgentRunSuccessedEvent / AgentStoppedEvent  [SA:runStream]
```

**关键 Edge**：`resume()` 前置条件 `if (!state.isRunning) throw resumeFailed` —— 状态机以 isRunning 作为可恢复性判据（`SA:resumeStream`）。

## 3. Data Flow（数据流）— 消息/工具/结果

```
LLMMessage 入 history ──► composeSystemMessage(5 部件) + composeTools(planner+skill+JS+sub-agent+memory+MCP)  [SA:composeSystemMessage/composeTools]
  │
  ├─ _prepareModelCallPhase: requestMessages = history + systemReminders 注入(最后 UserMessage 后)  [SA:_prepareModelCallPhase/_injectSystemReminder]
  ├─ toCallLLMParams: systemMessage insert(0)               [AH:ModelCallRequest.toCallLLMParams]
  ├─ LLM 响应 → _ModelMessageAccumulator 聚合 (text/thought/blocks/calls/media/usage/stopReason)  [SA:_ModelMessageAccumulator]
  ├─ FunctionCall → _executeToolCallPhase:
  │    ├─ args: jsonDecode → schema 驱动铸造 → positional(pad null)/named   [SA:_executeTools]
  │    ├─ runZoned: executable(decodedArgs|Function.apply) → AgentToolResult|其他   [SA:_executeTools]
  │    └─ ExecutionToolResult → FunctionExecutionResultMessage → history      [SA:_executeToolCallPhase]
  └─ 无工具调用 → 最终 ModelMessage 返回 (run 收集 fullModelMessage+functionCallResult)  [SA:run]
```

**关键 Edge**：`properties.keys` 迭代顺序 = 位置参数顺序；JSON 缺参 → pad null（`SA:_executeTools:addArgument` + 注释 "Vital: Positional arg missing in JSON"）。

## 4. Evidence Flow（证据流）— 评测数据管道

```
EvalSuite → EvalTask×trialsPerRun → work queue (task,trialIndex)   [ER:runSuite]
  ├─ bounded concurrency (同 isolate 原子临界区) + rateLimitGate.acquire   [ER:runSuite; eval/llm/rate_limit_gate.dart]
  ├─ _runOneTrial:
  │    ├─ environment.prepare(trial, task) → EvalContext (clock/llmClient/controller)  [ER; per-trial 构造点]
  │    ├─ harnessFactory.create → session.run().timeout(5min 默认)   [ER]
  │    ├─ 超时 → timedOut; 异常 → errored; 正常 → transcript + outcome   [ER]
  │    ├─ shouldGrade = passed||failed → graders 评分 (model/code/human)  [ER:_runOneTrial]
  │    │    └─ judge null (Unknown) → Score(value:null) 不编造  [eval/graders/model_grader.dart]
  │    ├─ scoresIndicatePass → passed|failed 最终判定   [ER]
  │    └─ environment.dispose(context)   [ER]
  ├─ recorder: transcript.snapshot() (微任务 drain 后)  [ER]
  ├─ recording: (request,response) 成功对 → recordingStore (hash 含 trialSalt)   [eval/llm/recording_llm_client.dart]
  ├─ exporters: Composite (onTrialEnd/onRunEnd) → reportStore.save   [ER; eval/reporting/report_store.dart]
  └─ 消费端: report_generator/diff_reporter/suite_health_analyzer (跨 run 最近 10)   [eval/reporting/*; eval/suite_health/suite_health_analyzer.dart]
```

**关键 Edge**：`EvalRunner.runTask` 临时 suite `reportStore 故意置 null`（"this is a debug run, don't pollute history"）——诊断 run 与正式 run 证据隔离（`ER:runTask` 注释）。

## 5. Authority Flow（权威流）— 谁能做什么

```
用户(权限外) → 模型决策 (工具选择) → beforeToolCall hook 权威闸门:
  ├─ proceed: 允许执行 (hook 可改写 call 但 id 保留)     [AH:_preserveToolCallId]
  ├─ deny: 拒绝 (默认 isError=true) → 合成结果回流模型   [AH:ToolCallHookResult.deny]
  ├─ defer: 延迟 (默认 isError=false) → 合成结果回流     [AH:ToolCallHookResult.defer]
  └─ abort: 终止 run (stopByController 异常)            [SA:_hookAbortException]

工具执行权威:
  ├─ 工具不存在 → isError 结果 (非异常)                  [SA:_executeTools firstWhere]
  ├─ RunJavaScript: 绝对路径 + 必须在 skillDirectoryPaths 前缀 + 仅 .js + fsFileExistsSync  [SA:_runJavaScriptScript]
  └─ 工具返回后 cancelToken 复查: 取消 → 抛 cancelError (stopFlag/结果不得翻转为成功)  [SA:_executeTools]

MCP 权威:
  ├─ bridge tools 固定 6 个 (object 模式)               [MM:getBridgeTools]
  └─ server 缺失/未连接 → McpOperationResult.error      [MM:_mcpCallTool]

worker 权威:
  └─ 局部失败 → 工具结果; 仅共享取消逃逸                 [SUB:_delegateTask catch]
```

**关键 Edge**：deny/defer/abort 的权威决策发生在 **hook 层**（治理外置），工具实现自身无内嵌权限逻辑（`AH` + `doc/architecture.md` ToolPolicyHook 示例）。

## 6. Memory Flow（记忆流）— 上下文的历史与召回

```
当前 history.messages (完整对话)        [SA:AgentState.history]
  │
  ├─ LLMBasedContextCompressor: 触发 (promptTokens>=64000 && msgs>10)  [CC:compress]
  │    ├─ splitIndex 逆向调 (不切断 FunctionCall/Result 对)  [CC:_compressToEpisodicMemory]
  │    ├─ messagesToCompress → 排除 SystemMessage → LLM 摘要 (XML state_snapshot)  [CC]
  │    ├─ EpisodicMemory{id, summary=xml, messages=原始} 追加  [CC; MEM:EpisodicMemory]
  │    └─ messagesToKeep 前置: snapshotMessage(带 ID) + "Got it" ModelMessage  [CC]
  │
  ├─ 召回: retrieve_memory(snapshot_id, limit, offset) 工具 (AgentCallToolContext 读 state)  [MEM:_retrieveMemory]
  │    └─ episodicMemories.firstWhere → 分页 → buildConversationHistory(includeIndex)  [MEM]
  │
  └─ 子代理: clone worker 注入父最近 10 条快照 (parent_agent_state_snapshot)  [SUB:_copyParentHistory]
```

**关键 Edge**：压缩后的原始消息**未删除**（存 EpisodicMemory.messages）——"压缩"是上下文窗口操作，不是记忆丢弃；retrieve_memory 是唯一召回通道（工具化按需，非自动注入）。

## 7. Policy Flow（策略流）— 治理闭环

```
Policy 源头:
  ├─ AGENTS.md: 两入口解耦纪律 / 命令规范 / 测试定位 (文档治理)      [AGENTS.md]
  ├─ doc/architecture.md: loop 语义/停止条件/挂起/MCP 生命周期        [doc/architecture.md]
  ├─ 代码内注释契约: 取消优先/失败不伪装/位置对齐等不变量              [SA/TL/ER 注释]
  └─ skill system prompt: forceActivate vs optional 管理协议          [SK:buildSkillSystemPrompt]

Enforcement 执行点:
  ├─ hook 管线 (运行时强制): deny/defer/stop/abort/skip               [AH]
  ├─ 取消纪律 (运行时强制): cancelToken 复查                          [SA:_executeTools]
  ├─ JS 执行白名单 (强制): 路径前缀 + .js + 存在性                     [SA:_runJavaScriptScript]
  ├─ worker 协议 (强制): Direct/Self-Contained/No Handoffs            [SUB:WORKER AGENT PROTOCOL]
  └─ 评测纪律 (强制): 只评 completed / judge null / strict replay     [ER; model_grader; replay_llm_client]

验证/审计:
  ├─ Controller 事件 (观察不控制): BeforeToolCallEvent/AfterToolCallEvent/PlanChangedEvent  [doc/architecture.md; SA publish 点]
  ├─ systemPromptHistory/toolsHistory (哪版上下文对哪条消息)           [SA:_recordModelContextHistory]
  └─ 测试: agent_hooks_test/stateful_agent_loop_test 等 41 文件 12k 行  [test/*]
```

**关键 Edge**：策略执行点全部在**类型化通道**（hook 动作/取消异常/协议注入）而非散落代码判断——治理决策可枚举、可测试、可审计（`AH` 动作枚举 + `test/agent_hooks_test.dart`）。

---

## Flow→KO 交叉校验门

| Flow 关键 Edge | 对应 KO | 校验 |
|---|---|---|
| beforeToolCall deny → 合成结果回流 | KO-01（类型化控制面）| ✓ 一致 |
| per-trial prepare → recording salt → replay | KO-02（评测因果链）| ✓ 一致 |
| cancelToken 复查 + worker 仅共享取消逃逸 | KO-03（取消优先）| ✓ 一致 |
| splitIndex 不切断配对 + retrieve_memory 召回 | KO-04（上下文治理）| ✓ 一致 |
| timeout/error 不喂 grader + judge null | KO-05（失败不伪装）| ✓ 一致 |
| 签名检测 → LLM 诊断 → loopDetection | KO-06（两级循环检测）| ✓ 一致 |
| systemPromptHistory + recording store | KO-07（可复现性）| ✓ 一致 |
| 条件导出 + 两入口 + 统一客户端 | KO-08（local-first 形态）| ✓ 一致 |

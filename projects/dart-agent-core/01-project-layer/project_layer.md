# 01 · Project Layer — dart_agent_core

> 项目地图（L0 事实）。每条可追溯到仓库实际内容（文件/符号）。

## 1. 是什么

- pubspec description：A mobile-first, local-first Dart library for building and evaluating stateful, tool-using AI agents with multi-provider LLM and MCP support.（`pubspec.yaml`）
- README：implements a full agentic loop with tool use, state persistence, multi-turn memory, skill system, context compression, MCP, and agent evals … entirely in Dart, making it suitable for Flutter apps without a Python or Node.js backend.（`README.md`）
- 语言：Dart SDK `^3.9.2`；MIT license（`pubspec.yaml` / `LICENSE`）
- 版本：2.1.7（快照 commit `43c11f1` "chore: release 2.1.7"）

## 2. 两个公开入口（刻意解耦）

| 入口 | 内容 | 治理纪律（AGENTS.md 原文） |
|---|---|---|
| `lib/dart_agent_core.dart` | agent 运行时（clients/agent/tools/skills/state） | 主库 |
| `lib/eval.dart` | 评测子系统（tasks/graders/suites/record-replay/metrics/reporting） | "**Do not pull eval primitives into the main library**" |

## 3. 核心模块地图（lib/src/，17,176 行）

### agent/（运行时）
- `stateful_agent.dart`（2015 行）：orchestrator，`runStream()` 主循环 + `run()` 便捷包装；AgentState 定义；工具执行；系统提醒注入
- `agent_hook.dart`（684 行，`part of stateful_agent.dart`）：10 阶段 hook 上下文/结果类型 + `AgentHookPipeline`
- `planner.dart`（165）：PlanMode none/auto/must + `write_todos` 工具 + PlanState
- `sub_agent.dart`（270）：`delegate_task` 工具 + clone 机制 + WORKER AGENT PROTOCOL
- `skill.dart`（594）：Skill 抽象 + activate/deactivate 工具 + Directory skills（SKILL.md 扫描/解析/提及检测/注入）
- `memory.dart`（143）：EpisodicMemory + `retrieve_memory` 工具
- `context_compressor.dart`（218）：LLMBasedContextCompressor（XML state_snapshot）
- `loop_detector.dart`（226）：DefaultLoopDetector（签名循环 + LLM 诊断）
- `mcp_manager.dart`（538）+ `mcp_session.dart`：MCP 多服务器会话 + 6 桥工具
- `state_storage.dart`/`file_state_storage{,_io,_web}.dart`：AgentState JSON 持久化（IO/Web）
- `controller.dart`（41）：EventBus 封装（观察不控制）
- `javascript_runtime.dart`/`node_javascript_runtime{,_io,_web}.dart`：directory-skill JS 执行（Web WASM-safe）

### core/（基础抽象）
- `llm_client.dart`（135）：统一 LLMClient 抽象（generate + stream）+ StreamingControlMessage(retry)
- `tool.dart`（51）：Tool 定义（JSON Schema + executable + 两种参数模式 + resultIsError）
- `message.dart`（593）：多模态 LLMMessage/ModelMessage/UserMessage + content parts
- `event_bus.dart`：pub/sub + request/response
- `fs{,_io,_stub}.dart`/`http_util{,_io,_web}.dart`：跨平台文件/HTTP（条件导出）

### llm/（多 provider）
- `openai_client.dart`（Chat Completions + 兼容：Kimi/Qwen/GLM/Ollama/OpenRouter）
- `responses_client.dart`（Responses API + 豆包）
- `claude_client.dart`（Anthropic + MiniMax）、`gemini_client.dart`、`bedrock_claude_client.dart`（AWS Bedrock）
- `llm_request_util.dart`（取消检测）

### eval/（评测子系统）
- core/：`eval_runner.dart`（374，并发调度/超时/评分）、`agent_harness_factory`、`eval_suite`、`eval_task`、`trial`（109）、`trial_result`、`transcript`/`transcript_recorder`、`eval_context`/`eval_environment`、`outcome`、`reference_solution`
- graders/：`model_grader.dart`（19，LLM-as-judge 抽象，含 Unknown escape hatch 要求）、`code_grader`（62）、`human_grader`、`grader`、`score`
- metrics/：`pass_at_k.dart`（24）、`pass_caret_k.dart`（24）、`classification_metrics`
- calibration/：`judge_calibrator.dart`（213，Spearman/Pearson/agreement/MAE）、`calibration_report`
- llm/：`recording_llm_client.dart`（110）、`replay_llm_client.dart`（130）、`recording_store`、`llm_request_hash`、`rate_limit_gate.dart`（139，RPM/TPM token bucket）
- loaders/：`suite_loader`、`json_eval_task`、`grader_registry`
- observability/：`langfuse/*`、`trace_exporter`/`composite_trace_exporter`/`jsonl_trace_exporter`、`transcript_viewer`
- reporting/：`diff_reporter`、`report_generator`、`report_store`
- suite_health/：`suite_health_analyzer.dart`（201）、`saturation_status`、`suite_health_report`

## 4. 生命周期（agent 运行时）

1. `run(messages)` / `runStream(messages)`（stateful_agent.dart）
2. `_prepareRunPhase`（beforeRun hook）→ 输入入 history → `_prepareDirectorySkills` → `state.isRunning = true`
3. 主循环：compressor.compress → `_prepareModelCallPhase`（beforeModelCall hook + system prompt/tools hash 记录）→ LLM 调用（stream/generate/synthetic）→ onModelChunk（loop detect）→ afterModelCall → 工具调用或完成
4. 工具调用：beforeToolCall（deny/defer/abort）→ 执行（zone 上下文）→ afterToolCall（stop/inject）→ history 追加 → `_persistState('afterToolCall')`
5. 停止：无工具调用 / stopFlag / hook stop / abort / 异常
6. finally：afterRun hook → persist → **MCP disconnectAll（per-run 生命周期）** → AgentStoppedEvent

## 5. 状态（AgentState 全序列化）

`toJson`/`fromJson` 覆盖：sessionId / history（messages + episodicMemories）/ usages / currentLoopUsages / metadata / systemReminders / plan / activeSkills / isRunning / totalLoopCount / currentLoopCount / lastError / **systemPromptHistory（SystemPromptHistoryItem: content + validFromMessageIndex）** / **toolsHistory（ToolsHistoryItem: tools + validFromMessageIndex）**（stateful_agent.dart）

## 6. 评测生命周期

`EvalRunner.runSuite` → suite.validate() → 任务×trial 队列 → 并发 worker（bounded concurrency + rate limit）→ `_runOneTrial`：environment.prepare（per-trial client 构造点）→ harness create → session.run().timeout → 评分（仅 completed）→ environment.dispose → exporter.onTrialEnd → EvalRunReport → reportStore.save（eval_runner.dart）

## 7. 测试体系（41 文件，12,072 行）

- agent loop：stateful_agent_loop_test（stopFlag/unknown tool/empty stopReason 预算/thought-only 非空/cancel 保持 isRunning）、sub_agent_test、agent_hooks_test、planner_test、loop_detector_test、context_compressor_test、in_memory_skills_test、directory_skills_test
- clients：claude/gemini/openai_stream_usage/openai_document_part/responses/bedrock/image_part/llm_cancel_no_retry
- MCP：mcp_bridge/mcp_integration/mcp_manager + eval/mcp_*（fixtures: mcp_echo_server/mcp_paged_server）
- eval：metrics（pass@k 数值用例）/calibration（Spearman ±1/Anthropic bar）/graders/llm_record_replay/suite_loader/suite_and_report/suite_health/trial/transcript_recorder/runner_e2e（Stub 环境无真实 LLM）/diff_and_store/langfuse_exporter/transcripts_cli
- state：file_state_storage_test/file_state_storage_session_id_test；core/event_bus_test

## 8. 关键配置

- pubspec：SDK ^3.9.2；依赖 mcp_dart ^2.4.0/dio ^5.9.0/http/logging/uuid/crypto/aws_signature_v4；dev: build_runner/lints/mockito/test
- 默认值（代码内）：maxTurns=20、maxTurnContinuations=3、maxRetryCount=3、toolLoopThreshold=5、llmCheckAfterTurns=30、llmCheckInterval=10、llmCheckHistorySize=20、totalTokenThreshold=64000、keepRecentMessageSize=10、agreementTolerance=0.15、topDisagreements=20、defaultTimeout=5min、concurrency=8
- CI 等价（AGENTS.md）：`dart analyze .` + `dart format .` + `dart test` + `pana --no-warning .`（无 CI 配置文件）

## 9. 权限与治理

- AGENTS.md（Qoder 治理文档）：两入口解耦纪律 / 命令规范 / 测试定位
- hook 管线 = 运行时治理通道（deny/defer 工具调用；skip/abort 状态持久化）
- RunJavaScript 路径安全检查（绝对路径 + 必须在 skillDirectoryPaths 前缀下 + 仅 .js）
- 取消优先：共享 CancelToken 取消终止 run，工具/worker 局部结果不得翻转为成功
- MCP 会话 per-run 生命周期（无跨 run 持久连接）

## 10. 外部依赖与连接

- LLM：6 provider 面（OpenAI/Responses/Claude/Gemini/Bedrock/OpenAI 兼容）；MCP stdio+Streamable HTTP（mcp_dart）；Langfuse 可选遥测
- 无 ADR；设计意图在 doc/architecture.md + README + 代码注释
- Corpus 连接：评测统计层（pass@k/pass^k/judge 校准）与 thinkingbox 同构；评测框架与 agentevals/deepseek-harness 同线；agent 运行时与 opencode/hermes-agent/mem0 同线；MCP 桥与 mcp corpus 连接；Tafcm（用户项目，Dart）直接候选

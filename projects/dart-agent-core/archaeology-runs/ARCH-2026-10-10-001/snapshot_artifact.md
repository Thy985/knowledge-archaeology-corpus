# Repository Snapshot Artifact · dart_agent_core

## 1. 快照身份

- **job_id**: ARCH-2026-10-10-001
- **project**: dart_agent_core（memex-lab/dart_agent_core）
- **repository**: https://github.com/memex-lab/dart_agent_core.git
- **mode**: initial（首次考古）
- **clone 方式**: `git clone --depth 1`（浅克隆，1 commit）

## 2. 版本与时间

- **commit SHA**: `43c11f1b9a00a554c27c8ad1839175c2dd0e4a91`
- **commit 信息**: "chore: release 2.1.7"（2026-10-09 11:57:08 +0800）
- **branch**: main（默认分支）
- **version**: 2.1.7（pubspec.yaml，最新 pub 版本）
- **license**: MIT
- **snapshot 时间**: 2026-10-10（UTC+8）

## 3. 项目定位（仓库内声明）

- pubspec description: "A mobile-first, local-first Dart library for building and evaluating stateful, tool-using AI agents with multi-provider LLM and MCP support."
- README: "implements a full agentic loop with tool use, state persistence, multi-turn memory, skill system, context compression, MCP, and agent evals. It connects to mainstream LLM providers (OpenAI, Gemini, Claude, and any OpenAI-compatible API) and handles the orchestration layer — tool calling, streaming, planning, sub-agent delegation — entirely in Dart, making it suitable for Flutter apps without a Python or Node.js backend."
- topics（pub.dev）: agent, agent-framework, llm, mcp, flutter

## 4. 语言与环境

- **语言**: Dart（SDK `^3.9.2`）
- **平台**: 6 平台（Android/iOS/Web/Windows/macOS/Linux）+ WebAssembly 兼容；平台差异经条件导出在编译期解决（`fs_io/fs_web`、`http_util_io/http_util_web`、`node_javascript_runtime_io/web`、`file_state_storage_io/web`）
- **依赖**: mcp_dart ^2.4.0（MCP 协议）、dio ^5.9.0、http ^1.6.0、logging ^1.3.0、uuid ^4.5.2、crypto ^3.0.7、aws_common ^0.7.12、aws_signature_v4 ^0.6.10
- **dev_dependencies**: build_runner ^2.11.0（mockito mock 生成）、lints ^6.0.0、mockito ^5.6.3、test ^1.25.6

## 5. 入口（两个公开库，刻意解耦）

| 入口 | 内容 | 纪律（AGENTS.md） |
|---|---|---|
| `lib/dart_agent_core.dart` | agent 运行时：LLM clients、StatefulAgent、tools、skills、state | 主库 |
| `lib/eval.dart` | 评测子系统：tasks、graders、suites、record/replay、metrics、reporting | **Do not pull eval primitives into the main library**（不用 evals 的应用不付 import 成本） |

## 6. 核心模块地图（lib/src/，17,176 行）

- **agent/**（核心运行时）：`stateful_agent.dart`（orchestrator，~1600 行，think-act-observe loop）、`agent_hook.dart`（hook 管线）、`controller.dart`（事件发布）、`planner.dart`（PlanMode/write_todos）、`sub_agent.dart`（委托/clone）、`skill.dart`（纯 Dart skills + forceActivate）、`directory_skills`（SKILL.md 加载 + JS 执行）、`memory.dart` + `context_compressor.dart`（压缩/召回）、`loop_detector.dart`（重复工具调用检测）、`mcp.dart`/`mcp_manager.dart`/`mcp_session.dart`（MCP stdio/HTTP 桥）、`state_storage.dart`/`file_state_storage*.dart`（JSON 持久化 IO/Web）
- **core/**：`llm_client.dart`（统一抽象）、`tool.dart`（Dart 函数→JSON Schema 工具 + 两种参数模式 + AgentToolResult）、`message.dart`（多模态 content parts）、`event_bus.dart`、`fs*.dart`（跨平台文件系统）
- **llm/**（多 provider）：`openai_client.dart`（Chat Completions + OpenAI 兼容：Kimi/Qwen/GLM/Ollama/OpenRouter）、`responses_client.dart`（Responses API + 豆包）、`claude_client.dart`（Anthropic + MiniMax）、`gemini_client.dart`、`bedrock_claude_client.dart`（AWS Bedrock）
- **eval/**（评测子系统，独立入口）：
  - core/：`agent_harness_factory`、`eval_runner`、`eval_suite`、`eval_task`、`trial`/`trial_result`、`transcript`/`transcript_recorder`、`eval_context`/`eval_environment`、`outcome`、`reference_solution`
  - graders/：`code_grader`、`human_grader`、`model_grader`（LLM-as-judge）、`score`
  - metrics/：`pass_at_k`、`pass_caret_k`（pass^k）、`classification_metrics`
  - calibration/：`judge_calibrator`、`calibration_report`（**judge 校准**）
  - llm/：`recording_llm_client`/`replay_llm_client`/`recording_store`/`llm_request_hash`/`rate_limit_gate`（**LLM 录制重放**）
  - loaders/：`suite_loader`、`json_eval_task`、`grader_registry`
  - observability/：`langfuse/*`（Langfuse 追踪）、`trace_exporter`/`composite_trace_exporter`/`jsonl_trace_exporter`、`transcript_viewer`
  - reporting/：`diff_reporter`、`report_generator`、`report_store`
  - suite_health/：`suite_health_analyzer`、`saturation_status`、`suite_health_report`（**suite 健康/饱和检测**）

## 7. 测试体系（test/，12,072 行，41 文件）

- **agent loop**: `stateful_agent_loop_test`、`sub_agent_test`、`agent_hooks_test`、`planner_test`、`loop_detector_test`、`context_compressor_test`、`in_memory_skills_test`、`directory_skills_test`
- **clients**: `claude_client_test`、`gemini_client_test`、`openai_stream_usage_test`、`openai_document_part_test`、`responses_client_test`、`bedrock_claude_client_test`、`image_part_test`、`llm_cancel_no_retry_test`
- **MCP**: `mcp_bridge_test`、`mcp_integration_test`、`mcp_manager_test`、`eval/mcp_manager_test`、`eval/mcp_connection_config_test`、`eval/mcp_data_types_test`、fixtures（`mcp_echo_server`、`mcp_paged_server`）
- **eval**: `metrics_test`、`calibration_test`、`graders_test`、`llm_record_replay_test`、`suite_loader_test`、`suite_and_report_test`、`suite_health_test`、`trial_test`、`transcript_recorder_test`、`runner_e2e_test`、`diff_and_store_test`、`langfuse_exporter_test`、`transcripts_cli_test`
- **state**: `file_state_storage_test`、`file_state_storage_session_id_test`
- **core**: `core/event_bus_test`

## 8. 关键配置

- `pubspec.yaml`：依赖/平台/topics
- `analysis_options.yaml`：Dart 静态分析配置
- AGENTS.md 指定 CI 等价物：`dart analyze .` + `dart format .`（pana 评分含格式）+ `dart test` + `pana --no-warning .`
- 无 CI 配置文件（.github 不存在）；质量门 = pub.dev pana + 本地 analyze/test/format

## 9. 权限与治理机制

- **AGENTS.md**：给 Qoder 的仓库治理——两个公开入口解耦纪律（eval 不入主库）、命令规范、测试定位方法
- **eval 层有护栏**：`rate_limit_gate`（LLM 调用限速）、`recording/replay`（可复现评测）、`suite_health`（suite 饱和/健康分析）
- **工具面**: 无授权矩阵；工具即 Dart 函数（本地代码），权限取决于宿主 App；MCP server 经 mcp_dart 桥接
- **hook 管线**: agent_hook 可 deny/defer/rewrite 工具调用（运行时治理点）

## 10. 外部依赖与可追溯性

- 运行时外部依赖：6 个 LLM provider（OpenAI/Responses/Claude/Gemini/Bedrock/OpenAI 兼容）；MCP（stdio + Streamable HTTP）；Langfuse（可选遥测）
- 仓库内文档：`doc/architecture(.zh-CN).md`、`doc/eval-guide(.zh-CN).md`、`doc/mcp(.zh-CN).md`、`doc/providers(.zh-CN).md`、`doc/state_and_memory(.zh-CN).md`、`doc/tools_and_planning(.zh-CN).md`——中英双语 6 篇 + README 双语 + AGENTS.md
- 无 ADR 目录；设计决策主要内嵌于 doc/architecture.md 与代码注释

## 11. 与既有 Corpus 的连接（预判）

- **评测统计层同构**：pass_at_k / pass_caret_k（pass^k）/ judge 校准——与 **thinkingbox**（Python，ARCH-2026-10-08-001）的 pass@k/pass^k/goldilocks 统计层**跨语言同构**，可直接对照
- **评测框架线**：suite/trial/grader/transcript/record-replay——与 **agentevals / evolver / deepseek-harness**（eval 层）同线
- **agent 运行时线**：stateful loop/skills/sub-agent/hooks——与 **opencode / hermes-agent / letta-code / mem0** 同线（但 Dart 侧首次）
- **MCP 工具线**：mcp_dart 桥——与 **mcp**（corpus）工具定义线连接
- **用户项目**：Tafcm（Dart/Flutter）直接候选复用对象

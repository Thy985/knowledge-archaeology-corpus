# Snapshot Artifact — ARCH-2026-10-08-001（ThinkingBox）

## 1. 仓库快照
| 字段 | 值 |
|---|---|
| repository | https://github.com/microsoft/thinkingbox.git |
| HEAD commit | `892964e7226044e5463ad188da119df838f4bec1` |
| HEAD message | "Update README with training repository details (#38)" |
| HEAD timestamp | 2026-10-03 16:23:47 -0700 |
| branch | main |
| tags | 无（浅克隆 depth 50 未见 tag；仓库极新） |
| version | 0.1.0（pyproject.toml `[project] version`） |
| commits (浅) | 18（50b1367 Initial commit → 892964e，全部集中于 2026-10） |
| license | MIT（LICENSE.txt + .license-header.txt） |
| requires-python | >=3.10（dev 建议 3.12 + uv） |
| 规模 | 94 个 .py（含 tests/scripts）；thinkingbox 包内 55 py；**核心包 12,832 行** |

## 2. 项目定位（README 溯源）
- **自述**："framework for defining tool mocks as MCP servers, running LLM agents against them, and evaluating agent behavior"
- **Paper**：arXiv 2608.19741 —— "**One Success Isn't Reliability**: ThinkingBox, a Sandbox and Benchmark for Agents in Stateful Business Workflows"（2026）
- **起源**：Microsoft Copilot Studio agentic RL 团队内部项目开源（Nicola Ferri 发起核心框架）
- **三仓库架构**：thinkingbox（本仓库=框架/CLI/评测 harness）/ thinkingbox-data（真实数据集+MCP 工具 server 包）/ thinkingbox-training（RL 后训练：GRPO/LoRA/多轮 MCP rollout/FSDP2）
- 本仓库仅捆绑一个离线 smoke-test 场景 `cloud_drive`（dataset/ + mcp_cloud_drive.py）

## 3. 语言与入口
- 语言：Python（Pydantic v2，lowercase type annotations，3.10+）
- 入口：CLI `tb`（`thinkingbox/cli/main.py:main`，click）——命令：`mcp-start`（Session Proxy）/ `infer`（单测/批量推理）/ `runtest`（解码结果处理）/ `sbs`（候选 vs baseline 对比）/ `agg`（聚合指标）/ `dump-tests` / `tui`（交互）/ `pp`（美化输出）/ `testscript-worker`

## 4. 核心模块（行数实测，12832 行）
| 模块 | 行 | 职责 |
|---|---|---|
| tools/session_proxy.py | 941 | **MCP Session Proxy**：长驻 HTTP server（:7111），前端 MCP 工具进程舰队；session_create/list_tools/call_tool/get_effects/session_destroy 五操作；隔离 session；每 server 一个 stdio JSON-RPC 进程 |
| common/http_client.py | 675 | HTTP 客户端（LLM API/代理调用） |
| common/anthropic_messages_session.py | 596 | Anthropic Messages API session |
| cli/infer.py | 568 | `tb infer` 主流程：load config/test case → 建 agent/user/judge 三 LLM session → 连 proxy → agent loop → 取 effects → 跑断言 |
| cli/agg_main.py | 520 | `tb agg`：JSONL 结果聚合统计（pass^k 指标） |
| common/config_types.py | 497 | 数据集 schema：AgentConfig/ScenarioConfig/TestCase/TestCaseFile + ConfigFile |
| tools/client/worker.py | 476 | MCP 工具 worker 客户端 |
| common/aoai_responses_session.py | 453 | Azure OpenAI Responses API session |
| common/hydrator.py | 446 | 数据水合（config/test case/scenario 加载） |
| common/testrunner.py | 443 | **评测执行核心**：运行测试用例、断言执行、结果产出 |
| common/ordered_parallel_executor.py | 442 | 保序并行执行器（Worker/WorkResult/iter_map_parallel_ordered） |
| cli/tui_main.py | 421 | 交互 TUI |
| fixtures/answer_evaluator.py | 384 | GeneratedAnswerEvaluator fixture（知识 QA/RAG） |
| common/fixtures.py | 380 | fixture 装配（conftest.yaml 依赖注入 + scenario 覆盖） |
| common/aoai_session.py | 373 | Azure OpenAI session（chat completions） |
| common/agent_session.py | 366 | Agent 会话（工具循环、decode_turn_iter） |
| common/mcp_proxy_client.py | 341 | MCP Proxy 客户端（call_tool/list_tools/get_effects） |
| common/chat_types.py | 333 | 对话类型（消息/工具调用/响应） |
| common/agent_user_loop.py | 329 | agent 与模拟用户双循环 |
| common/rubrics_judge.py | 314 | Rubric Judge（评分/reward 计算） |
| tools/client/common.py | 299 | 客户端公共 |
| common/judge.py | 285 | Judge 会话（LLM judge 调用） |
| common/user_simulated_answer.py | 252 | 模拟用户回答生成 |
| cli/runtest_main.py | 213 | run-test：处理解码结果/更新写新结果 |
| common/python_test_file.py | 178 | Python 测试用例文件格式解析 |
| common/tag_types.py | 173 | 测试 tag 类型 |
| common/agent_session_base.py | 168 | agent session 基类 |
| common/llm_session_base.py | 157 | LLM session 基类 |
| tools/mcp_cloud_drive.py | 155 | 捆绑示例 MCP server（文件存储） |

## 5. 核心数据结构（config_types.py schema）
- **ConfigFile**：MCP session proxy 地址 + LLM service 配置（config_o4mini.yaml / config_vllm.yaml）
- **AgentConfig**：`<dataset>/agent/<agent>.yaml` —— agent prompts/配置
- **ScenarioConfig**：`<dataset>/scenario/<scenario>.yaml` —— 每 server 初始状态 + 可用工具列表 + 工具配置
- **TestCase**：`<dataset>/test_case/<file>` —— uid=`<filename>:<testname>` + user query + User-LLM context + test code；双格式（Python `python_test_file.py` / YAML `TestCaseFile`）
- **MCP server 保留方法约定**：`__reserved__init`（setup state）+ tool functions + `__reserved__geteffects`（供评测取状态）

## 6. 状态与数据流（README Architecture 溯源）
- agent loop：`decode_turn_iter()` → LLM 返回 ToolCall → `mcp_proxy.call_tool` → POST /call_tool → session_proxy → ToolDispatcher → MCP server（JSON-RPC stdio）→ 结果回对话 → agent 继续推理
- 评测真相：**效果（effects）= tool 执行后的 session 状态变化**，经 /get_effects 取回用于 judge——"按数据库最终状态+副作用评分"的实现机制
- 会话生命周期：session_create（spawn+init servers，隔离 session）→ list_tools → call_tool ×N → get_effects → session_destroy（terminate+释放）

## 7. 测试体系
- pytest + pytest-asyncio，34 个测试文件（tests/*.py）
- marker：`typesense`（需运行 typesense server 的测试）
- 测试 MCP servers：tests/servers.yaml（mcp-remote streamable-http/sse + mcp-process 三例 + cloud_drive + notepad）
- conftest.py + tb_fixtures.py + mock_session.py（session mock）
- Claude.md 规范：pytest test functions（非 class）、Pydantic v2、uv 运行

## 8. 配置与治理
- 配置：`config/config_o4mini.yaml`（Azure OpenAI + az login）、`config/config_vllm.yaml`（OpenAI-compatible/vLLM）；`THINKINGBOX_DATA` 环境变量（support 文件）；docs/ 15 篇（llm_endpoint_config / session_proxy_config / scenario_tools_config / prompts / rubrics_judge / test_case_format / fixtures / tutorial…）
- 权限与治理（MS 供应链纪律，git log 溯源）：GitHub Actions **pin 全长度 commit SHA**（#29 ea053ba）；workflow token permissions 最小化（#19 126808f）；**CodeQL**（#20 94f2bda）；**Scorecard SARIF**（#15/#16 3fe31b6/0a81b35）；SECURITY.md（MS 标准）；pre-commit（black/isort/flake8，.pre-commit-config.yaml）；pyproject dev 依赖组
- 第三方代码：不 vendor，全部 PyPI 依赖（pyproject 声明）

## 9. 外部依赖（pyproject）
mcp[cli] / fastmcp>=3.4.7,<4 / httpx / pydantic / pydantic[email] / pyyaml / prompt_toolkit / uvicorn / scipy / starlette / numpy / typesense / requests / rich / azure-identity / click / python-dateutil

## 10. 考古范围声明
- 深读：session_proxy / mcp_proxy_client / testrunner / agent loop（agent_session+agent_user_loop）/ judge 族（judge+rubrics_judge+graders+eval_utils）/ ordered_parallel_executor / hydrator+fixtures / config_types+chat_types+tag_types / python_test_file / infer+runtest+agg CLI / worker / mcp_cloud_drive / fixtures/answer_evaluator+user_simulated_answer / docs 关键篇（rubrics_judge/test_case_format/session_proxy_config/prompts/fixtures）
- 声明缺口：thinkingbox-data/thinkingbox-training 两仓库不在本 run（数据与训练在独立仓库）；67.24%/79.9% 统计数字位于论文/数据仓库分析，本仓库可复现的是**评测框架机制**（每任务 20 次连续执行、pass^k、状态真相评分）

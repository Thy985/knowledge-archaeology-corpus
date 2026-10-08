# 01 · Project Layer — ThinkingBox 项目地图

> 本文件全部事实来自仓库实际内容（README.md 520 行 / pyproject.toml / git log / 18 模块源码），行内标注来源。未验证的外部宣称不进入本层。

## 1. 项目定位

ThinkingBox 是微软 Copilot Studio RL 团队（Nicola Ferri 发起，Li Zhuochun 等）开源的 **Agent 评测沙箱与基准框架**：在**有状态业务工作流**（cloud drive、记账、RAG 引用等）中评测 agent 的**长期可靠性**。核心评测单元 = 一个 MCP server 模拟的真实业务环境 + 一个 agent + 一个用户模拟器 + 一个对世界状态（而非对话文本）断言的可执行测试。

来源：README.md（定位/论文引用）、pyproject.toml（name=thinkingbox，version=0.1.0）。

## 2. 语言与运行时

- Python >= 3.10；依赖：fastmcp（<4，pin 于 #27 40c1212）、mcp、pydantic>=2、click、rich、prompt_toolkit、numpy、scipy、httpx、pyyaml
- 异步为主（asyncio）；测试执行经子进程（`python -m thinkingbox.cli.testscript_worker`）
- 后端兼容：Azure OpenAI（aoai_session / aoai_responses_session）、Anthropic（anthropic_messages_session）、自定义 factory（config_types.LLMSessionConfigT）

## 3. 入口（CLI）

```
tb = thinkingbox.cli.main:main  (click)
├── infer       # 评测主命令：agent 解码 + 测试执行 → JSONL/YAML
├── agg         # 聚合指标：pass@k / pass^k / goldilocks / CI
├── sbs         # A/B 对比：baseline vs candidate 贝叶斯显著性
├── run-test    # 从已有 DecodeResult 重跑测试（解码与测试解耦）
├── mcp-start   # 启动 Session Proxy（端口 7111）+ MCP server 组
├── dump-tests  # 导出测试用例
├── pp          # 打印解码结果
├── tui         # 交互界面
└── testscript-worker  # 内部：子进程测试执行入口（\0 分隔协议）
```

## 4. 模块地图（核心行数，wc -l 实测）

| 模块 | 行数 | 职责 |
|---|---|---|
| common/testrunner.py | 443 | 测试执行器：TestScript（exec）/ TestScriptSubprocess（子进程）/ TestScriptDebug（IDE 调试） |
| common/session_proxy.py | 941 | Session Proxy（Starlette + FastMCP）：REST + /mcp + 认证 + GC |
| common/mcp_proxy_client.py | 341 | MCP 客户端（代理）：五端点 + 超时不重试 + session context |
| common/agent_session.py | 366 | Agent 会话：decode_turn_iter 主循环 / direct_response / end-turn |
| common/agent_user_loop.py | 329 | agent×模拟用户双循环 + 五态 finish_reason + 历史重放 |
| common/ordered_parallel_executor.py | 442 | 保序并行执行器（watchdog / SIGUSR1 取消 / 防死锁） |
| common/config_types.py | 497 | 全部配置模型（ConfigFile/Scenario/TestCase/Agent/Rubric）+ deprecated 迁移 |
| common/chat_types.py | 333 | Message 五类 / TestContext / TestResult / DecodeResult / FinishReason |
| common/judge.py | 285 | LLM Judge：legacy（yes/no）+ motivation（JSON）+ 注入防护 |
| common/rubrics_judge.py | 314 | Rubric 评分管道（加分×罚项 / threshold / 并行） |
| common/hydrator.py | 447 | Dataset 加载 / conftest 分层 / 标签合并 / 水合 |
| common/fixtures.py | 369 | Fixture DI：拓扑排序 / 环检测 / ExitStack |
| common/ordered_parallel_executor.py | 442 | 见上 |
| common/infer（cli/infer.py） | 568 | TBWorker / async_main / run_metadata |
| cli/agg_main.py | 521 | 指标聚合（pass@k/pass^k/goldilocks/CI） |
| cli/sbs_main.py | 150 | A/B 贝叶斯对比 |
| tools/client/common.py | 300 | ToolDispatcher / ServersConfig / server 配置 |
| tools/client/worker.py | 477 | mcp_worker 进程内队列 + 三种 transport client |
| tools/mcp_cloud_drive.py | 155 | 示例 server（__reserved__init/geteffects） |
| fixtures/answer_evaluator.py | 385 | 答案评估 fixture（语义/引用/质量三维） |

## 5. 核心数据结构

- **Message 五类**（chat_types.py）：`Text`（tag: think/text/direct；`is_visible` = tag≠think）、`ToolCall`、`ParallelToolCall`、`ToolResponse`、`ToolCallResponse`（tool_call+tool_response 配对，存 conversation.metadata["tool_calls"]）
- **TestContext**（评分输入快照）：`response`（最后可见 assistant text）/ `tool_direct_responses`（tag=direct 的注入文本）/ `effects`（server 世界状态）/ `tool_calls`（全记录）/ `messages` / `session_id` / `init_result` / `query()`（首个非 dummy user Text）
- **TestResult**：`result` / `reward`（可 0..1）/ `is_system_error`（非断言异常=true）/ `tb`（完整 traceback）/ `lineno`+`line_content` / `prints`（MockPrint 捕获）/ `metadata["judge_motivation"]`
- **DecodeResult**：`messages` / `test_result` / `test_context` / `test_tags` / `tools` / `user_llm_history` / `usage` / `metadata` / `is_system_error` / `finish_reason`
- **FinishReason 八值**（Literal）：`done` / `end_turn_tool` / `agent_error` / `agent_limit` / `user_limit` / `user_done` / `no_user_llm` / `skipped`
- **HydratedTestCase**：uid（`<file>:<testname>`）/ agent / scenario / query / test_code / bot_instructions / user_context / max_user_sim_turns（默认 10）/ max_agent_sim_turns（默认 sys.maxsize）/ init / metadata / tags / history / fixtures

## 6. 生命周期（一次评测 run）

1. **配置**：`config.yaml`（ConfigFile：mcp_proxy / judge_model / judge_type / orchestrator.agent_model / user_model 可选）+ dataset 目录（agent/<name>.yaml、scenario/<name>.yaml、test_case/<file>.py|yaml + .meta.yaml + conftest.yaml）
2. **水合**：hydrator 按 uid 加载并合并标签/历史/fixtures → HydratedTestCase（skip 标签过滤）
3. **Session 创建**：`merge_init_config(world_state, tc.init)` → `MCPProxyClient.session_context_from_config(available_tools=...)` → Session Proxy 内初始化 MCP server 组（`__reserved__init` 校验每个 available_tool 存在，失败回滚 destroy）
4. **双循环**：`run_agent_user_loop`（agent decode_turn_iter × UserSimulator）→ 五态 finish_reason 终止
5. **测试执行**：TestScriptSubprocess 子进程跑 `test_code`（fixtures DI + judge）→ TestResult（AssertionError=失败，其余=system error）
6. **落盘**：JSONL（每个 DecodeResult 一行）+ `<stem>_run_metadata.yaml`（setup/agent/user 配置 + 起止时间）
7. **聚合**：`tb agg` → pass@k / pass^k / goldilocks / 95% CI；`tb sbs` → P(pCand>pBase)

## 7. 状态与数据流（高层）

```
dataset ─水合→ HydratedTestCase ─decode→ conversation（Message 流）
    MCP servers（init→工具→effects）─get_effects→ TestContext（评分输入快照）
    TestContext ─TestScriptSubprocess→ TestResult（断言真相）
    DecodeResult ─JSONL→ tb agg / tb sbs（贝叶斯统计）
```

## 8. 配置体系

| 配置 | 关键字段 | 来源 |
|---|---|---|
| ConfigFile | mcp_proxy（host/port/api_key/require_auth）/ judge_model / judge_type（legacy|motivation）/ user_model（可选）/ orchestrator（type=thinkingbox + agent_model）/ user_can_end_conversation | config_types.py |
| AgentConfig | builtin_tools / system_instructions / model | config_types.py |
| ScenarioConfig | world_state / tools（ToolDefOverride：override_description/direct_response/is_end_turn）/ bot_instructions / tags / fixtures / metadata | config_types.py |
| TestCase | uid / scenario / query / test_code / history / max_user_sim_turns / max_agent_sim_turns / init / format_query | config_types.py |
| ServersConfig | use_internal_servers / servers（mcp-process / mcp-remote：sse|streamable-http + credential） | tools/client/common.py |
| env | THINKINGBOX_DATA（dataset 根）/ THINKINGBOX_SESSION_PROXY_KEY（proxy 认证） | session_proxy.py / README |

## 9. 治理与安全（git log 实证）

- **#29 ea053ba**：pin GitHub Actions 到 full-length commit SHA（供应链）
- **#19 126808f**：token 权限最小化
- **#20 94f2bda**：CodeQL
- **#15/#16**：Scorecard
- pre-commit：black / isort / flake8；SECURITY.md 存在（README 声明）
- 运行时认证：Session Proxy `require_auth`（除 /health 外全要 Bearer token：api_key 或 THINKINGBOX_SESSION_PROXY_KEY）

## 10. 依赖与外部系统

- **MCP 生态**：FastMCP（server 框架）/ mcp SDK（ClientSession/stdio/sse/streamable-http）
- **LLM 后端**：Azure OpenAI（含 AOAI Responses API）/ Anthropic / 自定义 factory；Azure 凭据（azure_credential / azure_interactive_browser_credential / credential_factory）
- **统计**：numpy / scipy（betainc/betaincinv/betaln / Gauss-Legendre）
- **部署形态**：Session Proxy（uvicorn workers=1）+ MCP server 子进程/远程

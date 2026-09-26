# 01 · Project Layer — browser-use 项目地图

## 1. 项目定位
- 类型：浏览器 Agent 库（Python）——"Navigate the web like a human does"
- 版本：0.13.10（pyproject.toml）；Python >=3.11,<4.0；uv 构建
- 运行方式：库 API（`Agent(task=...).run()`）/ CLI（基于 browser-harness）/ MCP 服务器 / beta 终端 agent
- 生态位置：开源事实标准（115,743★），Claude Code/Codex/Cursor/Hermes/OpenClaw 直接可用

## 2. 架构（CLAUDE.md + 代码证据）
```
用户任务 → Agent（agent/service.py 主循环）→ 浏览器状态摘要（dom/）→ LLM 决策（llm/）
    → 动作执行（tools/ + controller）→ ActionResult → 历史/压缩（agent/message_manager）
    → 验证（judge + watchdog 监视）
BrowserSession（browser/session.py）：事件驱动双层架构
    ├── 高层事件处理（agent/tools）
    ├── 直接 CDP/Playwright 调用（cdp-use）
    └── bubus EventBus → 16 个 Watchdog（安全/下载/captcha/权限/DOM/崩溃/HAR/截图/…）
```

## 3. 核心模块
| 模块 | 路径 | 职责 | 规模 |
|---|---|---|---|
| Agent 主循环 | agent/service.py | 任务→动作循环、预算门控、循环检测、失败恢复 | 4,163 行 |
| Beta 服务 | beta/service.py | Rust SDK 终端 agent（JSON-RPC + agent tools） | 6,810 行 |
| BrowserSession | browser/session.py | CDP 会话/标签/生命周期 | 4,153 行 |
| Watchdogs | browser/watchdogs/ | 16 个事件驱动监视器 | ~7,000 行 |
| DOM 处理 | dom/ | AX tree→交互元素序列化、Markdown 提取 | ~4,700 行 |
| 工具注册表 | tools/ | 动作全集（ActionModel）+ 注册执行 | 2,327 行 |
| LLM 抽象 | llm/ | 12+ 供应商归一化 + 工厂 | ~3,000 行 |
| MCP 服务器 | mcp/server.py | MCP 工具面（browser_navigate 等 12+） | 1,294 行 |
| Sandbox | sandbox/ | 远程隔离执行（cloudpickle + SSE） | — |
| CLI | cli.py | 基于 browser-harness 的命令行 | — |

## 4. 核心数据结构
- `AgentOutput`（agent/views.py:388）：`current_state` + `action[]`；变体 NoThinking / FlashMode（views.py:437-487）
- `ActionResult`（views.py:307）：结构化动作返回（is_done/success/extracted_content/attachments）
- `AgentState`（views.py:251）：agent_id / message_manager_state / loop_detector / consecutive_failures
- `BrowserStateSummary` / `SerializedDOMState`（dom/views.py:936）：DOM 快照 + selector_map
- `EnhancedDOMTreeNode`（dom/views.py:379）：backend_node_id / is_visible / 交互属性
- `PageFingerprint`（views.py:95）：页面停滞检测
- `ActionLoopDetector`（views.py:157）：动作 hash 滚动窗口 + 页面指纹
- `AgentSettings`（views.py:59）：控制面参数（详见 02）

## 5. 生命周期
1. `Agent()` 初始化：settings/llm/browser/profile/message_manager/action models/skills 注册（service.py:136-780）
2. `run()` → step 循环（service.py:1035）：prepare → get_next_action → execute_actions → post_process → finalize
3. 终止条件：任务 done / max_steps / max_failures / Ctrl+C 停止暂停（_check_stop_or_pause）
4. 失败恢复：consecutive_failures 计数 → final_response_after_failure 最后恢复调用 → 强制 done

## 6. 主要配置（BrowserProfile + AgentSettings + env）
- 浏览器：headless / channel / allowed_domains / prohibited_domains / keep_alive / captcha_solver / cookie_whitelist_domains / proxy / permissions / wait_between_actions / cross_origin_iframes / max_iframes
- Agent：use_vision / max_failures=5 / max_actions_per_step=5 / use_thinking / flash_mode / use_judge / message_compaction / enable_planning / planning_replan_on_stall=3 / planning_exploration_limit=5 / llm_timeout=60 / step_timeout=180 / final_response_after_failure / loop_detection_window=20
- env：BROWSER_USE_* 前缀（FlatEnvConfig, config.py:191）；各 LLM 供应商 API_KEY

## 7. 权限 / policy / governance 机制
- **SecurityWatchdog**（security_watchdog.py）：allowed_domains 域白名单；导航前检查 + 重定向捕获 + 新标签关闭
- **PermissionsWatchdog**（permissions_watchdog.py）：CDP Browser.grantPermissions 最小权限授予
- **AGENTS.md v2**：开发治理（uv 强制/类型安全/pre-commit/不建随机示例/推荐 ChatBrowserUse/推荐 use_cloud）
- **Sandbox**：隔离执行
- **telemetry**：posthog 匿名遥测（ANONYMIZED_TELEMETRY 默认 true，可关闭）

## 8. 主要外部依赖
aiohttp/httpx（网络）、pydantic v2 + pydantic-settings（模型/配置）、bubus（EventBus）、cdp-use（类型化 CDP）、posthog（遥测）、mcp、browser-harness（CLI）、browser-use-sdk（云）、google-genai/openai/anthropic/groq/ollama/mistral/deepseek/cerebras/aws/azure/litellm（LLM）、pyotp（TOTP）、markdownify（HTML→MD）、psutil/screeninfo

## 9. 测试体系
- tests/ci/browser/：浏览器级测试（cdp_headers / cloud_browser / cross_origin_click / dom_selector_index_collisions / dom_serializer / navigation_readiness / send_keys_literal_plus / tabs / proxy / screenshot 等）
- tests/agent_tasks/：mind2web 任务级 yaml（amazon_laptop / browser_use_pip）
- 合计 35,131 行测试

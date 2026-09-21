# 04 · Flow Atlas（七类流）— browser-use

> 所有 Edge 从真实代码导出，可回溯 symbol/file/condition/state transition。
> 关键 Edge 标注回溯点。

## 1. Control Flow（控制流）
```
Agent.run() [service.py:run]
  → step() [service.py:1035]
      → Phase0: browser_session.wait_if_captcha_solving() [service.py:1040]  // captcha 阻塞等待
      → _prepare_context(step_info) [service.py:1087]  // DOM 摘要
      → state.last_model_output/last_result 清空 [service.py:1072-1074]  // 防超时残留
      → _get_next_action(browser_state_summary) [service.py:1176]
          → message_manager.get_messages()
          → asyncio.wait_for(_get_model_output_with_retry, llm_timeout) [service.py:1184]  // 超时→TimeoutError
          → _check_stop_or_pause() [service.py:1205]  // Ctrl+C/暂停检查
          → _handle_post_llm_processing() [service.py:1208]
          → _check_stop_or_pause() [service.py:1211]
      → _execute_actions() [service.py:1211] → multi_act() [service.py:2730]
      → _post_process() [service.py:1219]
          → _check_and_update_downloads / _update_plan_from_model_output / _update_loop_detector_actions / 失败计数
      → (异常) _handle_step_error(e) [service.py:1258]
      → (finally) _finalize() [service.py:1356]
```
**关键 Edge**：step → _handle_step_error 是唯一异常出口；_finalize 无条件执行（finally）——骨架兜底。

## 2. State Flow（状态流）
```
AgentState [views.py:251]
  ├── message_manager_state [views.py:271]  // 历史消息 + compacted_memory
  ├── loop_detector: ActionLoopDetector [views.py:275]
  │     ├── recent_action_hashes[window=20] [views.py:168]  // 动作重复
  │     └── recent_page_fingerprints [views.py:171]  // 页面停滞
  ├── consecutive_failures [views.py:251+]  // 单动作失败计数
  ├── last_model_output / last_result [service.py:1072]  // 每步清空
  └── n_steps / agent_id

BrowserSession [session.py:134]
  ├── CDP connection（_cdp_client）
  ├── tabs/pages 集合
  ├── _cached_browser_state_summary [service.py:2745]  // multi_act 用缓存 selector_map
  └── captcha wait 状态 [captcha_watchdog.py:65-67]
```
**状态转换**：每步 `last_model_output = None; last_result = None` 先清后设——保证 LLM 超时/动作异常时不把上一步状态当本次状态（service.py:1072-1074）。

## 3. Data Flow（数据流）
```
task(str) [Agent.__init__]
  → enhance_task_with_schema（附加输出 schema）[service.py:611]
  → message_manager.add_task_message
  → browser_state_summary（DOM 快照）[service.py:1087]
      → dom service: get_dom_tree → AX tree [dom/service.py:703]
      → serializer: serialize_accessible_elements → SerializedDOMState{root, selector_map} [serializer.py:114]
  → _message_manager.get_messages()（拼装 system+history+state）[service.py:1176]
  → LLM ainvoke(messages, output_format=AgentOutput) [llm/base.py:50]
  → AgentOutput{current_state, action[]} [views.py:388]
  → multi_act(action[]) → list[ActionResult] [service.py:2730]
  → ActionResult 写回 history（_make_history_item）[service.py:1738]
```
**关键 Edge**：DOM → selector_map → 动作索引：`_prepare_context` 的 DOM 状态里 selector_map 的索引被 LLM 动作引用，multi_act 用缓存 selector_map 校验索引有效性（service.py:2745-2750）。

## 4. Evidence Flow（证据流）
```
observe decorator（LMNR 可观测）[service.py:1190]
  → telemetry: ProductTelemetry.capture(event) → posthog [telemetry/service.py:103]
  → device_id（持久化/machine_fingerprint）[telemetry/service.py:28-53]
  → conversation save（save_conversation_path）[views.py:62]
  → gif 录制（generate_gif）[views.py:63]
  → har_recording_watchdog（HAR 网络记录）[watchdogs/har_recording_watchdog.py]
  → screenshot_watchdog（截图）
```
**关键 Edge**：`ANONYMIZED_TELEMETRY=true`（默认）→ posthog capture 开启（config.py:201）；可关闭。

## 5. Authority Flow（权威流）
```
用户（task + settings 授权）
  → Agent 控制面（allowed_domains/prohibited_domains 等）[profile.py:628-640]
  → SecurityWatchdog.on_NavigateToUrlEvent [security_watchdog.py:35]
      ├── _is_url_allowed(url) ? no → 阻断导航（事件 result 阻塞）
      ├── on_NavigationCompleteEvent（重定向后复查）? no → 重定向 about:blank [security_watchdog.py:50]
      └── on_TabCreatedEvent ? no → 关闭标签 [security_watchdog.py:73]
  → PermissionsWatchdog.on_BrowserConnectedEvent → CDP Browser.grantPermissions [permissions_watchdog.py:23]
  → multi_act：terminates_sequence 动作中止队列 [service.py:2733-2737]
  → 硬门控：预算/失败 → 强制 done（DoneAgentOutput 唯一工具）[service.py:1568-1593]
```
**关键 Edge**：SecurityWatchdog 的事件 handler 返回 dict（阻断信息）→ EventBus 阻止后续动作——授权边界在事件层执行（security_watchdog.py:38-48）。

## 6. Memory Flow（记忆流）
```
message_manager [message_manager/service.py:104]
  ├── 全历史（agent_history_items，带 token 元数据）
  ├── compacted_memory（压缩块）[service.py:153-162]  // <compacted_memory>...</compacted_memory> 注入
  └── maybe_compact_messages [message_manager/service.py:216]
       触发条件：steps_since >= compact_every_n_steps(25) 或 trigger_char_count(40000) [views.py:35-58]
       执行：旧历史 → compaction_llm 摘要 → compacted_memory；保留 keep_last_items=6
  → filesystem 状态保存（save_file_system_state）[service.py:749]
  → 变量检测（variable_detector：任务中 {var} 提取/回填）[agent/variable_detector.py]
```
**关键 Edge**：compaction 后旧历史被摘要替换但 `compacted_memory` 前缀始终注入（service.py:153-167）——压缩不是删除，是"降级保留"。

## 7. Policy Flow（策略流）
```
AGENTS.md v2（开发治理策略）→ 开发行为约束（uv/pre-commit/类型安全/不建随机示例）[AGENTS.md]
  → README/AGENTS 商业策略（推荐 ChatBrowserUse / use_cloud）→ 用户选型被引导 [AGENTS.md guidelines]
  → BrowserProfile 策略字段（allowed/prohibited/cookie_whitelist/permissions）[profile.py]
  → SecurityWatchdog/PermissionsWatchdog 执行策略
  → AgentSettings 运行策略（max_failures/max_actions/llm_timeout/loop_detection_enabled）[views.py:59-95]
  → 反馈：telemetry 数据 → 产品路线图 → 新策略写入 AGENTS.md/默认设置
```
**关键 Edge**：治理闭环 = 策略（AGENTS.md/profile/settings）→ 执行（watchdog/gates）→ 度量（telemetry）→ 策略更新。这是完整的 Policy Flow（v2 第七类流）。

---

## Flow → KO 交叉校验
| KO | 对应 Flow | 关键 Edge 回溯 |
|---|---|---|
| KO-01 | Control | step → finalize（finally）|
| KO-02 | Control/State | budget warning → force_done |
| KO-03 | Control/Authority | loop nudge 注入（软）vs force_done（硬）|
| KO-05 | Authority | SecurityWatchdog 三层检查 |
| KO-06 | Data | DOM → serializer → SerializedDOMState |
| KO-08 | Data/Evidence | ActionResult → history → 下一步决策 |
| KO-04 | State/Authority | EventBus → 16 watchdog |

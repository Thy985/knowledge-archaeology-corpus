# 02 · Engineering Knowledge（EK Graph）— browser-use

> 每条 EK 必须声明 `links`（六类边：mechanism/subsystem/causal/dependency/constraint/contrast）；孤立 EK 降 D 级。
> 证据编号 E1-E25 对应 06_validation/evidence_map.md。

## EK-01 受控 Agent 四阶段主循环
- **内容**：`Agent.step()` = Phase0 captcha-wait → Phase1 `_prepare_context`（DOM 摘要）→ Phase2 `_get_next_action`（LLM）+ `_execute_actions`（multi_act）→ Phase3 `_post_process`（下载/plan/循环检测）；异常统一 `_handle_step_error`，`finally` 总是 `_finalize`（E1, service.py:1035-1087）
- **为什么**：把"LLM 决策"嵌在受控骨架里——任何异常/超时/中断都有统一出口，状态不悬空
- **证据**：service.py:1035-1087（step 主循环 + finally finalize）
- **links**：mechanism→EK-02（执行防护）/ causal→EK-08（预算门控）→EK-07（失败强制 done）/ subsystem→EK-14（watchdog 体系）
- **知识层**：L2（为什么这样设计——LLM 不可靠所以骨架兜底）

## EK-02 双层 stale-DOM 防护（multi_act）
- **内容**：多动作执行时两层防"对过期 DOM 执行动作"：①静态标志 `terminates_sequence=True`（navigate/search/go_back/switch）自动中止剩余队列；②运行时检测——每个动作后比较 URL + focused target，变化即中止剩余队列；`done` 只允许单动作（E2, service.py:2730-2770）
- **为什么**：DOM 是快照，动作会改变页面；LLM 一次性输出 5 个动作，若页面已跳转后续动作全部失效甚至误操作
- **证据**：service.py:2730-2770（multi_act docstring + 双层防护代码）
- **links**：mechanism→EK-03（软循环检测，都是"防 agent 失效"）/ constraint→EK-05（动作原子化）/ subsystem→EK-01
- **知识层**：L2

## EK-03 软循环检测（ActionLoopDetector）
- **内容**：滚动窗口 20 个动作 hash + 页面指纹停滞检测；**只生成 escalating nudge 注入上下文，从不阻断动作**（"The agent can still repeat if it wants to"）（E3, views.py:157-175 + service.py:1496-1508）
  - **Escalating 分级（Reconciliation 补充，M-3）**：nudge 按重复强度分级升级——≥5 次温和提示、≥8 次强调、≥12 次强烈提示；且明示"如果每次重复有进展就继续"（views.py:211-230）——**防误报设计**：避免把"有效的重复操作"误判为循环
- **为什么**：硬阻断会破坏任务连续性；LLM 是"可被提示说服"的——注入上下文让模型自己意识到循环；分级+进展豁免避免误伤正常重复操作
- **证据**：views.py:157-175（docstring 明确 soft detection）+ views.py:211-230（get_nudge_message 分级）+ service.py:1496-1508（nudge 注入）
- **links**：contrast→EK-04（预算门控是硬性，循环检测是软性）/ mechanism→EK-02（防 agent 失效两机制）/ subsystem→EK-01
- **知识层**：L2（软硬约束对比是重要工程经验）

## EK-04 上下文预算分级降级
- **内容**：四级预算控制：①compaction（25 步/40k 字符触发，压缩旧历史为 compacted_memory 块，keep_last_items=6，独立 compaction_llm）；②75% 预算警告注入（"优先保存部分结果"）；③最后一步强制 done（只留 done 工具）；④max_failures 后强制 done（恢复调用）（E5, views.py:35-58 + service.py:1542-1593）
- **为什么**：上下文有上限、步骤有预算——不降级就会"耗尽所有步数什么也没保存"；部分结果 > 无结果
- **证据**：MessageCompactionSettings（views.py:35-58）+ _inject_budget_warning/_force_done_after_last_step/_force_done_after_failure（service.py:1542-1593）
- **links**：causal→EK-07（失败降级路径）/ dependency→EK-06（DOM 压缩减少 token 压力）/ subsystem→EK-01
- **知识层**：L2

## EK-05 动作原子化 + 结构化 ActionResult
- **内容**：动作全集为 pydantic ActionModel（navigate/click/type/done/extract/scroll/send_keys/upload/switch/close/dropdown…），每个动作返回结构化 `ActionResult`（is_done/success/extracted_content/attachments/error）（E13+E21, tools/views.py + agent/views.py:307）
- **为什么**：结构化返回让 agent 推理（"上次动作失败了吗/成功了吗"），LLM 靠这个做下一步决策；`ActionResult` 是 agent 的"传感器读数"
- **证据**：tools/views.py:8-203（ActionModel 全集）+ agent/views.py:307（ActionResult）
- **links**：subsystem→EK-02（执行层）/ mechanism→EK-06（DOM→动作索引，都是 agent 观察通道）/ subsystem→EK-01
- **知识层**：L2

## EK-06 DOM 压缩：只序列化交互元素
- **内容**：`DOMTreeSerializer` 从 AX tree 过滤 → 仅可交互元素分配 selector_map 索引（interactive counter），非交互内容丢弃；缓存复用 previous selector_map；shadow DOM 内交互内容保留；Markdown 提取备选（E6, dom/serializer/serializer.py:42-165）
- **为什么**：完整 DOM 太大（几 MB），LLM 上下文有限——**上下文压缩的第一步在观察端**，不在模型端
- **证据**：serializer.py（serialize_accessible_elements + _create_simplified_tree + _assign_interactive_indices）
- **links**：dependency→EK-04（压缩减少触发 compaction 概率）/ causal→EK-02（selector 索引供动作使用）/ subsystem→EK-05
- **知识层**：L2

## EK-07 失败计数与降级路径
- **内容**：`consecutive_failures` 只对**单动作步骤**计错；多动作步骤出错交给循环检测/replan；达到 max_failures（默认 5）+ final_response_after_failure → 最后恢复调用 → 强制 done（E22, service.py:1219-1246）
- **为什么**：单动作失败是明确的失败信号；多动作失败可能是序列性问题（页面跳变），单步计数会误判
- **证据**：service.py:1219-1246（_post_process 失败计数逻辑）+ settings（views.py:59-95）
- **links**：causal←EK-04（预算降级链末端）/ constraint→EK-01（主循环终态）/ subsystem→EK-03
- **知识层**：L2

## EK-08 域级安全边界（SecurityWatchdog）
- **内容**：三层域白名单检查：①导航前 `_is_url_allowed` 阻断；②导航完成后重定向捕获（非白名单 → 重定向 about:blank）；③新标签创建检查（不允许域 → 关闭标签）；root domain 启发式（www 补全）（E8, security_watchdog.py:22-101）
  - **IP 绕过防护（Reconciliation 补充，M-1）**：`block_ip_addresses`（profile.py:636）为 true 时，`_is_ip_address` 镜像 WHATWG host canonicalization 识别 decimal/hex/octal/short-form/percent-encoded/Unicode digits 的 IP 编码，在域检查前拒绝（security_watchdog.py:143-230）——域白名单最直接的绕过是 IP 直连，必须单独阻断
  - 性能设计：allowed_domains 为 set 时 fast path O(1) 精确匹配（www 变体）；list 时 slow path 模式匹配（glob 警告：`*.example.com` 同时匹配子域与主域）
- **为什么**：agent 自主导航会踩恶意/越界域；重定向是绕过检查的主要通道（先允许后跳转）；IP 直连是绕过域白名单的次要通道
- **证据**：security_watchdog.py（on_NavigateToUrlEvent / on_NavigationCompleteEvent / on_TabCreatedEvent / _is_ip_address / _is_url_allowed）+ profile.py:636（block_ip_addresses）
- **links**：mechanism→EK-09（权限最小化，都是 Authority 面）/ subsystem→EK-14（watchdog 体系）/ constraint→EK-02（导航动作受域约束）
- **知识层**：L2（重定向捕获是高级安全设计）

## EK-09 权限最小化授予（PermissionsWatchdog + cookie 白名单）
- **内容**：连接时 CDP `Browser.grantPermissions` 按 profile 权限列表授予（origin=None 全 origin）；cookie_whitelist_domains 限制持久化（E10, permissions_watchdog.py:14-42 + profile.py:628-640）
- **为什么**：浏览器权限（摄像头/通知/地理位置）默认不授予；agent 场景按需最小授予
- **证据**：permissions_watchdog.py（grantPermissions）+ profile.py（allowed/prohibited/cookie_whitelist）
- **links**：mechanism→EK-08（安全面两支柱）/ subsystem→EK-14
- **知识层**：L2

## EK-10 CaptchaWatchdog 阻塞等待
- **内容**：监听 `BrowserUse.captchaSolverStarted/Finished` CDP 事件；step Phase0 调 `wait_if_captcha_solving()` 阻塞等待 captcha 解决（带超时），结果注入上下文（E11, captcha_watchdog.py:34-67 + service.py:1040-1058）
  - **细节（Reconciliation 补充，M-2）**：等待结果含 vendor（验证码供应商）、outcome 三态（success/failed/timeout）、duration_ms；以 ActionResult(long_term_memory=...) 注入 LLM 上下文；等待耗时从 step 计时中扣除（step_start_time 重置，service.py:1043-1055）
- **为什么**：验证码是 agent 真实世界的必经关卡；等待比盲目操作可靠，结果要让 LLM 看到（含失败/超时结果）
- **证据**：captcha_watchdog.py + service.py:1040-1058（Phase0 captcha 等待 + 结果注入 + 计时重置）
- **links**：subsystem→EK-14 / dependency→EK-01（Phase0 前置等待）/ contrast→EK-18（云浏览器 captcha 解决服务）
- **知识层**：L2

## EK-11 DownloadsWatchdog 内容协商下载判定
- **内容**：基于 content-disposition / 文件扩展名 / content-type / URL 判断是否自动下载网络响应；注册下载回调（on_download_start/progress/complete）（E12, downloads_watchdog.py:64-170）
- **为什么**：agent 点击会触发下载；区分"该下载的附件"与"误触页面资源"需要内容协商
- **证据**：downloads_watchdog.py（_should_auto_download_network_response + 回调注册）
- **links**：subsystem→EK-14 / causal→EK-01（_post_process 检查下载）
- **知识层**：L1（项目事实）

## EK-12 LLM 多供应商归一化 + 工厂注册
- **内容**：`BaseChatModel` Protocol（provider/name/model_name/ainvoke + 结构化输出）；`get_llm_by_name` 工厂按 `provider_model` 命名解析（underscore→dash 还原、mistral alias、环境变量 key）；支持 12+ 供应商（E7, llm/models.py:89-160 + llm/base.py:33-62）
- **为什么**：web agent 强依赖"结构化输出"（AgentOutput）；供应商差异（thinking 模型/vision/超时）必须在抽象层归一化
- **证据**：llm/base.py（Protocol + ChatInvokeCompletion）+ models.py（工厂）
- **links**：dependency→EK-01（LLM 是决策引擎）/ subsystem→EK-01 / contrast→EK-16（自带 ChatBrowserUse 模型）
- **知识层**：L2

## EK-13 MCP 服务器：browser 能力工具面化
- **内容**：MCP server 暴露 12+ 工具（browser_navigate/click/type/get_state/extract_content/get_html/screenshot/scroll/go_back/list_tabs/switch_tab/close_tab）；session_timeout 管理；支持 allowed_domains 初始化；失败时重试用 browser-use agent（E15, mcp/server.py:186-585）
- **为什么**：MCP 是 agent 生态标准接入面；browser 能力做成 MCP 工具让任何 MCP client 复用
- **证据**：mcp/server.py（BrowserUseServer + _setup_handlers）
- **links**：subsystem→EK-05（工具面）/ contrast→EK-16（beta 终端路线）/ mechanism→EK-01（agent 循环复用）
- **知识层**：L2

## EK-14 事件驱动 Watchdog 架构（16 监视器）
- **内容**：BrowserSession 用 bubus EventBus 协调 16 个 watchdog（aboutblank/captcha/crash/default_action/dom/downloads/har_recording/local_browser/permissions/popups/recording/screenshot/security/storage_state）；BaseWatchdog.attach_handler_to_session 事件订阅（E9, watchdog_base.py:15-243 + session.py:134-137）
- **为什么**：浏览器操作横切关注点太多（安全/下载/验证码/权限/DOM/崩溃），事件驱动让每个关注点独立成监视器，主循环保持干净
- **证据**：watchdog_base.py（attach/detach）+ session.py（2-layer 架构 + 事件注册）
- **links**：subsystem→EK-08/EK-09/EK-10/EK-11（各 watchdog）/ dependency→EK-01（主循环依赖监视器状态）
- **知识层**：L3 潜质（事件驱动监视器模式，对照 OpenClaw 的 plugin 架构）

## EK-15 Beta 终端 Agent（Rust SDK 路线）
- **内容**：beta/service.py 通过 RustSdkClient（JSON-RPC over stdio）驱动终端二进制 + agent tools（检测 ripgrep、prepend PATH）；Laminar 可观测集成（span/trace）；错误分级 RustSdkJsonRpcError（E16, beta/service.py:104-472）
- **为什么**：终端操作（读写文件/运行命令）需要与浏览器互补的"第二能力面"；Rust 核心提供确定性执行
- **证据**：beta/service.py（_find_browser_use_terminal_binary / RustSdkClient / _agent_tools_dir_contains_ripgrep）
- **links**：contrast→EK-13（MCP 工具面 vs 终端 agent）/ subsystem→EK-01 / dependency→EK-12
- **知识层**：L1-L2（beta 未完全验证，测试覆盖有限）

## EK-16 OSS 漏斗产品化（AGENTS.md 明文策略）
- **内容**：AGENTS.md 明确要求：默认推荐模型 ChatBrowserUse（"best for browser automation"）、用户问性能就推荐 `use_cloud=True`（云端浏览器，captcha 绕过/住宅代理/认证同步）、需要 BROWSER_USE_API_KEY；README 展示 Cloud $0.02/browser-hour + Hosted API（E18, AGENTS.md + README）
- **为什么**：OSS 库是流量入口，商业层（云浏览器/托管 API/自有模型）是收入——"开发者在 README 被教育使用付费服务"
- **证据**：AGENTS.md guidelines + README（cloud badges + $0.02/browser-hour）
- **links**：contrast→EK-12（多供应商中立 vs 自有模型推荐）/ subsystem→EK-17（telemetry 支撑）
- **知识层**：L3 潜质（与 mem0 同构——跨项目候选 C-02）

## EK-17 遥测（posthog 匿名 + device_id）
- **内容**：ProductTelemetry.capture → posthog；device_id 持久化 + machine_fingerprint；ANONYMIZED_TELEMETRY 默认 true（E19, telemetry/service.py:28-136 + config.py:201）
- **为什么**：产品团队需要使用数据驱动路线图；匿名默认开、可关闭
- **证据**：telemetry/service.py（get_or_create_device_id / ProductTelemetry）+ config.py:201
- **links**：subsystem→EK-16（商业漏斗数据侧）/ contrast→EK-03（遥测 vs 本地 nudge 都是"观察 agent"）
- **知识层**：L1

## EK-18 云端浏览器集成（cloud/ + 本地/云双层 BrowserSession）
- **内容**：BrowserSession overload 双模式：本地（headless/user_data_dir/profile）vs 云（cloud_profile_id/proxy_country_code/use_cloud）；cloud 浏览器提供 captcha 解决/住宅代理/本地-远端 profile 同步（E18+E20, session.py:134-230 + README）
- **为什么**：本地浏览器有反爬/captcha/环境问题；云端托管解决生产可用性
- **证据**：session.py overload 签名 + README cloud 描述
- **links**：contrast→EK-10（本地 captcha watchdog vs 云端 captcha 服务）/ subsystem→EK-16 / dependency→EK-01
- **知识层**：L1-L2

## EK-19 CLI 基于 Browser Harness
- **内容**：cli.py 是 browser-harness 的薄封装（设置 BH_CLIENT=browser-use-cli 环境标记、版本检测、退出码映射）；browser-harness 为 pyproject 依赖（E20, cli.py + pyproject.toml）
- **为什么**：CLI 是低门槛入口；复用自愈 harness（agent 边跑边写代码）而非重复造轮子
- **证据**：cli.py（_set_harness_client_env / _exit_code）+ pyproject（browser-harness==0.1.13）
- **links**：subsystem→EK-16（入口漏斗）/ contrast→EK-15（CLI harness vs beta Rust SDK）
- **知识层**：L1

## EK-20 类型动作的深度工程（default_action_watchdog）
- **内容**：输入动作处理大量边界：清空字段（_clear_text_field）、直接赋值（_set_value_directly 检测 class/data 属性）、触发框架事件（_trigger_framework_events）、元素遮挡检测（_check_element_occlusion）、滚动姿势（_scroll_with_cdp_gesture）、ARIA combobox 处理（E24, default_action_watchdog.py:573-3088）
- **为什么**：真实网页输入是"脏"的（React/Vue 框架事件、遮挡、combobox）——天真输入经常失效
- **证据**：default_action_watchdog.py 多个 _impl 方法
- **links**：subsystem→EK-05（动作执行）/ dependency→EK-06（元素索引）
- **知识层**：L2

## EK-21 停止/暂停检查（_check_stop_or_pause）
- **内容**：LLM 调用前后多次 `_check_stop_or_pause`（Ctrl+C/暂停信号）；LLM 超时（llm_timeout=60，gemini 30/o3 90 自动检测）→ 明确 TimeoutError（E1, service.py:1013-1033 + 1176-1211）
- **为什么**：agent 运行数分钟到数小时，用户必须能随时中断；超时要有明确错误而非静默
- **证据**：service.py（_check_stop_or_pause + asyncio.wait_for(llm_timeout)）
- **links**：subsystem→EK-01 / constraint→EK-04（预算与中断同属控制面）
- **知识层**：L2

## EK-22 EventLoop 兼容补丁（monkeypatch）
- **内容**：`__init__.py` monkeypatch `BaseSubprocessTransport.__del__`——关闭的 event loop 上不抛 "Event loop is closed" 噪音错误（E23, __init__.py:30-45）
- **为什么**：asyncio 关闭时序在退出时是常见噪音崩溃源；显式打补丁保证干净退出
- **证据**：__init__.py（_patched_del + 原 __del__ 保存）
- **links**：subsystem→EK-01 / contrast→EK-21（都是"健壮退出"）
- **知识层**：L1

---

## EK Graph 检查
- EK 总数：22；有 links 的 EK：22（100%）；平均出边：~2.5
- 游离 EK：无
- D 级 EK：EK-11、EK-17、EK-19、EK-22（项目局部事实，保留为底座）

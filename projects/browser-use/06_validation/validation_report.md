# 06 · Validation & Evidence — browser-use (ARCH-2026-09-22-001)

## Evidence Map（E1-E25）
| # | 证据 | 位置 | 类型 |
|---|---|---|---|
| E1 | Agent.step() 四阶段 + finally finalize | agent/service.py:1035-1087 | code |
| E2 | multi_act 双层 stale-DOM 防护 + done 单动作 | agent/service.py:2730-2770 | code |
| E3 | ActionLoopDetector 软检测（never blocks） | agent/views.py:157-175 + service.py:1496-1508 | code+docstring |
| E4 | budget warning 75% / force done last / force done failure | agent/service.py:1542-1593 | code |
| E5 | MessageCompactionSettings（25 步/40k/keep 6/独立 llm） | agent/views.py:35-58 | code |
| E6 | DOM 只序列化交互元素 + selector_map | dom/serializer/serializer.py:42-165 | code |
| E7 | LLM 工厂 get_llm_by_name + 供应商归一化 | llm/models.py:89-160 + base.py:33-62 | code |
| E8 | SecurityWatchdog 三层域检查 + root domain 启发 | browser/watchdogs/security_watchdog.py:22-101 | code |
| E9 | EventBus + 16 watchdog | watchdog_base.py:15-243 + session.py:134-137 + watchdogs/ 目录 | code |
| E10 | PermissionsWatchdog grantPermissions | permissions_watchdog.py:14-42 | code |
| E11 | CaptchaWatchdog wait_if_captcha_solving | captcha_watchdog.py:34-67 + service.py:1040-1058 | code |
| E12 | DownloadsWatchdog 内容协商 | downloads_watchdog.py:64-170 | code |
| E13 | AgentOutput + NoThinking/FlashMode 变体 | agent/views.py:388-487 | code |
| E14 | Judge（use_judge/ground_truth） | agent/judge.py + views.py:59-95 | code |
| E15 | MCP 12+ 工具面 + session_timeout | mcp/server.py:186-585 | code |
| E16 | RustSdkClient JSON-RPC + agent tools + ripgrep | beta/service.py:104-472 | code |
| E17 | sandbox cloudpickle + SSE | sandbox/sandbox.py | code |
| E18 | 商业三层（OSS/Cloud/Hosted/ChatBrowserUse） | README.md + AGENTS.md | doc |
| E19 | posthog telemetry + device_id + 匿名默认 | telemetry/service.py:28-136 + config.py:201 | code |
| E20 | CLI 基于 browser-harness（BH_CLIENT） | cli.py + pyproject.toml | code |
| E21 | AgentSettings 控制面（max_failures=5 等） | agent/views.py:59-95 | code |
| E22 | 失败计数仅单动作步骤 | agent/service.py:1219-1246 | code |
| E23 | BaseSubprocessTransport monkeypatch | __init__.py:30-45 | code |
| E24 | 类型动作深度工程（遮挡/框架事件/combobox） | default_action_watchdog.py:573-3088 | code |
| E25 | 循环检测 nudge 分级（repetition+stagnation） | views.py:157-175 | code |
| M-1 | block_ip_addresses IP 绕过防护（WHATWG canonicalization） | security_watchdog.py:143-230 + profile.py:636 | code（Reconciliation 补充）|
| M-2 | captcha vendor/outcome 三态 + 计时重置 | service.py:1043-1055 | code（Reconciliation 补充）|
| M-3 | loop nudge escalating 分级 + 进展豁免 | views.py:211-230 | code（Reconciliation 补充）|

## Validator 执行摘要

### 1. Truth Audit（真值审计）— PASS
- 方法：Blind Reconstruction（不读考古结果，独立 grep 关键 symbol）
- 抽查 10 条 Fact（E1/E2/E3/E8/E13/E15/E18/E21/E22/E23）全部在代码/文档中找到原文
- 无事实错误

### 2. Coverage Audit（覆盖审计）— PARTIAL
- 已覆盖：Agent 主循环/DOM 压缩/动作执行/安全 watchdog/LLM 抽象/MCP/beta 概览/sandbox 概览/商业/遥测/CLI
- 未穷尽：beta/service.py 6,810 行深读（时间盒限制）；sandbox 完整协议；default_action_watchdog 全部边界（E24 仅抽样）；tests 全量逻辑
- 影响：C-04 保持 Observation；EK-15 标 L1-L2

### 3. Flow Audit（流审计）— PASS
- 七类流全部从代码导出；关键 Edge 可回溯 symbol/file/line
- Flow→KO 交叉校验通过（07 项）

### 4. Abstraction Audit（抽象审计）— PASS_WITH_DOWNGRADES
- KO-09（软约束可说服性）从"Principle"降为 L4 + Cross-project validation pending（单项目证据）
- KO-07 保留 L4（browser-use 强证据 + 两对照项目）
- 无"单案例→Pattern"违规

### 5. Counterexample Audit（反例审计）— PASS（3 反例）
- 反例 1：**"软门控总是有效"** → 反驳：EK-04 显示预算警告 75% 是软 nudge 但最后一步是硬强制——软门控有失效场景（LLM 忽略警告），所以才有硬兜底（service.py:1568-1593）
- 反例 2：**"watchdog 全部阻断"** → 反驳：DownloadsWatchdog 只是登记下载；CaptchaWatchdog 只是等待；只有 SecurityWatchdog 真正阻断——watchdog 是"监视/协调"不是"全阻断"（downloads_watchdog.py + captcha_watchdog.py）
- 反例 3：**"事件驱动适用于所有关注点"** → 反驳：动作执行本身（multi_act）是命令式顺序代码不是事件驱动；事件驱动只用于横切监视（service.py:2730）
- 结果：KO-03/KO-04 边界被收紧，但成立

### 6. Epistemic Audit（认知状态审计）— PASS
- Fact/Observation/Hypothesis/Pattern/Cognitive Model/Methodology 分离
- C-01~C-05 全部标 Hypothesis/pending；KO-09 标 Cross-project validation pending
- EK 层与 KO 层严格分层（无"把 EK 当 KO"）

## 质量指标
| 指标 | 值 | 目标 |
|---|---|---|
| Facts/Evidence | 25 | 100+（小项目按比例缩放）|
| Engineering Knowledge | 22 | 40-60（小项目按比例缩放）|
| Patterns (L3) | 6 | 15-25 |
| Cognitive Models (L4) | 3 | 7-12 |
| Methodology (L5) | 1 | — |
| Flow Atlas | 7 类 | 7 |
| Candidates | 5 | — |
| EK links 覆盖率 | 100% | 100% |
| 游离 EK | 0 | <20% |
| KO 可回溯率 | 100% | 100% |

## 已知局限
1. 浅克隆（depth=1）：无 git history——未分析提交演化/失败修复史（Failure Analyst 视角受限）
2. beta 模块深读不足（C-04）
3. 未运行测试/未实机运行（只读考古，不安装依赖）
4. 云浏览器/商业层为 README/AGENTS 声明，未验证实际行为
